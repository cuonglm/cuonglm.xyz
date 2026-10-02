---
title: "Closures that capture constants no longer allocate"
date: "2026-10-02"
tags: ["go", "golang", "compiler", "escape analysis"]
draft: false
---

---

### Prologue

[Issue 5370][issue_5370] was filed in 2013, back when the Go compiler was still written in C. It asks for a simple thing:
if a closure only captures variables that are effectively constants, the compiler should not need to build a closure for it at all.

It took more than 13 years, but the fix finally landed in [CL 783641][cl_783641].

---

### The problem

Consider following code:

```go
package p

var sink func() int

func f() {
	n := 42
	sink = func() int { return n }
}
```

`n` is never reassigned, nor is its address taken, so it holds the constant `42` for its whole lifetime. We call such a variable a "dynamic constant":
it is not declared with `const`, but it behaves exactly like one.

Yet, compiling above code with `go1.27.1 tool compile -m -d=closure x.go`:

```text
x.go:5:6: can inline f
x.go:7:9: can inline f.func1
x.go:7:9: func literal escapes to heap
x.go:7:9: heap closure, captured vars = [n]
```

The func literal captures `n`, and because it escapes to `sink`, its closure record must be heap allocated. Looking at the assembly:

```text
<unlinkable>.f STEXT size=105 align=0x0 args=0x0 locals=0x28 funcid=0x0
	0x0000 00000 (x.go:5)	TEXT	<unlinkable>.f(SB), ABIInternal, $40-0
	0x0000 00000 (x.go:5)	CMPQ	SP, 16(R14)
	0x0004 00004 (x.go:5)	JLS	98
	0x0006 00006 (x.go:5)	PUSHQ	BP
	0x0007 00007 (x.go:5)	MOVQ	SP, BP
	0x000a 00010 (x.go:5)	SUBQ	$32, SP
	0x000e 00014 (x.go:7)	MOVL	$16, AX
	0x0013 00019 (x.go:7)	LEAQ	type:noalg.struct { F uintptr; X0 int }(SB), BX
	0x001a 00026 (x.go:7)	MOVL	$1, CX
	0x001f 00031 (x.go:7)	NOP
	0x0020 00032 (x.go:7)	CALL	runtime.mallocgcSmallNoScanSC2(SB)
	0x0025 00037 (x.go:7)	LEAQ	<unlinkable>.f.func1(SB), DX
	0x002c 00044 (x.go:7)	MOVQ	DX, (AX)
	0x002f 00047 (x.go:7)	MOVQ	$42, 8(AX)
	...
	0x0055 00085 (x.go:7)	MOVQ	AX, <unlinkable>.sink(SB)
	...
```

At `0x0020`, the compiler calls into the runtime to allocate a `struct { F uintptr; X0 int }`, the closure record holding the function pointer
and a copy of `n`. Then at `0x002f`, it stores `$42` into that record. We pay a heap allocation just to carry around a value the compiler
already knows at compile time!

---

### The fix

The idea is simple: during escape analysis, once the compiler has determined which variables are captured by value (meaning they are neither
address taken nor reassigned), it asks the [ReassignOracle][reassign_oracle] whether the captured variable has a static value which is a constant.
If it does, the captured variable is rewritten into an ordinary local variable of the closure, initialized with that constant.

Conceptually, the closure above becomes:

```go
sink = func() int {
	n := 42
	return n
}
```

A closure that captures nothing is not a closure anymore, so the compiler can refer to the function directly, without building a closure record for it.

With the fix, `go tool compile -m -d=closure x.go` now reports:

```text
x.go:5:6: can inline f
x.go:7:9: can inline f.func1
x.go:7:9: func literal escapes to heap
x.go:7:9: closure converted to global
```

and the assembly:

```text
<unlinkable>.f STEXT size=57 align=0x0 args=0x0 locals=0x8 funcid=0x0
	0x0000 00000 (x.go:5)	TEXT	<unlinkable>.f(SB), ABIInternal, $8-0
	0x0000 00000 (x.go:5)	CMPQ	SP, 16(R14)
	0x0004 00004 (x.go:5)	JLS	50
	0x0006 00006 (x.go:5)	PUSHQ	BP
	0x0007 00007 (x.go:5)	MOVQ	SP, BP
	...
	0x0022 00034 (x.go:7)	LEAQ	<unlinkable>.f.func1·f(SB), AX
	0x0029 00041 (x.go:7)	MOVQ	AX, <unlinkable>.sink(SB)
	...
```

No more allocation, `sink` simply points to the static func value `f.func1·f`. A quick benchmark shows the difference:

```go
package bench

import "testing"

var sink func() int

//go:noinline
func f() {
	n := 42
	sink = func() int { return n }
}

func BenchmarkClosure(b *testing.B) {
	for b.Loop() {
		f()
	}
}
```

Running it with `go test -bench . -benchmem`:

```text
# go1.27.1
BenchmarkClosure-8   	100000000	        12.60 ns/op	      16 B/op	       1 allocs/op
# tip
BenchmarkClosure-8   	1000000000	         1.146 ns/op	       0 B/op	       0 allocs/op
```

Variables that are not constants are still captured as usual, so a closure capturing a mix of both only captures the non-constant ones.

---

### The devil is in the details

The idea is simple, but getting it right required handling a few subtle cases.

**Substitute the constant, or assign it?**

The most obvious approach would be replacing every use of `n` with the literal `42`. But that breaks code like:

```go
n := -1
f := func() { _ = make([]byte, n) }
```

`make([]byte, -1)` is a compile time error, while the spec requires `make([]byte, n)` with a negative `n` to panic at run time
(see [issue 4085][issue_4085]). That's why the fix declares a new local variable assigned with the constant, instead of substituting it.
This also keeps the value visible to the later pass that knows where replacing an expression with a literal is safe, so e.g. converting `n`
to an interface inside the closure still does not allocate.

**Nested closures**

A closure nested inside another closure reads its captured variables out of the enclosing closure's record. If the outer closure stopped
capturing a variable while the inner one still captured it, the inner closure would have nothing to read from. So the decision must depend
only on the canonical variable, so that every closure capturing it reaches the same decision.

**Inlined closures**

When a function containing a closure is inlined, the closure is copied at every call site. All these copies share a single linker symbol,
since the inline call stack hash is stripped when writing the object file (see [issue 60324][issue_60324]). Consider:

```go
func str(s string) func() string {
	return func() string { return s }
}

func main() {
	a, b := str("a"), str("b")
	...
}
```

After inlining, the copy in `str("a")` captures `"a"`, and the copy in `str("b")` captures `"b"`. If we specialized the body of each copy,
both would end up using the same symbol `str.func1·f`, and one of them would return the wrong string! So the compiler leaves inlined closures alone.

**Stale ReassignOracles**

Rewriting the closure body modifies the IR that the cached [ReassignOracle][reassign_oracle]s were computed from. Those oracles must be dropped,
so they will be recomputed from the up-to-date IR if needed later.

---

### Epilogue

This change is expected to be part of the Go 1.28 release.

It's always fun to close an issue that is older than many Go programmers' careers. For me, this is the oldest issue I have ever fixed, and the first 4-digit one, too.

Thanks for reading so far.

Till next time!

---

[issue_5370]: https://go.dev/issue/5370
[issue_4085]: https://go.dev/issue/4085
[issue_60324]: https://go.dev/issue/60324
[cl_783641]: https://go.dev/cl/783641
[reassign_oracle]: https://github.com/golang/go/blob/go1.27.1/src/cmd/compile/internal/ir/reassignment.go#L20
