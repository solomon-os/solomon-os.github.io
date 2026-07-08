---
layout: post
title: "Memory Profiling and Optimisation in Go: How One Line Allocated 687GB"
date: 2026-07-07 10:00:00 +0000
categories: [golang, performance]
tags: [go, pprof, profiling, memory, redis, codecrafters]
---

I built a multi-threaded [Redis clone in Go](https://github.com/solomon-os/redis-server) as part of the [CodeCrafters challenge](https://codecrafters.io). It supports a subset of Redis features; you can check the project page for details.

I've been reading about profiling and optimisation in Go, so I decided to profile my Redis implementation and see how I could improve it. A note before we start: this is not production code. I made some deliberately lazy implementation choices, knowing faster alternatives existed.

There aren't many good articles about profiling real Go programs, so I wanted to share what I learnt by actually doing it. Let's jump straight in.

## Run the profiler

To start tuning the program, we have to enable profiling. I used `net/http/pprof` so I could profile the server in real time while it handles traffic ([see the diff](https://github.com/solomon-os/redis-server/commit/ae2b96bc3248a17f903c03f876d238fbbad3e8c4)).

```go
	"net/http"
	_ "net/http/pprof"

	"github.com/codecrafters-io/redis-starter-go/internal/server"
)

var pprofAddr = flag.String(
	"pprof-addr",
	"",
	"serve live pprof endpoints on this address (e.g. localhost:6060)",
)
func main() {
	flag.Parse()

	if *pprofAddr != "" {
		go func() {
			log.Println(http.ListenAndServe(*pprofAddr, nil))
		}()
	}

	server := server.New("6379")

	if err := server.ListenAndAccept(); err != nil {
		log.Fatalln(err)
	}
}

```

Then we run the application with the profiler enabled:

```bash
$ go build ./app/main.go
$ ./main -pprof-addr=:3000
```

## Benchmark first

With the profiler and application running, we need load; profiles of an idle server show nothing interesting. I wrote `stress.sh`, a small script that wraps `redis-benchmark` with the command below and prints a summary table of the results:

```bash
$ redis-benchmark -p 6379 -t set,get,incr,lpush,lpop,lrange -n 300000 -r 1000000 -c 100 -d 128 -q
```

All benchmarks in this post were run on a 16-core M3 Max with 64GB of RAM.

```bash
$ ./stress.sh
running: -t set,get,incr,lpop,lrange_100 -n 300000 -r 1000000 -c 100 -d 128 against :6379
```

| COMMAND                                |   REQUESTS |  TIME(s) |          RPS |  AVG(ms) |  P50(ms) |  P99(ms) |
|----------------------------------------|------------|----------|--------------|----------|----------|----------|
| SET                                    |     300000 |     2.19 |    137174.20 |    0.386 |    0.391 |    0.583 |
| GET                                    |     300000 |     2.15 |    139275.77 |    0.374 |    0.391 |    0.471 |
| INCR                                   |     300000 |     2.16 |    138696.25 |    0.376 |    0.391 |    0.479 |
| RPUSH                                  |     300000 |     2.13 |    140845.06 |    0.371 |    0.383 |    0.479 |
| LPOP                                   |     300000 |     2.14 |    139860.14 |    0.373 |    0.383 |    0.471 |
| LPUSH (needed to benchmark LRANGE)     |     300000 |    71.49 |      4196.16 |   23.813 |   24.511 |   50.879 |
| LRANGE_100 (first 100 elements)        |     300000 |     4.95 |     60642.81 |    1.174 |    0.647 |    8.295 |
| **TOTAL**                              |    2100000 |    87.21 |     24079.81 |          |          |          |

(The odd "needed to benchmark LRANGE" label is redis-benchmark's doing: whenever LRANGE is tested, it first runs an LPUSH phase to fill the lists. That pre-fill reports a normal stats row, so it doubles as our LPUSH benchmark.)

One row ruins the party. LPUSH took roughly 70 of the 87 seconds and averaged ~4,100 rps, 14x slower than the next-slowest command (LRANGE). Worse, RPUSH is nearly the same operation, and it's **33x faster**. Something is clearly wrong with LPUSH, and there has to be an explanation.

## Profile the CPU

The answer is to measure. We're going to record a 60-second CPU profile while the benchmark runs. The command below asks the live server for a profile and drops us into pprof's interactive terminal:

```bash
go tool pprof "http://localhost:3000/debug/pprof/profile?seconds=60"
```

The first thing I do with any profile is run `top10`, which ranks the biggest CPU consumers:

```bash
File: main
Type: cpu
Time: 2026-07-07 20:16:52 WAT
Duration: 60.19s, Total samples = 144.65s (240.31%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top10
Showing nodes accounting for 140.50s, 97.13% of 144.65s total
Dropped 219 nodes (cum <= 0.72s)
Showing top 10 nodes out of 81
      flat  flat%   sum%        cum   cum%
    63.05s 43.59% 43.59%     63.05s 43.59%  runtime.pthread_cond_signal
    34.34s 23.74% 67.33%     34.34s 23.74%  runtime.pthread_cond_wait
    15.34s 10.60% 77.93%     15.34s 10.60%  syscall.rawsyscalln
     9.84s  6.80% 84.74%      9.84s  6.80%  runtime.kevent
     7.14s  4.94% 89.67%      7.14s  4.94%  runtime.usleep
     3.42s  2.36% 92.04%      3.79s  2.62%  runtime.tryDeferToSpanScan
     2.95s  2.04% 94.08%      2.95s  2.04%  runtime.pthread_kill
     2.14s  1.48% 95.55%      2.33s  1.61%  runtime.typePointers.next
     1.20s  0.83% 96.38%      3.18s  2.20%  runtime.scanObject
     1.08s  0.75% 97.13%      1.08s  0.75%  runtime.wbBufFlush1
```

Interesting, and a dead end. The biggest CPU consumers are thread sleep/wake churn (`pthread_cond_signal`/`wait`) and network syscalls: the cost of a multi-threaded server juggling 100 concurrent clients. Our own functions don't even make the top 10, because command handling is in-memory work that's nearly invisible to a sampling profiler. Nothing here explains why LPUSH specifically is 33x slower than RPUSH; both would pay the same networking and scheduling costs.

So if LPUSH isn't burning CPU on computation, maybe it's doing something else expensive. Could memory allocation be the bottleneck? Let's profile the memory:

```bash
$ go tool pprof http://localhost:3000/debug/pprof/allocs

File: main
Type: alloc_space
Time: 2026-07-07 20:51:55 WAT
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top10
Showing nodes accounting for 714828.96MB, 99.93% of 715300.94MB total
Dropped 60 nodes (cum <= 3576.50MB)
Showing top 10 nodes out of 13
      flat  flat%   sum%        cum   cum%
687904.78MB 96.17% 96.17% 687911.28MB 96.17%  github.com/codecrafters-io/redis-starter-go/internal/store.(*Store).LPush
17938.07MB  2.51% 98.68% 17938.07MB  2.51%  strings.(*Builder).WriteString (inline)
 4166.50MB  0.58% 99.26%  4166.50MB  0.58%  io.WriteString
 4124.06MB  0.58% 99.84%  4135.08MB  0.58%  fmt.Sprintf
  476.01MB 0.067% 99.90%  4587.07MB  0.64%  github.com/codecrafters-io/redis-starter-go/internal/resp.BulkString (inline)
  212.03MB  0.03% 99.93% 715297.44MB   100%  github.com/codecrafters-io/redis-starter-go/internal/server.(*Server).handleConnection
    7.50MB 0.001% 99.93% 22474.64MB  3.14%  github.com/codecrafters-io/redis-starter-go/internal/resp.BulkStringArray
         0     0% 99.93% 710918.91MB 99.39%  github.com/codecrafters-io/redis-starter-go/internal/handler.(*Handler).Handle
         0     0% 99.93% 710595.88MB 99.34%  github.com/codecrafters-io/redis-starter-go/internal/handler.(*Handler).handleCommand
         0     0% 99.93% 687929.29MB 96.17%  github.com/codecrafters-io/redis-starter-go/internal/handler.(*Handler).handleLPush

```

Voila! There's our problem. During the benchmark, LPUSH allocated a total of **687GB** of memory.

Hmm, that can't be right. These benchmarks ran on a 64GB MacBook; there's no way the application used 687GB of RAM. And it didn't: `alloc_space` measures *total allocated* memory, not *in-use* memory. LPUSH really did allocate 687GB over the course of the run, but almost all of it became garbage moments after being allocated, and the garbage collector kept reclaiming it. We can confirm by switching to the in-use view:

```bash
(pprof) sample_index=inuse_space
(pprof) top10
Showing nodes accounting for 248.29MB, 99.60% of 249.29MB total
Dropped 15 nodes (cum <= 1.25MB)
Showing top 10 nodes out of 22
      flat  flat%   sum%        cum   cum%
  113.02MB 45.34% 45.34%   247.28MB 99.20%  github.com/codecrafters-io/redis-starter-go/internal/server.(*Server).handleConnection
   55.97MB 22.45% 67.79%    55.97MB 22.45%  strings.(*Builder).WriteString (inline)
   46.09MB 18.49% 86.28%    46.09MB 18.49%  github.com/codecrafters-io/redis-starter-go/internal/store.(*Store).setUnlocked
   14.50MB  5.82% 92.10%       15MB  6.02%  fmt.Sprintf
    9.12MB  3.66% 95.76%     9.12MB  3.66%  io.WriteString
    4.58MB  1.84% 97.59%     4.58MB  1.84%  github.com/codecrafters-io/redis-starter-go/internal/store.(*Store).LPush
    3.50MB  1.40% 99.00%    18.50MB  7.42%  github.com/codecrafters-io/redis-starter-go/internal/resp.BulkString (inline)
    1.50MB   0.6% 99.60%     1.50MB   0.6%  runtime.mallocgc
         0     0% 99.60%   125.14MB 50.20%  github.com/codecrafters-io/redis-starter-go/internal/handler.(*Handler).Handle
         0     0% 99.60%   125.14MB 50.20%  github.com/codecrafters-io/redis-starter-go/internal/handler.(*Handler).handleCommand
```

Only 249MB is actually being held by the application, and LPUSH's contribution to live memory is a whole 4.58MB. So LPUSH generated 687GB of pure garbage. To find where, we switch back to `alloc_space` and ask pprof to annotate the function's source, line by line:

```bash
(pprof) sample_index=alloc_space
(pprof) list LPush
Total: 698.54GB
ROUTINE ======================== github.com/codecrafters-io/redis-starter-go/internal/store.(*Store).LPush in internal/store/store.go
  671.78GB   671.79GB (flat, cum) 96.17% of Total
         .          .    140:func (s *Store) LPush(k string, v []string) int {
         .          .    141:	slices.Reverse(v)
         .     6.50MB    142:	s.Lock()
         .          .    143:	defer s.Unlock()
         .          .    144:
         .          .    145:	s.ensureList(k)
         .          .    146:
         .          .    147:	var popped int
         .          .    148:
         .          .    149:	// check for listeners
         .          .    150:	listeners, exist := s.listListeners[k]
         .          .    151:	if exist {
         .          .    152:		for i := 0; i < min(len(listeners), len(v)); i++ {
         .          .    153:			listeners[i] <- v[popped]
         .          .    154:			popped++
         .          .    155:		}
         .          .    156:		s.listListeners[k] = slices.Clone(s.listListeners[k][popped:])
         .          .    157:		if len(s.listListeners[k]) == 0 {
         .          .    158:			delete(s.listListeners, k)
         .          .    159:		}
         .          .    160:	}
         .          .    161:
  671.78GB   671.78GB    162:	s.kvList[k] = append(v[popped:], s.kvList[k]...)
         .          .    163:
         .          .    164:	return len(s.kvList[k]) + popped
         .          .    165:}
         .          .    166:
         .          .    167:func (s *Store) LRange(k string, start, end int) []string {
(pprof)
```

There it is: one line directly responsible for 671GB of the 687GB. The remaining ~16GB comes from the smaller allocations around it.

## Why does one line allocate 672GB?

Look at line 162 closely:

```go
s.kvList[k] = append(v[popped:], s.kvList[k]...)
```

This prepends the new values by appending the *old list* onto them. `append` can't grow `v` in place to fit the whole list, so every single LPUSH allocates a brand-new backing array big enough for `new values + entire existing list`, and copies every element of the old list into it. The old array becomes garbage immediately. The next LPUSH? Same thing again, except the list is now one element longer.

So the cost of each push grows with the size of the list. If a list grows by one element per push, the accounting looks like this:

```
push 1:  allocate array of 1
push 2:  allocate array of 2   (copy 1 old element)
push 3:  allocate array of 3   (copy 2)
...
push n:  allocate array of n   (copy n-1)
────────────────────────────────────────
total:   1+2+3+...+n = n(n+1)/2 ≈ n²/2
```

The list itself only ever *holds* O(n) memory (that's why the in-use numbers looked normal), but the *total allocated over time* is O(n²), and that's exactly the quantity `alloc_space` measures. Our benchmark hammered 300k LPUSHes into a small set of lists, so the lists got long, every push paid to copy all of it, and n²/2 quietly integrated into 672GB. That number was never a size anything *was*; it's the area under the churn curve, all of it allocated, copied once, and handed straight to the garbage collector.

Compare that with RPUSH, which was 33x faster:

```go
s.kvList[k] = append(s.kvList[k], v...)
```

Same `append`, opposite behaviour. Appending to the *existing* slice lets Go's `append` use its growth strategy: when capacity runs out, it allocates double, so reallocations only happen at sizes 1, 2, 4, 8, ... n. Total copying across n pushes is about 2n, amortized O(1) per push. LPUSH can't benefit from that, because the spare capacity `append` leaves is at the *tail* of the array, and prepending needs room at the *head*. Every LPUSH lands on an exact-fit array and pays full price, every time.

Two appends, six lines apart. One is O(1), the other is O(n²). I'd read this code a dozen times and never noticed. The profiler pointed at it in seconds.

## The fix

We need LPUSH to stop reallocating on every push. What if we create one array with two indices, `left` and `right`, and start writing from the middle? LPUSH writes leftward, RPUSH writes rightward, and only when either index hits the end of the array do we reallocate one twice the size, copy the elements into the middle, and reset the indices.

There's a name for this data structure: an **array-backed deque**, a close cousin of the ring buffer. Both ends now get the same amortized O(1) treatment that `append` was giving RPUSH. You can see the [full diff here](https://github.com/solomon-os/redis-server/commit/ed0d5d243bed174975d63187ff49c12750a3d2f8).

We run the benchmark again:

```bash
running: -t set,get,incr,rpush,lpop,lrange_100 -n 300000 -r 1000000 -c 100 -d 128 against :6379
```

| COMMAND                                |   REQUESTS |  TIME(s) |          RPS |  AVG(ms) |  P50(ms) |  P99(ms) |
|----------------------------------------|------------|----------|--------------|----------|----------|----------|
| SET                                    |     300000 |     2.16 |    139017.61 |    0.379 |    0.383 |    0.543 |
| GET                                    |     300000 |     2.13 |    140845.06 |    0.371 |    0.383 |    0.471 |
| INCR                                   |     300000 |     2.13 |    141110.08 |    0.370 |    0.383 |    0.471 |
| RPUSH                                  |     300000 |     2.15 |    139729.84 |    0.374 |    0.391 |    0.471 |
| LPOP                                   |     300000 |     2.15 |    139534.88 |    0.375 |    0.391 |    0.479 |
| LPUSH (needed to benchmark LRANGE)     |     300000 |     2.13 |    140581.06 |    0.372 |    0.383 |    0.471 |
| LRANGE_100 (first 100 elements)        |     300000 |     5.00 |     60024.01 |    1.190 |    0.639 |    8.823 |
| **TOTAL**                              |    2100000 |    17.85 |    117647.06 |          |          |          |

LPUSH went from 4,196 to 140,581 requests per second: **33x faster**, now indistinguishable from RPUSH. The whole suite dropped from 87 to 18 seconds. Purely from fixing how memory was allocated. One more look at the allocation profile to confirm:

```bash
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top10
Showing nodes accounting for 27244.29MB, 99.23% of 27456.18MB total
Dropped 53 nodes (cum <= 137.28MB)
Showing top 10 nodes out of 15
      flat  flat%   sum%        cum   cum%
17959.79MB 65.41% 65.41% 17959.79MB 65.41%  strings.(*Builder).WriteString (inline)
 4188.57MB 15.26% 80.67%  4189.57MB 15.26%  fmt.Sprintf
 4077.85MB 14.85% 95.52%  4077.85MB 14.85%  io.WriteString
  465.51MB  1.70% 97.22%  4643.58MB 16.91%  github.com/codecrafters-io/redis-starter-go/internal/resp.BulkString (inline)
  236.53MB  0.86% 98.08%   236.53MB  0.86%  strings.genSplit
  220.03MB   0.8% 98.88% 27453.68MB   100%  github.com/codecrafters-io/redis-starter-go/internal/server.(*Server).handleConnection
   88.50MB  0.32% 99.20%   325.03MB  1.18%  github.com/codecrafters-io/redis-starter-go/internal/parser.parseArray
    7.50MB 0.027% 99.23% 22558.36MB 82.16%  github.com/codecrafters-io/redis-starter-go/internal/resp.BulkStringArray
         0     0% 99.23% 23154.79MB 84.33%  github.com/codecrafters-io/redis-starter-go/internal/handler.(*Handler).Handle
         0     0% 99.23% 22829.76MB 83.15%  github.com/codecrafters-io/redis-starter-go/internal/handler.(*Handler).handleCommand
(pprof)
```

LPUSH is gone from the top 10 entirely. Total allocations for the same workload: 27GB, down from 715GB.

## Takeaways

- **An empty profile is still an answer.** The CPU profile "failing" to show my code wasn't a dead end. It was the profiler correctly telling me the cost wasn't computation, which is what justified looking at memory.
- **`alloc_space` and `inuse_space` answer different questions.** In-use tells you what's *holding* memory (footprint, leaks). Alloc tells you what's *generating garbage* (GC pressure). My 687GB existed in one view and was 4.58MB in the other. Both numbers were true.
- **Cumulative allocation is an area, not a size.** A quadratic algorithm can hide behind a perfectly normal live heap. The n²/2 sum is invisible to `top`, `htop`, and your dashboard; only the allocation profile sees it.

---

*The code is from my [CodeCrafters Redis challenge](https://codecrafters.io) implementation in Go. Follow me on [GitHub](https://github.com/solomon-os) for the full source.*
