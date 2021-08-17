---
title: "Fixing regression bug in Go compiler"
date: "2021-08-17"
tags: ["go", "golang", "compiler"]
draft: false
---

---

# Prologue

On August 15, 2021, [@johejo](https://github.com/johejo) filed an [issue](https://github.com/golang/go/issues/47712) about Go compiler internal compiler error.
This is a regression bug, since when it happens on current [tip](https://github.com/golang/go/commit/717894cf8024cfaad629f0e66a4b9bc123676be5), not in go1.17,
nor in older go versions.

To reproduce the bug:

```text
git clone https://gitlab.com/cznic/libc.git
cd libc/
go build ./...
```

and the error:

```text
# modernc.org/libc
walk
.   RECOVER tc(1) INTER-interface {} # libc.go:90:21 INTER-interface {}
./libc.go:90:21: internal compiler error: walkExpr: switch 1 unknown op RECOVER

<truncated error>
```

`git bisect` points me to [this commit](https://github.com/golang/go/commit/574ec1c6457c7779cd20db873fef2e2ed7e31ff1).

Since when I reviewed [CL 330192][cl_330192] before, I have some ideas in my mind, but I can't know for sure
until having a concrete, simpler reproducer.

Let try it!

# Reproducing the bug

First, look at the [line (./libc.go:90)](https://gitlab.com/cznic/libc/-/blob/e5917aaeaa0922ac3735d58466d78255cc237416/libc.go#L90) where the panic happens:

```go
if dmesgs {
	wd, err := os.Getwd()
	dmesg("%v: %v, wd %v, %v", origin(1), os.Args, wd, err)

	defer func() {
		if err := recover(); err != nil {
			dmesg("%v: CRASH: %v\n%s", origin(1), err, debug.Stack())
		}
	}()
}
```

Hmm..., nothing special, let try writing a minimal reproducer:

```go
package p

func f() {
	defer func() {
		_ = recover()
	}()
}
```

compile it:

```text
$ go tool compile p.go
$
```

Success!

So there must be something special. Looking at the original code again, I notice the condition `if dmesgs`, let see what is `dmesgs`.
With help from [gopls][gopls], I was able to see it's a [constant false](https://gitlab.com/cznic/libc/-/blob/e5917aaeaa0922ac3735d58466d78255cc237416/nodmesg.go#L9):

```go
const dmesgs = false
```

Let adjust the reproducer:

```go
package p

func f() {
	if false {
		defer func() {
			_ = recover()
		}()
	}
}
```

Now:

```text
$ go tool compile p.go
walk
.   RECOVER tc(1) INTER-interface {} # p.go:6:15 INTER-interface {}
p.go:6:15: internal compiler error: walkExpr: switch 1 unknown op RECOVER
<truncated output>
```

The bug is now reproducible, let's examine why this happens.

# Investigating

Since when the panic happens during [walk][walk] pass, let use `go tool compile -W` to examine the generated AST:

```text
$ go tool compile -W p.go
before walk f <nil>
after walk f <nil>

before walk f.func1
.   AS tc(1) # p.go:6:6
.   .   NAME-p._ tc(1) Offset:0 blank
.   .   RECOVER tc(1) INTER-interface {} # p.go:6:15 INTER-interface {}
walk
.   RECOVER tc(1) INTER-interface {} # p.go:6:15 INTER-interface {}
p.go:6:15: internal compiler error: walkExpr: switch 1 unknown op RECOVER
```

Oh! There're two things that caught my eyes:

 - The function `f` body is empty.
 - The funtion `f.func1` body contains `ORECOVER`, which must not happen during [walk][walk] pass.
   (Do you remember [CL 330192][cl_330192] I said above, go readind it and you would know why!)

`f` body is empty because the compiler run an early [deadcode][deadcode] pass after typechecking, thus it evals the `if false` condition and discards
the `if` body.

`f.func1` is the node that represents the function literal in `defer` call:

```go
defer func() {
	_ = recover()
}()
```

At this time, I have enough information to know how the bug happens:

 - In [noder][noder] pass, the compiler generates two function node, `f` and `f.func1`
 (`f.func1` is a hidden closure, because it's a closure inside a function).
 - When typechecking `f.func1`, the compiler push it to the compiling queue to compile later.
 - Then the compiler run [deadcode][deadcode] pass on all functions in compiling queue.
 - When deadcoding `f`, the `if false` block in `f` body is discarded.
 - The compiler run [escape analysis][escape] pass on all functions in compiling queue, [except for hidden closure](https://github.com/golang/go/blob/717894cf8024cfaad629f0e66a4b9bc123676be5/src/cmd/compile/internal/ir/scc.go#L59).
 It's because the hidden closure is part of its outer function, so it will be process when the compiler
 process the outer function anyway.

But in this case, the `f.func1` isn't part of `f` anymore, due to the [deadcode][deadcode] pass above. Thus, the desugaring `ORECOVER` during
[escape analysis][escape] never happens for `f.func1`, causing the compiler goes boom!

---
**NOTE**

If you turn `if false` into `if true`, and re-run `go tool compile -W`, you will have a clearer picture.

I would leave it as an excercise for the readers.

---

# Fixing

I sent [CL 342350](https://go-review.googlesource.com/c/go/+/342350) to fix the bug.

The idea is simple:

 - During [deadcode][deadcode] pass, keep track of which hidden closure is discarded.
 - When draining function to compile from compiling queue, skipping the discarded one.

# Epilogue

Working on the Go compiler is quite fun, and help me learning a lot of thing. I encourage you to give this a try, and hope you will have the same feel!

If you have any question, feel free to shoot me an email.

Thanks for reading so far.

Till next time!

---

[cl_330192]: https://go-review.googlesource.com/c/go/+/330192
[deadcode]: https://github.com/golang/go/tree/717894cf8024cfaad629f0e66a4b9bc123676be5/src/cmd/compile/internal/deadcode
[noder]: https://github.com/golang/go/tree/717894cf8024cfaad629f0e66a4b9bc123676be5/src/cmd/compile/internal/noder
[escape]: https://github.com/golang/go/tree/717894cf8024cfaad629f0e66a4b9bc123676be5/src/cmd/compile/internal/escape
[walk]: https://github.com/golang/go/tree/717894cf8024cfaad629f0e66a4b9bc123676be5/src/cmd/compile/internal/walk
[gopls]: https://github.com/golang/tools/tree/master/gopls
