# golang profiling
 - what is it?
 - what can it do?
 - how can I use it?

---

# golang profiling
If you leave with anything today: It's just stacktraces.

---
## what is it?

Profiling is Golang's way of measuring its runtime.
It does this by sampling goroutine callstacks at predefined triggers.
For example, a CPU profile captures callstacks of running goroutines every 10ms.
The heap profile captures the callstack of the allocating goroutine every 512KB.

---
## what can it do?

Give you X-ray vision.
A good code editor lets you peer through potential callstacks to see dependencies.
Profiling lets you look at the running system and see the interaction between the program and itself, and the program its environment.

---
## what can it do?

1. Profile the CPU: is your code using 50% of your CPU while running? this will tell you why.
1. Profile memory usage: Is something needlessly allocating a lot of memory and needing constant garbage collection?
1. Tracing (not profiling): What's the whole system doing? See the execution, from a DNS query's UDP syscall to the TLS handshake with the target server.

---
## how can I use it?
### the 7 HTTP profiler endpoints: Goroutine & Heap
1. /debug/pprof/goroutine
    - with `?debug=1` goroutines with identical stacks are combined with a counter, memory addresses are omitted: useful to find where all the goroutines are spending their time
    - with `?debug=2` each goroutine is printed on its own: useful to see what a particular goroutine is doing, and metadata about how long it's been alive
    - with `?debug=0` (same as no debug parameter) provides the same data as `?debug=1`, but in protobuf format for `go tool pprof` to read and interpret
2. /debug/pprof/heap
    - not human readable, exports a protobuf encoded heap profile for `go tool pprof` to interpret
    - roughly once per 512KB of allocated memory, a sample is taken
    - tightly coupled with the garbage collector, so this profile understands in use vs allocated memory

---
## how can I use it?
### the 7 HTTP profiler endpoints: The Ones Nobody Uses
3. /debug/pprof/threadcreate
    - human readable, similar to `/debug/pprof/goroutine?debug=1`, but for OS threads (`m` in stdlib)
4. /debug/pprof/block
    - not human readable, exports a protobuf encoded block profile for `go tool pprof` to interpret
    - disabled by default, use `runtime.SetBlockProfileRate` to enable
    - at the above rate samples are captured for goroutines that are blocked (e.g. network IO, channels)
5. /debug/pprof/mutex
    - not human readable, exports a protobuf encoded mutex profile for `go tool pprof` to interpret
    - disabled by default, use `runtime.SetBlockProfileRate` to enable
    - at the above rate samples are captured for goroutines that are blocked on mutex contention

---
## how can I use it?
### the 7 HTTP profiler endpoints: CPU & Trace
6. /debug/pprof/profile
    - not human readable, exports a protobuf encoded cpu profile for `go tool pprof` to interpret
    - CPU profile: every 10ms captures a stacktrace of all running goroutines
    - anything waiting is not captured. so if CPU usage isn't high, this profile will not have much insight
    - capture length can be tuned with `?seconds=n` parameter, defaults to 30s
7. /debug/pprof/trace
    - not human readable, exports a trace for `go tool trace` to interpret
    - has an appreciable performance overhead, most estimates put it at 5-15%
    - instruments the runtime scheduler for insight on goroutine interactions
    - has special support for syscalls, garbage collection, and blocking
    - useful for diagnosing latency (e.g. why did this request take 2 seconds to process?)
    - capture length can be tuned with `?seconds=n` parameter, defaults to 1s

---
## things I think you should know
- `self` vs `cum`(ulative) ~~ `flat` vs `sum`
- absolute import paths and symbolization with `trim_path` and `source_path`
- `inuse_space` vs `alloc_space`
- `pprof` and `trace` are packaged in `go tool` for convenience, but have their own lifecycles
- when, why, how to implement your own profiler
- the 7 HTTP profiler endpoints
- one profile at a time
- cgo will eat your lunch
- `http.DefaultServeMux` registers the endpoints in its `init()`, and That's a Threat Vector (TM)

---
## useful references
- [Julia Evans' overview blog post](https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/)
- [Felix Geisendörfer's notes on the profiler](https://github.com/DataDog/go-profiler-notes)
- [pretty much half of JBD's blog](https://rakyll.org/archive/)
- [Dave Cheney's "high performance go" workshop](https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html)

---