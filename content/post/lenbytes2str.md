---
title: "A practical optimization for Go compiler"
date: "2023-05-25"
tags: ["go", "golang", "compiler"]
draft: false
---

---

### Prologue

While reading random Go code, I see people writing something like [this][example_link]:

```go
if respbody, err := io.ReadAll(bodyReader); err == nil {
	resp.ContentLength = int64(len(string(respbody)))
	resp.Data = respbody
}
```

`respbody` is `[]byte`, so why not just simply do `int64(len(respbody))`, why there's string conversion there?

### The problem

For correctness, `len(respbody)` and `len(string(respbody))` will give you the same result, since when the 
[`len`][len_link] builtin will give you number of bytes in string.

However, the compiler is not taught to recognize that case (probably because no one should ever write that code?) 
so the string conversion still happens, making your code slower:

```text
$ cat x_test.go
```
```go
package x

import (
	"crypto/rand"
	"testing"
)

func f(s []byte) int {
	return len(string(s))
}

func g(s []byte) int {
	return len(s)
}

func BenchmarkX(b *testing.B) {
	length := 10
	s := randBytes(length)
	b.Run("len(string([]byte))", func(b *testing.B) {
		for i := 0; i < b.N; i++ {
			if f(s) != length {
				b.Fatal("unexpected result")
			}
		}
	})
	b.Run("len([]byte)", func(b *testing.B) {
		for i := 0; i < b.N; i++ {
			if g(s) != length {
				b.Fatal("unexpected result")
			}
		}
	})
}

func randBytes(n int) []byte {
	b := make([]byte, n)
	rand.Read(b)
	return b
}
```

```text
$ go1.20.4 test -bench=. -benchmem x_test.go
goos: linux
goarch: amd64
cpu: 11th Gen Intel(R) Core(TM) i7-1165G7 @ 2.80GHz
BenchmarkX/len(string([]byte))-8         	370225268	         3.345 ns/op	       0 B/op	       0 allocs/op
BenchmarkX/len([]byte)-8                 	1000000000	         0.2214 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	command-line-arguments	1.815s
```

---

*NOTE*

Looking at the assmebly output of function `f`, you will know the slow part.

### Solution

Normally, we should not make the compiler more complicated to handle an unusual case. However, a [quick search][github_sample] on Github
indicates that people is doing this unusual case in their code. Moreover, the fix would be quite simple, it's worth to do this practical
optimization for making user code better.

[CL 497276][cl_497276] was sent and submitted to address the issue, and will be part of go1.21 release.

The benchmark now show nearly identical running time for both cases:

```text
$ go version
go version devel go1.21-4042b90001 Wed May 24 15:04:44 2023 +0000 linux/amd64
$ go test -bench=. -benchmem x_test.go
goos: linux
goarch: amd64
cpu: 11th Gen Intel(R) Core(TM) i7-1165G7 @ 2.80GHz
BenchmarkX/len(string([]byte))-8         	1000000000	         0.2207 ns/op	       0 B/op	       0 allocs/op
BenchmarkX/len([]byte)-8                 	1000000000	         0.2192 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	command-line-arguments	0.490s
```

### Epilogue

Quote from [Keith Randall][khr_link]:

> Man, people are strange. Probably a holdover from Java where the length in bytes and runes is different.
I feel like this should be in a code-cleanliness sanitizer somewhere.

Happy optimizing.

Till next time!

---

[example_link]: https://github.com/ffuf/ffuf/blob/5fd821c17d5589c8f66211389d99e6f0885354d1/pkg/runner/simple.go#LL170C1-L173C3
[len_link]: https://pkg.go.dev/builtin#len
[github_sample]: https://github.com/search?q=%2Flen%5C%28string%5C%28.*Body%2F+language%3AGo&type=code
[cl_497276]: https://go-review.googlesource.com/c/go/+/497276
[khr_link]: https://github.com/randall77
