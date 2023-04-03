---
author: ""
date: ""
paging: "%d / %d"
---

# golang profiling
 - what is it?
 - what can it do?
 - how can I use it?

---

# Ground rules
 - profiling is just stacktraces.
 - listed implementation details are for reference. don't worry about them.

---
## what is it?

Profiling is Golang's way of measuring its runtime, It does this by sampling goroutine callstacks at predefined triggers.
for example:
 - the CPU profile captures callstacks of running goroutines every 10ms.
 - the heap profile captures the callstack of the allocating goroutine every 512KB.

---
## what can it do?

 - Profiling lets you look at the running system and see the interaction between the program and itself, and the program its environment.
 - Profiling pierces through the abstraction of libraries.
 - It gives you X-ray vision.

---
## what are common usages?

1. Profile the CPU: is your code using 50% of your CPU while running? this will tell you why.
1. Profile memory usage: Is something allocating a lot of objects and needing constant garbage collection?
1. Tracing (not profiling): What's the whole runtime doing? See the execution, from a DNS query's UDP syscall to the TLS handshake with the target server.

---
## how can I use it?
### Adding the HTTP pprof endpoint
```go
import(
    "net/http"
    _ "net/http/pprof" // contains an init to register routes on DefaultServeMux
)
func main() {
    http.ListenAndServe("127.0.0.1:8080", http.DefaultServeMux)
}
```
---
#### What it looks like in-browser
```
/debug/pprof/

Types of profiles available:
Count Profile
0     allocs
0     block
0     cmdline
10    goroutine
0     heap
0     mutex
0     profile
12    threadcreate
0     trace
full goroutine stack dump
```
---
## how can I use it?
### Query parameters
1. `debug`
   - `?debug=1`: identical stacks are coalesced
   - `?debug=2`: all stacks are individually printed, allowing memory addresses to be printed
   - `?debug=0`: binary response of a gzip-encoded .proto file
2. `seconds`
   - `profile` and `trace` both start sampling from the moment the HTTP endpoint is hit, this changes their sample time.
---
## how can I use it?
### the 8 HTTP profiler endpoints: Goroutine & Heap
1. `/debug/pprof/goroutine`
    - Stop The World (STW): what is the stacktrace of every goroutine?
    - ex: am I leaking goroutines? how many inflight network requests are there?
2. `/debug/pprof/heap` == `/debug/pprof/allocs` 
    - once per 512KiB of allocated memory, a sample is taken
    - the only difference between `heap` and `allocs` is whether inuse or alloc space is shown by default in `pprof`
    - ages like wine
    - ex: am I not caching something that should be? am I growing buffers too often?
---
## how can I use it?
### the 8 HTTP profiler endpoints: The Ones Nobody Uses
3. `/debug/pprof/threadcreate`
    - what golang stacktrace resulted in the creation of a new OS thread 
    - [useless since 2013](https://github.com/golang/go/issues/6104)
4. `/debug/pprof/block`
    - disabled by default, use `runtime.SetBlockProfileRate` to enable
    - at the above rate samples are captured for goroutines that are blocked (e.g. network IO, channels)
5. `/debug/pprof/mutex`
    - disabled by default, use `runtime.SetMutexProfileFraction` to enable
    - at the above rate samples are captured for goroutines that are blocked on mutex contention
6. `/debug/pprof/cmdline`
    - what arguments was this go program run with?
    - Not even remotely a profile.
---
## how can I use it?
### the 8 HTTP profiler endpoints: CPU & Trace
7. `/debug/pprof/profile`
    - CPU profile: every 10ms captures a stacktrace of all running goroutines
    - anything waiting is not captured. so if CPU usage isn't high, this profile will not give much insight
    - capture length can be tuned with `?seconds=n` parameter, defaults to 30s
    - ex: am I using reflections too much? spending too much time managing goroutines?
8. `/debug/pprof/trace`
    - does not respect `debug` parameters, exports a trace for `go tool trace` to interpret
    - has an appreciable performance overhead, most estimates put it at ~15%
    - the runtime contains 49 flavors of events (as of 1.20) that it will emit when tracing, like entering a syscall or GC start
    - has special support for syscalls, garbage collection, and blocking
    - capture length can be configured with `?seconds=n` parameter, defaults to 1s
    - ex: why did this request take 2 seconds to process?

---
## Example CPU Profile
### Startup
`go tool pprof my-profile.profile`
```
Type: cpu
Time: Apr 1, 2023 at 5:48pm (MST)
Duration: 74.25s, Total samples = 103.91s (139.94%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) 
```

---
## Example CPU Profile
### Top
```
Type: cpu
Time: Apr 1, 2023 at 5:48pm (MST)
Duration: 74.25s, Total samples = 103.91s (139.94%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 101.68s, 97.85% of 103.91s total
Dropped 129 nodes (cum <= 0.52s)
Showing top 10 nodes out of 46
      flat  flat%   sum%        cum   cum%
    87.53s 84.24% 84.24%     87.53s 84.24%  runtime.cgocall
     6.36s  6.12% 90.36%      7.33s  7.05%  main.LCSubs
     2.67s  2.57% 92.93%      2.67s  2.57%  runtime.memmove
     1.71s  1.65% 94.57%     85.13s 81.93%  main.writeNonmatchesToFile
     0.99s  0.95% 95.52%      0.99s  0.95%  runtime.madvise
     0.70s  0.67% 96.20%      0.73s   0.7%  main.writeNonmatchesToFile.func1
     0.55s  0.53% 96.73%      0.55s  0.53%  runtime.usleep
```

---
## things I think you should know
- `self`==`flat`, `cum`(ulative)=="this function, and below in the callstack"
- absolute import paths and symbolization with `trim_path` and `source_path`
- `list` lets you read the code in context instead of just stacktraces
- `inuse_space` vs `alloc_space`, `inuse_objects` vs `alloc_objects`
- you can implement your own profiler to track access to limited resources (e.g. database connections)
- only run one profile at a time
- cgo will eat your lunch
- `net/http/pprof` registers debug endpoints to `DefaultServeMux` in its `init()`
- `go tool pprof` can read directly from the HTTP endpoint by streaming the gzipped protobuf
- if you have the binary, pprof will disassemble it alongside the source code
- pprof's interactive web server (`-http`) is overrated

---
## useful references
- [Julia Evans' overview blog post](https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/)
- [Felix Geisendörfer's notes on the profiler](https://github.com/DataDog/go-profiler-notes)
- [pretty much half of JBD's blog](https://rakyll.org/archive/)
- [Dave Cheney's "high performance go" workshop](https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html)
- [tracing events](https://github.com/golang/go/blob/go1.20/src/runtime/trace.go#L25-L74)
