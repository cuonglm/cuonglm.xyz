---
title: "A tale from The Go Programming Language Specification"
date: "2021-08-23"
tags: ["go", "golang"]
draft: false
---

---

### Prologue

The story started with [this nice poll][go101_poll] from [@go101][go101_twitter]:

```go
package main

import "reflect"

type T int

func (t T) M() { print(t) }

type S struct{ *T }

var t = new(T)
var s = S{T: t}

func main() {
	f := t.M
	g := s.M
	h := reflect.ValueOf(s).MethodByName("M").Interface().(func())
	*t = 5
	f()
	g()
	h()
}
```

The answer is `005`, but it's not the end of the story, since when [@go101 stated that][go101_opinion]:

> For the official Go compiler, the output is 005. "s.M" is promoted as "t.M" by the compiler. 
> However, personally, I think the output is compiler depended. 000 and 055 should be also right answers. 
> In fact, the two make the direct coding way and the reflection way more consistent.

My opinion is [different][cuonglm_opinion], any Go implementation, includes compiler+runtime, must produce `005` to obey the [Go specification][go_spec].

### Problem

Since then, I have a long dicussion with [@go101][go101_twitter], via some tweets and direct Twitter message. [@go101][go101_twitter]'s conclusion is that:

> s.M() is shorthand for s.T.M()
>
> !=
>
> s.M is shorthand for s.T.M

We will find out why the Go specification mandate that behavior and the output of:

```go
g := s.M
g()
```

must be `0`.

First, let look at the specification to see what is [Method values][go_method_values]:

> If the expression x has static type T and M is in the method set of type T, x.M is called a method value. 
> The method value x.M is a function value that is callable with the same arguments as a method call of x.M. 
> The expression x is evaluated and saved during the evaluation of the method value; the saved copy is then 
> used as the receiver in any calls, which may be executed later.

So in `g := s.M`, `s.M` is a [selector expression][go_selector_expression], and from the spec:

> For a value x of type T or \*T where T is not a pointer or interface type, x.f denotes the field or method at 
> the shallowest depth in T where there is such an f. If there is not exactly one f with shallowest depth, 
> the selector expression is illegal.

Thus for `s.M` to be valid, we need to find where is `M` appears in `S`. And the answer is `(*s.T).M`. Further, the declared
receiver parameter type of `M` is `T`, so `s` must be evaluated to something typed `T` during method value evaluation. And
the answer is also `*(s.T)`.

Here's a simple program to prove that:

```go
package main

import (
	"fmt"
	"go/ast"
	"go/importer"
	"go/parser"
	"go/token"
	"go/types"
)

const input = `
package p

type T int

func (t T) M() { print(t) }

type S struct{ *T }

func (s S) N() {}

var t = new(T)
var s = S{T: t}
`

var fset = token.NewFileSet()

func main() {
	f, err := parser.ParseFile(fset, "main.go", input, 0)
	if err != nil {
		panic(err)
	}

	conf := types.Config{Importer: importer.Default()}
	pkg, err := conf.Check("p", fset, []*ast.File{f}, nil)
	if err != nil {
		panic(err)
	}

	styp := pkg.Scope().Lookup("S").Type()
	fmt.Printf("Method set of %s:\n", styp)
	ms := types.NewMethodSet(styp)
	fmt.Printf("%v\n", ms.Lookup(pkg, "M").Obj())
	fmt.Printf("%v\n", ms.Lookup(pkg, "N").Obj())
}
```

Run above program output:

```text
Method set of p.S:
func (p.T).M()
func (p.S).N()
```

I think the only problem here is the lacking of example in the specification for method value receiver evaluation.
There're examples for method calls in the [Selectors][go_selector_expression] section:

```go
p.M0()       // ((*p).T0).M0()      M0 expects *T0 receiver
p.M1()       // ((*p).T1).M1()      M1 expects T1 receiver
```

Though we are stay here, honestly, I still don't understand why [@go101][go101_twitter] has this [conclusion](#problem).
The method value `s.M` and method call `s.M()` both must evaluate `s.M` the same way, it's just that in method value case,
the function value is never called.

### Solution

I opened an [issue][go_issue_47863] and sent a [CL][go_cl_344209] to add an example for method value evaluation, which
I hope will eliminate the confusion.

### Epilogue

Another fun story with Go and its community for me. How about you?

Thanks for reading so far.

Till next time!

---

[go101_twitter]: https://twitter.com/go100and1
[go101_poll]: https://twitter.com/go100and1/status/1427942567795073025
[go101_opinion]: https://twitter.com/go100and1/status/1428310079577608194
[go_spec]: https://golang.org/ref/spec
[cuonglm_opinion]: https://twitter.com/cuonglm_/status/1428391651660091398
[go_method_values]: https://golang.org/ref/spec#Method_values
[go_selector_expression]: https://golang.org/ref/spec#Selectors
[go_issue_47863]: https://github.com/golang/go/issues/47863
[go_cl_344209]: https://go-review.googlesource.com/c/go/+/344209
