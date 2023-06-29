---
title: "Improving parallel calls from C to Go performance"
date: "2023-06-29"
tags: ["go", "golang", "cgo", "runtime"]
draft: false
---

---

### Prologue

Some weeks ago, my ex-colleague asked me a question about performance problem in his go/cgo code:

 - The Go code use [cgo][cgo_link],
 - The code perform parallel Go -> C and C -> Go calls.
 - The code run on bare metal with high number of CPUs.

Though we all know that `cgo` is slow, but the quirkish only happens when adding more CPUs.

Here's roughly English translated version of what's my ex-colleague told me:

> It's strange that when adding more CPUs, starting from 32, the performaces start decreasing.

### The problem

The `go` runtime runs `C` code in separated OS thread. If `C` code callback to `Go`, the runtime must
ensure the callback is run on the same OS thread. Otherwise, the scheduler could move the goroutine
which has been run the callback to different `M` (aka OS thread), prevent the callback from going back
to `C` code after finish.

Further, every calls from `C` to `Go` needs to check whether `go` runtime initialization done:


---

```go
fmt.Fprintf(fgcc, "\tsize_t _cgo_ctxt = _cgo_wait_runtime_init_done();\n")
```
[source](https://github.com/golang/go/blob/release-branch.go1.20/src/cmd/cgo/out.go#L1005)

---

So a [global mutex](runtime_cgo_global_mutex_link) is used for synchronization. This often does not matter,
because the lock is held only briefly. However, with code that does a lot of parallel calls from C to Go,
there could be a heavy contention on the mutex.

From this [pprof][pprof_link] result:

![pprof result](../go_issue_60961_pprof.png)

We can see that the code spending most of the time for lock/unlock/wait.

Here's a simplified program to reproduce the problem:

```go
package main

/*
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

extern void c_call_go_get_value(uint8_t* key32, uint8_t *value32);

static inline void get_value(uint8_t* key32, uint8_t *value32) {
    uint8_t value[32];
    c_call_go_get_value(key32, value);
    memcpy(value32, value, 32);
}
*/
import "C"
import (
	"fmt"
	"runtime"
	"sync"
	"time"
	"unsafe"
)

func main() {
	numWorkers := runtime.NumCPU()
	fmt.Printf("numWorkers = %v\n", numWorkers)
	numTasks := 10_000_000
	tasks := make(chan struct{}, numTasks)
	for i := 0; i < numTasks; i++ {
		tasks <- struct{}{}
	}
	close(tasks)
	start := time.Now()
	var wg sync.WaitGroup
	wg.Add(numWorkers)
	for i := 0; i < numWorkers; i++ {
		go func() {
			defer wg.Done()
			for range tasks {
				_ = getValue([32]byte{})
			}
		}()
	}
	wg.Wait()
	fmt.Printf("took %vms\n", time.Since(start).Milliseconds())
}

func getValue(key [32]byte) []byte {
	key32 := (*C.uint8_t)(C.CBytes(key[:]))
	value32 := (*C.uint8_t)(C.malloc(32))
	C.get_value(key32, value32)
	ret := C.GoBytes(unsafe.Pointer(value32), 32)
	C.free(unsafe.Pointer(key32))
	C.free(unsafe.Pointer(value32))
	return ret
}

//export c_call_go_get_value
func c_call_go_get_value(key32 *C.uint8_t, value32 *C.uint8_t) {
	key := C.GoBytes(unsafe.Pointer(key32), 32)
	_ = key
	value := make([]byte, 32)
	for i := 0; i < len(value); i++ {
		*(*C.uint8_t)(unsafe.Pointer(uintptr(unsafe.Pointer(value32)) + uintptr(i))) = (C.uint8_t)(value[i])
	}
}
```

Running with `go1.20.5`:

```text
$ go1.20.5 run main.go
numWorkers = 8
took 2687ms
```

### The fix

Since this is an initialization guard, we can use atomic operation to provide fast path in case the initialization
has been done. That would help reducing number of threads holding the mutex, prevent heavy contention.

[CL 505455][cl_505455_link] was sent to fixing this issue, reducing the running time of above program by `~1s`:

```text
$ go run main.go
numWorkers = 8
took 1706ms
```

The patch is also experiment by my ex-colleague in his environment, [reducing the mutex unlock from `~45%` to just `~17%`][patch_result_link].

### Epilogue

`cgo` is not `go`, and there's still a lot of room for improvement.

Thank you for reading so far!

Till next time!

---

[cgo_link]: https://pkg.go.dev/cmd/cgo
[pprof_link]: https://go.dev/blog/pprof
[runtime_cgo_global_mutex_link]: https://github.com/golang/go/blob/release-branch.go1.20/src/runtime/cgo/gcc_libinit.c#L18
[cl_505455_link]: https://go-review.googlesource.com/c/go/+/505455
[patch_result_link]: https://stackoverflow.com/questions/76509418/go-gi-pthread-mutex-unlock-takes-most-of-the-execution-time-when-using-c/76511790?noredirect=1#comment134971000_76511790
