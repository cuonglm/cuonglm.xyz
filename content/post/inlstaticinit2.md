---
title: "Enable inline static init for go 1.21"
date: "2023-04-19"
tags: ["go", "golang", "compiler"]
draft: false
---

---

### Prologue

In [previous post](/post/inlstaticinit), I talked about an upgrading node for go1.20, which has many regression bugs due to in-complete
implementation of inlining static init. The issue is now fixed, and will be enabled by default again for go1.21 version.

### The problem

The old implementation of inline static init use [typecheck.EvalConst][typecheck_evalconst_link], which will do constant-evaluation
for an expression. However, this function only evaluates the expression in constant context, while inline static init requires the
expression is evaluated in non-constant context.

For this given code:

```go
package p

var x = f(-1)

func f(x int) int {
	return 1 << x
}
```

It should be compiled and raise a runtime error at running time. With inline static init, the code will be rewritten to:

```go
package p

var x = 1 << -1

func f(x int) int {
	return 1 << x
}
```

which raise a compile-time error because of `1 << -1` expression.

### Solution

[CL 466277][cl_466277_link] implements a safer approach, making inline static init only handles cases that known to be safe at compile time.
Those cases are:

 - Conversion expressions: for example, `uint(x)` where `x` is an negative integer.
 - String concatenation: `"hello" + s` where `s` is a string.

That would prevents regressions, and still open room for supporting other arithmetic operations in the future.

---
**NOTE**

This approach also leads to a [better code generation for constant-fold switch][cl_484316_link], and [removing `typecheck.EvalConst`][cl_469595_link] entirely.

See this [CLs chain][cl_467016_link] for more details.

---

### Epilogue

It takes me a while for figuring out this approach for inline static init, which not only make user code better, but also improving the quality
of the go compiler code base. Please help testing with gotip (or RC version when it's out) and report any bugs that you seen.

Thank you all for reading so far!

Till next time!

---

[typecheck_evalconst_link]: https://cs.opensource.google/go/go/+/release-branch.go1.20:src/cmd/compile/internal/typecheck/const.go;l=355
[cl_466277_link]: https://go-review.googlesource.com/c/go/+/466277
[cl_467016_link]: https://go-review.googlesource.com/c/go/+/467016
[cl_484316_link]: https://go-review.googlesource.com/c/go/+/484316
[cl_469595_link]: https://go-review.googlesource.com/c/go/+/469595
