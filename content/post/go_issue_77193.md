---
title: "An old compiler bug with a very bad outcome"
date: "2026-04-08"
tags: ["go", "golang", "compiler"]
draft: false
---

---

### Prologue

This is the oldest compiler bug fix I have worked on so far.

---

### The problem

Consider following code:

```go
package p

func f() {
	var x int
	_ = [...][]*int{
		{&x},
		nil,
		nil,
		nil,
		nil,
	}
}
```

Compiling above code with `go1.24.13 tool compile -S x.go`:

```text
<unlinkable>.f STEXT size=79 args=0x0 locals=0x10 funcid=0x0 align=0x0
	0x0000 00000 (/home/cuonglm/x.go:3)	TEXT	<unlinkable>.f(SB), ABIInternal, $16-0
	0x0000 00000 (/home/cuonglm/x.go:3)	CMPQ	SP, 16(R14)
	0x0004 00004 (/home/cuonglm/x.go:3)	PCDATA	$0, $-2
	0x0004 00004 (/home/cuonglm/x.go:3)	JLS	72
	0x0006 00006 (/home/cuonglm/x.go:3)	PCDATA	$0, $-1
	0x0006 00006 (/home/cuonglm/x.go:3)	PUSHQ	BP
	0x0007 00007 (/home/cuonglm/x.go:3)	MOVQ	SP, BP
	0x000a 00010 (/home/cuonglm/x.go:3)	SUBQ	$8, SP
	0x000e 00014 (/home/cuonglm/x.go:3)	FUNCDATA	$0, gclocals·FzY36IO2mY0y4dZ1+Izd/w==(SB)
	0x000e 00014 (/home/cuonglm/x.go:3)	FUNCDATA	$1, gclocals·FzY36IO2mY0y4dZ1+Izd/w==(SB)
	0x000e 00014 (/home/cuonglm/x.go:4)	MOVQ	$0, <unlinkable>.x(SP)
	0x0016 00022 (/home/cuonglm/x.go:5)	CMPL	runtime.writeBarrier(SB), $0
	0x001d 00029 (/home/cuonglm/x.go:5)	PCDATA	$0, $-2
	0x001d 00029 (/home/cuonglm/x.go:5)	JEQ	55
	0x001f 00031 (/home/cuonglm/x.go:5)	NOP
	0x0020 00032 (/home/cuonglm/x.go:5)	CALL	runtime.gcWriteBarrier2(SB)
	0x0025 00037 (/home/cuonglm/x.go:5)	LEAQ	<unlinkable>.x(SP), AX
	0x0029 00041 (/home/cuonglm/x.go:5)	MOVQ	AX, (R11)
	0x002c 00044 (/home/cuonglm/x.go:5)	MOVQ	<unlinkable>..stmp_1(SB), CX
	0x0033 00051 (/home/cuonglm/x.go:5)	MOVQ	CX, 8(R11)
	0x0037 00055 (/home/cuonglm/x.go:5)	LEAQ	<unlinkable>.x(SP), AX
	0x003b 00059 (/home/cuonglm/x.go:5)	MOVQ	AX, <unlinkable>..stmp_1(SB)
	0x0042 00066 (/home/cuonglm/x.go:12)	PCDATA	$0, $-1
	0x0042 00066 (/home/cuonglm/x.go:12)	ADDQ	$8, SP
	0x0046 00070 (/home/cuonglm/x.go:12)	POPQ	BP
	0x0047 00071 (/home/cuonglm/x.go:12)	RET
	0x0048 00072 (/home/cuonglm/x.go:12)	NOP
	0x0048 00072 (/home/cuonglm/x.go:3)	PCDATA	$1, $-1
	0x0048 00072 (/home/cuonglm/x.go:3)	PCDATA	$0, $-2
	0x0048 00072 (/home/cuonglm/x.go:3)	CALL	runtime.morestack_noctxt(SB)
	0x004d 00077 (/home/cuonglm/x.go:3)	PCDATA	$0, $-1
	0x004d 00077 (/home/cuonglm/x.go:3)	JMP	0
```

At `0x0037`, the code executes `LEAQ <unlinkable>.x(SP), AX`, which calculate the memory address of local variable `x`, stores that address into the `AX` register.
Then at `0x003b`, the value in `AX` (which we know is the stack address of `x`) was moved into global static `..stmp_1`.

That is very, very bad behavior!

A stack pointer should not be treated like long-lived global data, because stack memory is temporary and tied to a specific call frame.
Once that frame is gone, the pointer becomes invalid. If a compiler mishandles that boundary, the resulting behavior can be subtle,
dangerous, and extremely hard to reason about.

This is the sort of issue that does not merely affect performance or code cleanliness. It goes straight to the heart of correctness.

---

### The fix

[CL 737821][cl_737821] is a straightforward fix, the compiler needed to ensure that static array initialization happens in the correct context,
instead of taking the wrong path and allowing the bad behavior to slip through.

---

### Epilogue

This bug is a good reminder that compiler work often uncovers deep historical quirks,

What makes it especially interesting is that the bug has apparently existed since the Go compiler was still written in C.
That means this issue has been hiding in the compiler for a very long time, surviving major implementation changes and countless other fixes along the way.

That alone makes it a fun one to look at.

Thanks for reading so far.

Till next time!

---

[cl_737821]: https://go-review.googlesource.com/c/go/+/737821
