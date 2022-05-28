---
title: "Open code unsafe.Slice"
date: "2022-05-28"
tags: ["go", "golang", "unsafe", "compiler"]
draft: false
---

---

### Prologue

[unsafe.Slice][unsafe_slice_link] was added in go1.17 ([CL 312214][cl_312214]), providing Go developers a safer way to construct a slice.
With a pointer `ptr` of type `*T`, `unsafe.Slice(ptr, len)` will:

 - Creates a slice `[]T`
 - The underlying array of `[]T` starts at `ptr`
 - The length and capacity of `[]T` are `len`

Without [unsafe.Slice][unsafe_slice_link], we have to use [reflect.SliceHeader][reflect_sliceheader_link], which is very clumsy to use, because
its `Data` field is `uintptr`, not an [unsafe.Pointer][unsafe_pointer_link]. Or introducing own type, which mimic `reflect.SliceHeader`, but with
`Data` field is an `unsafe.Pointer` instead like the [runtime][runtime_link] package.

---
_Note_

I invite you to read [unsafe.Pointer rules][unsafe_pointer_link] to understand how much clumsy is!

---

### The problem

The go compiler implements `unsafe.Slice` as a runtime call, so it would cause performance regression for existing code.

Compiling following code:

```golang
package p

import "unsafe"

func unsafeSlice() []byte {
	var p [10]byte
	return unsafe.Slice(&p[0], 10)
}
```

with `-S`, you can see:

```text
$ go1.18.2 tool compile -S p.go
"".unsafeSlice STEXT size=92 args=0x0 locals=0x28 funcid=0x0 align=0x0
	0x0000 00000 (p.go:5)	TEXT	"".unsafeSlice(SB), ABIInternal, $40-0
	0x0000 00000 (p.go:5)	CMPQ	SP, 16(R14)
	0x0004 00004 (p.go:5)	PCDATA	$0, $-2
	0x0004 00004 (p.go:5)	JLS	85
	0x0006 00006 (p.go:5)	PCDATA	$0, $-1
	0x0006 00006 (p.go:5)	SUBQ	$40, SP
	0x000a 00010 (p.go:5)	MOVQ	BP, 32(SP)
	0x000f 00015 (p.go:5)	LEAQ	32(SP), BP
	0x0014 00020 (p.go:5)	FUNCDATA	$0, gclocals·69c1753bd5f81501d95132d08af04464(SB)
	0x0014 00020 (p.go:5)	FUNCDATA	$1, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
	0x0014 00020 (p.go:6)	LEAQ	type.[10]uint8(SB), AX
	0x001b 00027 (p.go:6)	PCDATA	$1, $0
	0x001b 00027 (p.go:6)	NOP
	0x0020 00032 (p.go:6)	CALL	runtime.newobject(SB)
	0x0025 00037 (p.go:6)	MOVQ	AX, "".&p+24(SP)
	0x002a 00042 (p.go:7)	MOVQ	AX, BX
	0x002d 00045 (p.go:7)	MOVL	$10, CX
	0x0032 00050 (p.go:7)	LEAQ	type.uint8(SB), AX
	0x0039 00057 (p.go:7)	PCDATA	$1, $1
	0x0039 00057 (p.go:7)	CALL	runtime.unsafeslice(SB)
	0x003e 00062 (p.go:7)	MOVQ	"".&p+24(SP), AX
	0x0043 00067 (p.go:7)	MOVL	$10, BX
	0x0048 00072 (p.go:7)	MOVQ	BX, CX
	0x004b 00075 (p.go:7)	MOVQ	32(SP), BP
	0x0050 00080 (p.go:7)	ADDQ	$40, SP
	0x0054 00084 (p.go:7)	RET
	0x0055 00085 (p.go:7)	NOP
	0x0055 00085 (p.go:5)	PCDATA	$1, $-1
	0x0055 00085 (p.go:5)	PCDATA	$0, $-2
	0x0055 00085 (p.go:5)	CALL	runtime.morestack_noctxt(SB)
	0x005a 00090 (p.go:5)	PCDATA	$0, $-1
	0x005a 00090 (p.go:5)	JMP	0
    ...
    <retracted>
```

And here's the benchmark result:

```golang
package p

import (
	"testing"
)

func BenchmarkUnsafeSlice(b *testing.B) {
	for i := 0; i < b.N; i++ {
		unsafeSlice()
	}
}
```

```text
$ go1.18.2 test -run=NONE -bench=. -benchmem p.go p_test.go
goos: linux
goarch: amd64
cpu: 11th Gen Intel(R) Core(TM) i7-1165G7 @ 2.80GHz
BenchmarkUnsafeSlice-8   	780883747	         1.509 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	command-line-arguments	1.336s
```

### Solution

Open code (or inlining) `unsafe.Slice` instead of making it a runtime call!

The implementation of [`unsafe.Slice in go1.18.2`][runtime_unsafeslice] is quite short and able to be inlined:

```golang
func unsafeslice(et *_type, ptr unsafe.Pointer, len int) {
	if len < 0 {
		panicunsafeslicelen()
	}

	mem, overflow := math.MulUintptr(et.size, uintptr(len))
	if overflow || mem > -uintptr(ptr) {
		if ptr == nil {
			panic(errorString("unsafe.Slice: ptr is nil and len is not zero"))
		}
		panicunsafeslicelen()
	}
}
```

The only problem is `math.MulUintptr`, which is from [runtime/internal/math][runtime_internal_math_link]. The compiler can [generate stubs for `runtime` package][mkbuiltin_link],
but not for `runtime/internal/math`. Adding ability for the compiler to generate stubs for packages other than `runtime` package is doable, but will require more works, and should
be deserved for another CL.

Luckily, the compiler recognizes `runtime/internal/math.MulUintptr` as an intrinsic, so we can add a wrapper function for it, and make the
compiler recognizes the wrapper as an intrinsic, too:

```golang
// This is a wrapper over runtime/internal/math.MulUintptr,
// so the compiler can recognize and treat it as an intrinsic.
func mulUintptr(a, b uintptr) (uintptr, bool) {
	return math.MulUintptr(a, b)
}
```

and in [cmd/compile/internal/ssagen][ssagen_link]:

```golang
addF("runtime/internal/math", "MulUintptr",
	func(s *state, n *ir.CallExpr, args []*ssa.Value) *ssa.Value {
		if s.config.PtrSize == 4 {
			return s.newValue2(ssa.OpMul32uover, types.NewTuple(types.Types[types.TUINT], types.Types[types.TUINT]), args[0], args[1])
		}
		return s.newValue2(ssa.OpMul64uover, types.NewTuple(types.Types[types.TUINT], types.Types[types.TUINT]), args[0], args[1])
	},
	sys.AMD64, sys.I386, sys.MIPS64, sys.RISCV64)
alias("runtime", "mulUintptr", "runtime/internal/math", "MulUintptr", all...)
```

The full implementation at [CL 362934][cl_362934].

The benchmark result with `go version devel go1.19-1247354a08 Sat May 28 00:28:55 2022 +0000 linux/amd64`:

```text
$ go test -run=NONE -bench=. -benchmem p.go p_test.go
goos: linux
goarch: amd64
cpu: 11th Gen Intel(R) Core(TM) i7-1165G7 @ 2.80GHz
BenchmarkUnsafeSlice-8   	1000000000	         0.7844 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	command-line-arguments	0.868s
```

It's ~50% faster than making a runtime call:

```text
name           old time/op    new time/op    delta
UnsafeSlice-8    1.71ns ±12%    0.82ns ± 8%  -52.17%  (p=0.000 n=10+10)
```

### Epilogue

Though go1.19 development cycle focus mainly on generic, IMHO, it's still worth to make this optimization land in, helping Go developers to write
safer go code with acceptable performance (especially for `runtime` package).

Thanks [@commaok][commaok] for [the idea][opencode_unsafeslice_link].

And thank you all for reading so far!

Till next time!

---

[unsafe_slice_link]: https://pkg.go.dev/unsafe#Slice
[cl_312214]: https://go-review.googlesource.com/c/go/+/312214
[reflect_sliceheader_link]: https://pkg.go.dev/reflect#SliceHeader
[unsafe_pointer_link]: https://pkg.go.dev/unsafe#Pointer
[runtime_link]: https://cs.opensource.google/go/go/+/refs/tags/go1.18.2:src/runtime/slice.go;l=15
[runtime_unsafeslice]: https://cs.opensource.google/go/go/+/refs/tags/go1.18.2:src/runtime/slice.go;l=120
[runtime_internal_math_link]: https://cs.opensource.google/go/go/+/refs/tags/go1.18.2:src/runtime/internal/math
[mkbuiltin_link]: https://cs.opensource.google/go/go/+/refs/tags/go1.18.2:src/cmd/compile/internal/typecheck/mkbuiltin.go
[ssagen_link]: https://cs.opensource.google/go/go/+/refs/tags/go1.18.2:src/cmd/compile/internal/ssagen/ssa.go;l=3859
[cl_362934]: https://go-review.googlesource.com/c/go/+/362934
[commaok]: https://twitter.com/commaok
[opencode_unsafeslice_link]: https://github.com/golang/go/issues/48798
