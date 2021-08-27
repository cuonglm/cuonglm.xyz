---
title: "A long time go bug"
date: "2021-08-27"
tags: ["go", "golang"]
draft: false
---

---

### Prologue

An user reported a weird behavior of `go` command in [golang-dev mailing list][golang_dev_mailing_bug_report].

Here's an equivalent [program on playground][playground_bug_example], or inlined below, to reproduce the bug (with go 1.17 and below):

```go
package main

func main() {
	var s = []int{1, 2, 3}
	_ = (*[2]int)(s[1:])
}
-- go.mod --
module example.com

go 1.16
```

Running about program  give an error:

```text
$ go1.17 run -gcflags=-lang=go1.17 main.go
# command-line-arguments
./main.go:5:15: cannot convert s[1:] (type []int) to type *[2]int:
	conversion of slices to array pointers only supported as of -lang=go1.17
```

It's weird, isn't that we tell the compiler that we do want `-lang=go1.17`?

I [pointed out][golang_dev_mailing_cuonglm] why `go` behaves like that, and [@ianlancetaylor][ianlancetaylor] suggested that [it's a bug][golang_dev_mailing_ianlancetaylor].

### The bug

Since then, [an issue][go_issue_47682] was opened on golang Github issues. And the conclusion is that `cmd/go` is passing the wrong order of _flags_ to `cmd/compile`,
which causes the bug happens. This bug exists at least [since go version 1.5][go_15], and it's surprised to me that no one has reported before.

Note that this affects not only `-lang`, but also any compiler flags, example:

```text
$ cat main.go
package main

func main() {}
$ go1.17 build -a -x -work -gcflags=-buildid=foo main.go
WORK=/var/folders/rz/mxg9f82d2kl7jycb_rvrdp2r0000gn/T/go-build1278557172
...
<truncated output>
...
cd /Users/cuonglm
./sdk/go1.17/pkg/tool/darwin_arm64/compile -o $WORK/b001/_pkg_.a -trimpath "$WORK/b001=>" -shared -buildid=foo -p main -lang=go1.17 -complete -buildid VdM2tsNIpf8Qhrtrmssj/VdM2tsNIpf8Qhrtrmssj -goversion go1.17 -D _/Users/cuonglm -importcfg $WORK/b001/importcfg -pack ./main.go $WORK/b001/_gomod_.go
/Users/cuonglm/sdk/go1.17/pkg/tool/darwin_arm64/buildid -w $WORK/b001/_pkg_.a # internal
...
$ WORK=/var/folders/rz/mxg9f82d2kl7jycb_rvrdp2r0000gn/T/go-build1278557172
$ go1.17 tool buildid $WORK/b001/_pkg_.a
VdM2tsNIpf8Qhrtrmssj/VdM2tsNIpf8Qhrtrmssj
```

The `buildid` is not `foo` as expected.

Taking a quick look at the [cmd/go code][cmd_go_code], I [wonder](cuonglm_wonder) whether we can just swap the order of `gcflags` and `gcargs`.
[@jayconrod][jayconrod] said that (emphasis mine):

> The `flag` package requires that flags appear before _positional_ arguments.

It does not make sense, since when there's no _positional arguments_ involve in `gcargs`, it contains all the compiler flags that `cmd/go` wants to pass
to the compiler.

So it's clear that `cmd/go` is passing the wrong order of flags to the compiler.

### Fixing

In [CL 344909][cl_344909] that I sent to fix the bug, not only the wrong order was fixed, but also the misleading variable name `gcargs` was removed. With the fix,
`cmd/go` can now get it right:

```text
$ go build -a -x -work -gcflags=-buildid=foo main.go
WORK=/var/folders/rz/mxg9f82d2kl7jycb_rvrdp2r0000gn/T/go-build2599132168
...
<truncated output>
...
cd /Users/cuonglm
./sources/go/pkg/tool/darwin_arm64/compile -o $WORK/b001/_pkg_.a -trimpath "$WORK/b001=>" -p main -lang=go1.18 -complete -buildid zjrn3d_w3mjLM_-u-69Q/zjrn3d_w3mjLM_-u-69Q -shared -buildid=foo -D _/Users/cuonglm -importcfg $WORK/b001/importcfg -pack ./main.go $WORK/b001/_gomod_.go
/Users/cuonglm/sources/go/pkg/tool/darwin_arm64/buildid -w $WORK/b001/_pkg_.a # internal
...
$ WORK=/var/folders/rz/mxg9f82d2kl7jycb_rvrdp2r0000gn/T/go-build2599132168
$ go tool buildid $WORK/b001/_pkg_.a
foo
```

---
**NOTE**

While fixing this bug, I learn a cool trick to use `go build -n` for testing this kind of bug, instead of actually building the program.
That would speed up `go test cmd/go`, which sometimes will give you timeout when you run in your local machine.

---

### Epilogue

If you faced any issue after this fix (though, it's very unlikely, my 2¢), please [file an issue][golang_issue] or shoot me an email!

Thanks for reading so far.

Till next time!

---

[golang_dev_mailing_bug_report]: https://groups.google.com/g/golang-nuts/c/lhedc8YaWlM/m/exIWBOREBAAJ
[playground_bug_example]: https://play.golang.org/p/WhHyU3uUBp4
[golang_dev_mailing_cuonglm]: https://groups.google.com/g/golang-nuts/c/lhedc8YaWlM/m/R4Q3xuhHBAAJ
[golang_dev_mailing_ianlancetaylor]: https://groups.google.com/g/golang-nuts/c/lhedc8YaWlM/m/ER-r_JpPBAAJ
[ianlancetaylor]: https://github.com/ianlancetaylor
[go_issue_47682]: https://github.com/golang/go/issues/47682
[cmd_go_code]: https://github.com/golang/go/blob/7eaabae84d8b69216356b84ebc7c86917100f99a/src/cmd/go/internal/work/gc.go#L78-L160
[cuonglm_wonder]: https://github.com/golang/go/issues/47682#issuecomment-898804763
[jayconrod]: https://github.com/jayconrod
[go_15]: https://github.com/golang/go/blob/release-branch.go1.5/src/cmd/go/build.go#L2133
[cl_344909]: https://go-review.googlesource.com/c/go/+/344909
[golang_issue]: https://github.com/golang/go/issues
