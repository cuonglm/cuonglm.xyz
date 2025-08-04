---
title: "Safer unsafe.Add"
date: "2025-08-04"
tags: ["go", "golang", "unsafe", "compiler"]
draft: false
---

---

### Prologue

Go's [unsafe][unsafe_link] package is a powerful tool, allowing low-level memory manipulation when performance or specific system interactions demand it. 
However, with great power comes great responsibility – and potential for subtle bugs. To make pointer arithmetic a bit more manageable, Go 1.17 introduced
unsafe.Add ([CL 312214][cl_312214]).

This function was designed to offer a more convenient and ostensibly "safer" way to perform pointer arithmetic compared to the manual conversion dance of 
`unsafe.Pointer(uintptr(p) + uintptr(n))`. Essentially, `unsafe.Add(p, n)` achieves the same outcome but streamlines the syntax by removing the need to cast 
`unsafe.Pointer` to `uintptr` and back.

---

This streamlined approach is not just for convenience; it's also essential for enabling the Go runtime to support moving garbage collectors in the future.

---

### The problem


Dealing with raw pointers is inherently risky. To help developers catch common pointer-related errors, the Go compiler provides the valuable [`-d=checkptr`][checkptr_link] flag. 
This flag instruments your code with checks to ensure that `unsafe.Pointer` rules are being followed, often catching invalid memory accesses at runtime.

Consider the following problematic code:

```go
package main

import "unsafe"

var sink any

func main() {
	var x [2]int64
	// Attempting to access memory before the start of 'x'
	sink = unsafe.Pointer(uintptr(unsafe.Pointer(&x)) - 1)
	println("ops!")
}
```

When run with `-d=checkptr`, this code correctly panics, alerting us to an invalid pointer arithmetic operation:

```text
$ go run -gcflags=-d=checkptr -trimpath main.go
fatal error: checkptr: pointer arithmetic result points to invalid allocation

goroutine 1 gp=0xc0000061c0 m=0 mp=0x504100 [running]:
runtime.throw({0x488b15?, 0x10?})
	runtime/panic.go:1246 +0x48 fp=0xc0000506e0 sp=0xc0000506b0 pc=0x465fc8
runtime.checkptrArithmetic(0x70?, {0xc000050738, 0x1, 0xc000050740?})
	runtime/checkptr.go:69 +0x9c fp=0xc000050710 sp=0xc0000506e0 pc=0x40c1bc
main.main()
	./main.go:9 +0x3d fp=0xc000050750 sp=0xc000050710 pc=0x46e53d
runtime.main()
	runtime/proc.go:285 +0x2c7 fp=0xc0000507e0 sp=0xc000050750 pc=0x438fc7
runtime.goexit({})
	runtime/asm_amd64.s:1693 +0x1 fp=0xc0000507e8 sp=0xc0000507e0 pc=0x46b3e1
...
```

However, here's where the "safer" aspect of `unsafe.Add` fell short. An equivalent operation using `unsafe.Add` would, 
surprisingly, not trigger the `checkptr` panic:

```go
package main

import "unsafe"

var sink any

func main() {
	var x [2]int64
	// Attempting to access memory before the start of 'x'
	sink = unsafe.Pointer(unsafe.Add(unsafe.Pointer(&x), -1))
	println("ops!")
}
```

```text
$ go run -gcflags=-d=checkptr -trimpath main.go
ops!
```

The reason for this discrepancy was a simple oversight: the `-d=checkptr` instrumentation had not been extended to include `unsafe.Add`. 
This meant that `unsafe.Add`, despite its convenience, bypassed a crucial safety net that manual pointer arithmetic benefited from.

### Solution

The solution to this inconsistency was straightforward: integrate `unsafe.Add` into the existing `checkptr` validation logic. 
Since the compiler already had the necessary code paths for validating pointer arithmetic, it was a relatively simple task to 
reuse that machinery for `unsafe.Add`.

This important fix ([CL 692015][cl_692015]) is expected to be part of the Go 1.26 release, bringing `unsafe.Add` up to par with 
other `unsafe.Pointer` operations in terms of runtime safety checks.

### Epilogue

While I've continued to contribute to the Go compiler, it's been a while since I've had the chance to write a blog post about it. 
Still, moments like these are always a reminder of the joy of making even a small improvement to such a fundamental tool. 
Every contribution, no matter how seemingly minor, helps make Go a more robust and reliable language for everyone.

Thank you for reading so far!

Till next time!

---

[unsafe_link]: https://pkg.go.dev/unsafe
[unsafe_add_link]: https://pkg.go.dev/unsafe#Add
[cl_312214]: https://go.dev/cl/312214
[checkptr_link]: https://go.dev/cl/162237
[cl_692015]: https://go.dev/cl/692015
