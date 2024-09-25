---
title: "range-over-func bug in go1.23"
date: "2024-09-25"
tags: ["go", "golang", "compiler"]
draft: false
---

---

### Prologue

Since version 1.23, the Go language start allowing range over iterator functions:

 - [Changelog][rangefunc_changelog]
 - [Blog][rangefunc_blog]

If you have already known this feature, so far so good. Otherwise, I invite you to read
the two links above before continuing with this post.

The range-over-func feature is implemented inside the go compiler by rewriting:

```go
for x := range f {
    ...
}
```

roughly into:

```go
f(func(x T) bool {
    ...
})
```

The details does not matter for this post, you could just understand thatthe compiler frontend
simply rewrite the for-range loop into a function call with a function literal (aka closure) as
its argument.

### Problem

After go1.23 release, there's an [issue][issue_69434] which user reports a problem that range-over-func
loop is never terminated. After some analysis/triage, [Keith Randall raised his opinion][khr_comment] that
this looks like a compiler bug.

Some days later, [another issue][issue_69507] was reported, but this time with runtime mysterious crashes, and
with range-over-func, too. The issue was quickly marked as release blocker, since this sounds like a serious
mis-compilation of the compiler.

I decide to take a look, first with [issue 69434][issue_69434], because I already did some analysis at the day
the issue was reported.

A rule in the escape analysis of the go compiler: if variable x is re-assigned within a child closure of the function
where x is declared, then x must be heap allocated.

In term of code:

```go
func f() {
    var x *int
    func() {
        x = new(int) // must heap allocate if closure is not inlined
    }()
}
```

Let looks at this program:

```go
package main

import (
	"iter"
)

func All() iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := 0; i < 10; i++ {
			growStack(1)
			if !yield(i) {
				return
			}
		}
	}
}

type S struct {
	round int
}

func NewS(round int) *S {
	s := &S{round: round}
	return s
}

func (s *S) check(round int) {
	if s.round != round {
		panic("bad round")
	}
}

func f() {
	rounds := 0
	s := NewS(rounds)
	s.check(rounds)

	for range All() {
		s.check(rounds)
		rounds++
		s = NewS(rounds)
		s.check(rounds)
	}
}

func growStack(i int) {
	if i == 0 {
		return
	}
	growStack(i - 1)
}

func main() {
	f()
}
```

Check with `go tool compile -m`:

```
$ go1.23.1 build -gcflags=-m -trimpath main.go                                                                                                                                                             ✹
# command-line-arguments
./main.go:7:6: can inline All
./main.go:22:6: can inline NewS
./main.go:27:6: can inline (*S).check
./main.go:53:6: can inline main
./main.go:35:11: inlining call to NewS
./main.go:36:9: inlining call to (*S).check
./main.go:38:15: inlining call to All
./main.go:39:10: inlining call to (*S).check
./main.go:41:11: inlining call to NewS
./main.go:42:10: inlining call to (*S).check
./main.go:8:14: yield does not escape
./main.go:8:9: func literal escapes to heap
./main.go:23:7: &S{...} escapes to heap
./main.go:27:7: s does not escape
./main.go:29:9: "bad round" escapes to heap
./main.go:8:14: yield does not escape
./main.go:35:11: &S{...} does not escape
./main.go:36:9: "bad round" escapes to heap
./main.go:38:2: func literal does not escape
./main.go:39:10: "bad round" escapes to heap
./main.go:41:11: &S{...} does not escape
./main.go:42:10: "bad round" escapes to heap
./main.go:38:15: func literal does not escape
```

You can see `./main.go:41:11: &S{...} does not escape` while it must be.

Now run it:

```
$ go1.23.1 run -trimpath main.go                                                                                                                                                                       1 ↵ ✹
panic: bad round

goroutine 1 [running]:
main.(*S).check(...)
	./main.go:29
main.f-range1(0xc0000546e8?)
	./main.go:39 +0xb8
main.f.All.func1(0xc000054710)
	./main.go:11 +0x48
main.f()
	./main.go:38 +0x88
main.main()
	./main.go:54 +0xf
exit status 2
```

Aha, the problem is found.

For a function `f`, the closures within this function will be named: `f.funcN`, where `N` is start
at 1 and increasing each time another closure seen. This naming scheme is important for the escape
analysis, since this helps it to check whether the closure contained within function.

Look at the stacktrace above, you could see the closure `main.f-range1`, which is not followed the
naming pattern, so the escape analysis does not know that this is a closure in function f, causing
wrong allocation decision.

---

How I detect the bug immediately is another story, since it requires you to be familiar with the go
compiler development.

If any of the readers have any questions, feel free to ping me on [X][cuonglm_x] or shoot me an email.

---

### Solution

The fix is quite trivial, just extend the escape analysis to recognize new naming scheme of range-over-func.
This is done in [CL 614096][cl_614096], and will be part of go1.23 next minor release (probably go1.23.2).

```text
$ go1.23.1 build -trimpath -gcflags=-m main.go &> bad
$ go build -trimpath -gcflags=-m main.go &> good
$ grep -v -Ff bad good
./main.go:41:11: &S{...} escapes to heap
```

### Epilogue

Another day, another fix for the go compiler.

Thanks for reading so far.

Till next time!

---

[rangefunc_changelog]: https://tip.golang.org/doc/go1.23#language
[rangefunc_blog]: https://tip.golang.org/blog/range-functions
[issue_69434]: https://github.com/golang/go/issues/69434
[issue_69507]: https://github.com/golang/go/issues/69507
[khr_comment]: https://github.com/golang/go/issues/69434#issuecomment-2347979464
[cuonglm_x]: https://x.com/cuonglm_
[cl_614096]: https://go-review.googlesource.com/c/go/+/614096
