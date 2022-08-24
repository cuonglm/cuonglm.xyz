---
title: "Go 1.19 atomic types"
date: "2022-08-24"
tags: ["go", "golang"]
draft: false
---

---

### Prologue

Go 1.19 was [released][go119_release_date], which have many changes to the toolchain and libraries. IMHO,  one of the coolest one
is the introduction of [atomic types][atomic_types_link].

### Atomic types

In pre-go1.19 world, you will write code like this:

```go
package p

import "sync/atomic"

type S struct {
	// doing is set to non-zero when doing something.
	// Read/Write operations are done atomically.
	doing uint32
	child atomic.Value // of *S, created lazily.
}

func (s *S) do() {
	atomic.StoreUint32(&s.doing, 1)
	// doing thing here
}

func (s *S) check() {
	if atomic.LoadUint32(&s.doing) != 0 {
		panic("already doing")
	}
}

func (s *S) getChild() *S {
	child := s.child.Load()
	if child == nil {
		child = new(S)
		s.child.Store(child)
	}
	return child.(*S)
}
```

So far so good, but:

 - There's nothing prevent others from accessing `doing` field non-atomically.
 - `doing` field should be a boolean, but the atomic APIs force us to make it an `uint32`,
 since when [sync/atomic@go1.18.5][sync_atomic_link] don't have atomic boolean oprations.
 - Boxing with `atomic.Value` requires us to always perform type assertion.

With atomic types, the code would become:

```go
package p

import "sync/atomic"

type S struct {
	doing atomic.Bool       // doing is set to non-zero when doing something.
	child atomic.Pointer[S] // created lazily.
}

func (s *S) do() {
	s.doing.Store(true)
	// doing thing here
}

func (s *S) check() {
	if s.doing.Load() {
		panic("already doing")
	}
}

func (s *S) getChild() *S {
	child := s.child.Load()
	if child == nil {
		child = new(S)
		s.child.Store(child)
	}
	return child
}
```

Now the code is self-document and type safe.

Another cool property (but maybe less noticable) is the alignment guarantee.

> [Int64][atomic_int64_link] and [Uint64][atomic_uint64_link] are automatically aligned to 64-bit boundaries in structs and allocated data, even on 32-bit systems.

That would simplify a lot of Go code out there and make Go developers life easier. For example, in go1.18, [sync.WaitGroup][sync_waitgroup_go118_link] was defined as:

```go
type WaitGroup struct {
	noCopy noCopy

	// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
	// 64-bit atomic operations require 64-bit alignment, but 32-bit
	// compilers only guarantee that 64-bit fields are 32-bit aligned.
	// For this reason on 32 bit architectures we need to check in state()
	// if state1 is aligned or not, and dynamically "swap" the field order if
	// needed.
	state1 uint64
	state2 uint32
}

// state returns pointers to the state and sema fields stored within wg.state*.
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	if unsafe.Alignof(wg.state1) == 8 || uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		// state1 is 64-bit aligned: nothing to do.
		return &wg.state1, &wg.state2
	} else {
		// state1 is 32-bit aligned but not 64-bit aligned: this means that
		// (&state1)+4 is 64-bit aligned.
		state := (*[3]uint32)(unsafe.Pointer(&wg.state1))
		return (*uint64)(unsafe.Pointer(&state[1])), &state[0]
	}
}
```

The complicated part of the code comes from the alignment requirement for atomic operations (see all comments above). Since when the code must be portable
and work on both 32-bit and 64-bit systems.

Using atomic types, the code can be simplified to just:

```go
type WaitGroup struct {
	noCopy noCopy

	state atomic.Uint64 // high 32 bits are counter, low 32 bits are waiter count.
	sema  uint32
}
```

`WaitGroup.state` is now guaranteed to have 64-bit alignment, so it's always safe to perform atomic operations on it. That also leads
to the removal of `WaitGroup.state()` method.

Doing less, enable more!

---
_Note_

See [CL 424835][cl_424835] for details and benchmark.

---

---

### Epilogue

I hope you enjoy go1.19, it's probably best Go release ever!

Thanks for reading so far.

Till next time!

---

[go119_release_date]: https://groups.google.com/g/golang-dev/c/thONLwEV3Pc/m/JHzRE6wIBQAJ
[atomic_types_link]: https://tip.golang.org/doc/go1.19#atomic_types
[sync_atomic_link]: https://pkg.go.dev/sync/atomic@go1.18.5
[sync_waitgroup_go118_link]: https://cs.opensource.google/go/go/+/refs/tags/go1.18.5:src/sync/waitgroup.go;l=20
[cl_424835]: https://go-review.googlesource.com/c/go/+/424835
[atomic_int64_link]: https://pkg.go.dev/sync/atomic@go1.19#Int64
[atomic_uint64_link]: https://pkg.go.dev/sync/atomic@go1.19#Uint64
