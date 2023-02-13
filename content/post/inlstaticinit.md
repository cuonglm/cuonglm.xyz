---
title: "go 1.20 upgrading note"
date: "2023-02-13"
tags: ["go", "golang", "compiler"]
draft: false
---

---

### Prologue

[go 1.20][go120_release_link] was released on Feb, 2, 2023, which includes some language changes, improvements to tooling and the library, 
and better overall performance.

However, there's a thing to notice once you start upgrading you projects to go1.20 toolchain.

### The problem

Starting in go1.20, global error created by calling [errors.New][errors_new_link] will be laid out in static data:

```shell
$ cat p.go
package p

import "errors"

var FooErr = errors.New("foo")

$ go1.19.5 build -gcflags=-S p.go 2>&1 | grep -q stmp && echo laid out in static data
$ go1.20 build -gcflags=-S p.go 2>&1 | grep -q stmp && echo laid out in static data
laid out in static data
```

This will reduce the init time, have no link-time overhead, and open room for the linker to perform deadcode elimination.

However, after the release, there're some regressions reported:

 - [#58293][issue_58293]
 - [#58325][issue_58325]
 - [#58339][issue_58339]
 - [#58439][issue_58439]

All of them releated to how the compiler handle the inline call at init time.

Though there're some attempts to fix, and backported to 1.20.1 minor release, there's still edge case discovered
while working on fixing the code.

The conclusion is [disabling the optimization for backporting to 1.20 release][disable_inlstaticinit_cl], then 
[re-enable again during 1.21 development cycle][disable_inlstaticinit_comment], so we have more time to re-implement 
or fixing the code properly.

### Solution

If you're going to upgrade your go toolchain to 1.20, remember to add `-gcflags=all=-d=inlstaticinit=0` to your CI process.

For 1.20.1 release, just upgrading as usual.

### Epilogue

Admittedly, this issue is a bit embarrassing, it will be fixed in go1.20.1, and can be workaround without requiring user to
make changes in the code. Please patient while the reported regressions are addressed.

And thank you all for reading so far!

Till next time!

---

[go120_release_link]: https://groups.google.com/g/golang-dev/c/0ECA5PrMuWs
[go120_release_note_link]: https://go.dev/doc/go1.20
[errors_new_link]: https://pkg.go.dev/errors#New
[issue_58293]: https://github.com/golang/go/issues/58293
[issue_58325]: https://github.com/golang/go/issues/58325
[issue_58339]: https://github.com/golang/go/issues/58339
[issue_58439]: https://github.com/golang/go/issues/58439
[disable_inlstaticinit_cl]: https://go-review.googlesource.com/c/go/+/467035
[disable_inlstaticinit_comment]: https://github.com/golang/go/issues/58439#issuecomment-1424581970
