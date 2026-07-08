---
layout: post
title: "Memory Profiling in Go: How One Line Allocated 687GB Of Data"
date: 2026-07-07 10:00:00 +0000
categories: [golang, performance]
tags: [go, pprof, profiling, memory, redis, codecrafters]
---

I built a multi-threaded [Redis clone in Go](https://github.com/solomon-os/redis-server) as part of the [CodeCrafters challenge](https://codecrafters.io). It supports a subset of Redis features; you can check the project page for details.

I’ve been reading on profiling and optimisations in Go. So I decided to profile my Redis implementation and check how I can improve it. Note: this Redis project is not production code; I used some lazy implementations where faster alternatives are available.

This post is about profiling and memory optimisations in Go. I decided to write this article because there aren’t many articles on profiling and optimisations in Go, so I wanted to share what I learnt. Let’s jump straight into it.

## Run the profiler
To start tuning the program, we have to enable profiling. I used `http/pprof` to profile the server in real time.

[see the diff](https://github.com/solomon-os/redis-server/commit/ae2b96bc3248a17f903c03f876d238fbbad3e8c4)

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
We need to run the application with the profiler enabled.
```bash
$ go build ./app/main.go
$ ./main -pprof-addr=:3000
```

Now that we have the profiler and application running, we’ll generate load using `redis-benchmark`. I created stress. A script that runs `redis-benchmark` using the command below, but the script is customised so we get a nice summary of the results.

```bash
$ redis-benchmark -p 6379 -t set,get,incr,lpush,lpop,lrange -n 300000 -r 1000000 -c 100 -d 128 -q;

```

We run the stress script and inspect the benchmark results. The tests and results are run on a 64GB RAM, 16-core M3 MAX.

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

From our benchmark, we see that LPUSH took a total of approx. 70 seconds to run and averaged around 4,100 rps, which is 14x slower than the next-slowest command (LRANGE). LPUSH is slower than other commands, especially RPUSH, which is similar but 33x faster. Looking at the numbers, we can guess something is wrong. Surely there must be an explanation for why RPUSH is 33x faster than LPUSH.

The answer is to measure and profile the application. Good thing if you’re following, we already set up profiling. Now, we’re going to profile the CPU for 100 seconds (just enough to account for the benchmark run) to identify bottlenecks in the application and why LPUSH is taking so long. The command calls pprof to profile the CPU and launch a ui to inspect the results of the profile at 4200.

```bash
go tool pprof "http://localhost:3000/debug/pprof/profile?seconds=100"
```

The first thing I usually do when looking at the results is to sort by the top, which shows the biggest bottlenecks in my code.

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
     1.08s  0.75% 97.13%      3.52s  2.43%  runtime.wbBufFlush1
```


From the results, the biggest CPU bottlenecks are thread sleep and wake churn due to the multi-threaded nature of our server and concurrent client handling. Our workload is not in the top 10 because it’s in memory operations and thus extremely fast for our sampler. This doesn’t explain why LPUSH is slower than the other commands; perhaps there’s another reason it's slow besides computation?
Could memory allocations be our bottleneck? Let’s profile and measure the memory consumption to see if it’s the bottleneck:


```bash
$ go tool pprof -http=:8080 http://localhost:3000/debug/pprof/allocs

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

voila! We see that there’s a problem with LPUSH. During our benchmark LPUSH, we allocated a total of 687 GB of memory. Hmm! That can’t be right. I measured a 64GB MacBook; ok, there's no way the application uses 687 GB of RAM. Yes, you’re right; 687 GB is the total allocated memory, not in-use memory. What that means is LPUSH allocated a total of 687GB for its operations, but much of this memory was freed by the garbage collector and then allocated and freed again. We can check the total number of in-use memory, i.e memory currently being used by the application:

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

As we can see. only 249mb of memory is being used by the application, which means all the extra memory allocated by lpush has been discarded by the garbage collector because it's not needed. We can also investigate further by looking into LPUSH to figure out where in the code all that memory is being allocated. We need to switch our index back to alloc_space so it shows the total allocated space/memory.

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


From the result above, we can see the line directly responsible for 671GB of the 687GB allocated by LPUSH. The remaining ~16GB comes from the smaller allocations around it.

So why does one line allocate 672GB of memory? Look at `line 162` closely:

```go
s.kvList[k] = append(v[popped:], s.kvList[k]...)
```

this prepends the new values by appending the *old list* onto them. `append` can't grow `v` in place to fit the whole list, so every single LPUSH allocates a brand new backing array big enough for `new values + entire existing list`, and copies every element of the old list into it. the old array becomes garbage immediately. next LPUSH? same thing again, except the list is now one element longer.

so the cost of each push grows with the size of the list. if a list grows by one element per push, the accounting looks like this:

```
push 1:  allocate array of 1
push 2:  allocate array of 2   (copy 1 old element)
push 3:  allocate array of 3   (copy 2)
...
push n:  allocate array of n   (copy n-1)
────────────────────────────────────────
total:   1+2+3+...+n = n(n+1)/2 ≈ n²/2
```

the list itself only ever *holds* O(n) memory — that's why in-use memory looked normal — but the *total allocated over time* is O(n²), and that's exactly the quantity `alloc_space` measures. our benchmark hammered 300k LPUSHes into a small set of lists, so the lists got long, every push paid to copy all of it, and n²/2 quietly integrated into 672GB. the number was never a size anything *was* — it's the area under the churn curve, all of it allocated, copied once, and handed straight to the garbage collector.

compare that with RPUSH, which was 33x faster:

```go
s.kvList[k] = append(s.kvList[k], v...)
```

same `append`, opposite behaviour. appending to the *existing* slice lets go's `append` use its growth strategy: when capacity runs out it allocates double, so reallocations only happen at sizes 1, 2, 4, 8, ... n. total copying across n pushes is about 2n — amortized O(1) per push. LPUSH can't benefit from that because the spare capacity `append` leaves is at the *tail* of the array, and prepending needs room at the *head*. every LPUSH lands on an exact-fit array and pays full price, every time.

two appends, six lines apart. one is O(1), the other is O(n²). you'd read this code a dozen times and never noticed but the profiler pointed at it in seconds.

So now the fix? How do we fix it? We need to figure out a way to avoid a new allocation for LPUSH. Hmm! What if we create an array with two pointers, left and right, and start allocating from the middle? For LPUSH, we allocate to the left; for RPUSH, we allocate to the right. When we reach the end of the left or right array, we reallocate a new array twice the size of the current one, copy the elements, and reset the pointers.
Yes, we could do this. There’s even a name for this data structure, and it’s called a ring buffer. This might not fit the textbook definition of a ring buffer, but this is a variant. You can see the [diff code here](https://github.com/solomon-os/redis-server/commit/ed0d5d243bed174975d63187ff49c12750a3d2f8).

We run the benchmark again.

```bash
running: -t set,get,incr,rpush,lpop,lrange_100 -n 300000 -r 1000000 -c 100 -d 128 against :6379

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
```

As you can see from the results, `LPUSH`is now over 33x faster. We can achieve this due to the optimisations we made to memory allocations. Now we run pprof to check our top 10 allocations.

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
as you can see `LPUSH` is no longer in the top 10 allocations. our `LPUSH` implementation is 33x faster simply because we are able to profile and improve.


*The code is from my [CodeCrafters Redis challenge](https://codecrafters.io) implementation in Go. Follow me on [GitHub](https://github.com/solomon-os) for the full source.*
