---
categories:
- performance
date: 2017-02-20T00:00:00Z
title: Be Careful What You Benchmark
slug: be-careful-what-you-benchmark
---

I have a project where I need to generate unique database keys at the client rather than using autoincrementing keys, in order to support sharding at the client. (I'm effectively re-implementing [Etsy's MySQL master-master scheme](https://codeascraft.com/2012/04/20/two-sides-for-salvation/) if you want to know more.) I don't want to use UUIDs because they're 128 bits instead of 64 bits, and because they're not time ordered they can result in [poor insert performance](https://cjsavage.com/guides/mysql/insert-perf-uuid-vs-ordered-uuid-vs-int-pk.html). So I'm implementing Twitter's now-retired [Snowflake](https://github.com/twitter/snowflake/tree/snowflake-2010), which produces roughly-ordered but unique IDs. In so doing, I ran into an interesting performance riddle that's worth sharing.

> Be forewarned this whole post is a bit of a shaggy dog story. There isn't a satisfying conclusion with a smoking gun at the end, just a place to start work and some lessons about being careful around your assumptions about your test environment. But if you're interested in benchmarking golang code and how virtualization can foul the results, you might find it worth your while.


## Making Snowflakes

The Snowflakes I'm generating are 64 bit unsigned ints. The original Twitter Snowflake used int64, or 63 useful bits plus the unused sign bit, for what I assume have to do with the same legacy reasons they cited for wanting 64 bits in the first place. The first 41 bits are milliseconds since a custom epoch, which gives me 69 years worth of IDs. The major constraint here is that we have to ensure we're using a clock that never goes backwards within the lifetime of a process. The next 11 bits are a unique worker ID, which gives me 2048 worker processes.

These worker IDs are the sole point of coordination required; whatever schedules the worker will need to reserve the worker IDs. In practice this is 100 times more workers than I'm likely to need for this project but that gives me plenty of headroom. It's safe to recycle worker IDs so long as they're unique among the *running* workers. The last 12 bits of the ID are a monotonic sequence ID. Each worker can therefore generate a maximum of 4096 IDs per millisecond.

That last requirement means that our ID "server" for each worker process needs to make sure it doesn't issue sequence IDs larger than 4095 within a single millisecond, otherwise we overflow and end up repeating IDs. I also want the server to be safe to use by multiple threads (or goroutines in this case because I'm writing it in golang because... I'm too dumb to use generics?). I poked around on GitHub for existing implementations first without much satisfaction.

One version was a straight port of Twitter's Scala code which looked OK but obviously wasn't very idiomatic golang code. Another was laughably unthreadsafe. Another had an API that tangled up the server setup with implementation details for their worker ID generation. And so on. All the implementations I found return a `uint64` or `int64` directly without wrapping it. I want any consuming code I write later to notice if I accidentally do something stupid like try to do math on an ID, so embedding the `uint64` in a `Snowflake` type gives me a modicum of type safety. There was also a nagging doubt in my mind about the number of times the original design was checking the time. (*Ooo... foreshadowing!*)

So I implemented my own version, where the server listens on a channel for millisecond ticks and resets the sequence ID counter. In theory this would avoid a lot of calls to get the time but would involve an extra mutex per millisecond-long cycle. Because I'm paranoid I wanted to make sure I wasn't missing some profound performance regression in dumping the original Twitter design.

I also took the library that was the closest direct implementation of Twitter's design, [`sdming/gosnow`](https://github.com/sdming/gosnow), and ported it to use the same API as my own, and also took a slightly different approach, Matt Ho's [`savaki/snowflake`](https://github.com/savaki/snowflake), and gave it the same treatment. After writing a bunch of test code to make sure all three implementations were correct in behavior, I put them all through the same benchmark (each with different names of course):

``` go
var result Snowflake // avoids compiler optimizing-away output

func BenchmarkSnowflakeGen(b *testing.B) {
	server := NewServer()
	var id Snowflake
	b.ResetTimer()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			id = server.GetSnowflake()
		}
	})
	result = id
}
```

## The Benchmarks

Before I ran the benchmark I wanted to do some reasoning from first principles. Because I can't generate more than 4096 IDs per millisecond, I know that we have a lower bound of 244 nanoseconds assuming we can saturate the server. A single thread could manage this only if all the rest of the code was free. So that's our target for multi-threaded code. I ran through all three benchmarks on my Linux development machine and I was pretty happy with the result.

```
$ cat /proc/cpuinfo | grep "model name" | head -1
model name      : Intel(R) Core(TM) i7-4712HQ CPU @ 2.30GHz

$ nproc
8

$ go test -v -run NULL -benchmem -bench .
BenchmarkSavakiSnowflakeGen-8    5000000     334 ns/op    0 B/op   0 allocs/op
BenchmarkSnowflakeGen-8          5000000     244 ns/op    0 B/op   0 allocs/op
BenchmarkTwitterSnowflakeGen-8   5000000     292 ns/op    0 B/op   0 allocs/op
PASS
ok      snowflake    5.247s

$ GOMAXPROCS=4 go test -v -run NULL -benchmem -bench .
BenchmarkSavakiSnowflakeGen-4    5000000     360 ns/op    0 B/op   0 allocs/op
BenchmarkSnowflakeGen-4          5000000     244 ns/op    0 B/op   0 allocs/op
BenchmarkTwitterSnowflakeGen-4   5000000     250 ns/op    0 B/op   0 allocs/op
PASS
ok      snowflake    5.148s
```

My own implementation hits the lower-bound benchmark! But the Savaki and Twitter clone library both do respectably, so my concerns about lots of calls to `gettimeofday` seem to have been for nothing. Note that I've run this on both 8 cores and 4 (via `GOMAXPROCS`) to check for hidden [Amdahl's Law](https://en.wikipedia.org/wiki/Amdahl's_law) problems. This and all other tests are running under golang 1.7.

All done, right? Well I'm not going to be deploying on my 8-core Linux laptop. I'll probably be deploying to a Docker container on [Triton](https://www.joyent.com/triton), so I wanted to make sure the relative benchmark would hold up there. I didn't happen to have Triton keys on my Linux laptop so I switched over to my Mac that did. But first I needed to build a container with the code so I did that and sanity-checked it under Docker for Mac. At this point someone in the audience is slapping their head and saying "duh!", to which I can only say "ssh... you'll spoil it for everyone else!"

```
$ nproc
4

$ go test -v -run NULL -benchmem -bench .
BenchmarkSavakiSnowflakeGen-4     500000    3228 ns/op    0 B/op   0 allocs/op
BenchmarkSnowflakeGen-4          5000000     273 ns/op    0 B/op   0 allocs/op
BenchmarkTwitterSnowflakeGen-4   1000000    1086 ns/op    0 B/op   0 allocs/op
PASS
ok      _/go/snowflake       4.430s
```

*Wat?*

> Ok, for some reason when this post was first shared on Twitter a bunch of dumbasses who didn't bother to read the article decided to heap snark upon it because "hurr hurr I ran my software under virtualization and it was slower." That's not the interesting thing here! The interesting thing, which I'm sure I don't have to spell out to you because you've read this far, is that one implementation performed only a tiny bit worse while the other implementations performed much much worse.

Here's where we go off the rails a bit. The first thing I did, which wasn't all that necessary, was to run the benchmark in other environments to make sure I hadn't made some strange mistake.  There's no allocation in any of the algorithms, so from here on out I'll elide the `-benchmem` data.

Here I run it on the MacOS host:

```
$ sysctl -n hw.ncpu
8

$ sysctl -n machdep.cpu.brand_string
Intel(R) Core(TM) i7-4750HQ CPU @ 2.00GHz

$ go test -v -run NULL -bench .
BenchmarkSavakiSnowflakeGen-8       5000000      291 ns/op
BenchmarkSnowflakeGen-8             5000000      256 ns/op
BenchmarkTwitterSnowflakeGen-8      5000000      291 ns/op
PASS
ok      snowflake    5.045s

$ GOMAXPROCS=4 go test -v -run NULL -bench .
BenchmarkSavakiSnowflakeGen-4       5000000      342 ns/op
BenchmarkSnowflakeGen-4             5000000      245 ns/op
BenchmarkTwitterSnowflakeGen-4      5000000      244 ns/op
PASS
ok      snowflake    4.989s
```

Here I run it on a `c4.xlarge` on AWS:

```
$ nproc
4

$ cat /proc/cpuinfo | grep "model name" | head -1
model name      : Intel(R) Xeon(R) CPU E5-2666 v3 @ 2.90GHz

$ go test -v -run NULL -bench .
BenchmarkSavakiSnowflakeGen-4       5000000      373 ns/op
BenchmarkSnowflakeGen-4             5000000      263 ns/op
BenchmarkTwitterSnowflakeGen-4     10000000      247 ns/op
PASS
ok      _/go/snowflake       6.228s
```

And here I run it on a Triton `g4-highcpu-4G`. Note that in this case we're running on a bare metal container where we can see all cores of the underlying host but the process is only scheduled time across those processors according to the [Fair Share Scheduler (FSS)](https://wiki.smartos.org/display/DOC/Managing+CPU+Cycles+in+a+Zone). This instance has CPU shares very roughly equivalent to the same 4 vCPU that the AWS instance has, albeit on slightly slower processors and at half the price. This benchmark is also *pure* CPU which means that Triton's profound I/O advantages in avoiding hardware virtualization don't come into play at all! So don't try to directly compare the "marketing number" of 4 vCPU.

```
$ nproc
48
$ cat /proc/cpuinfo | grep "model name" | head -1
model name      : Intel(r) Xeon(r) CPU E5-2690 v3 @ 2.60GHz

# wow, lots of Amdahl's law appears with 48 processors visible!
$ go test -v -run NULL -bench .
BenchmarkSavakiSnowflakeGen-48      1000000     1280 ns/op
BenchmarkSnowflakeGen-48            1000000     1135 ns/op
BenchmarkTwitterSnowflakeGen-48     1000000     1414 ns/op
PASS
ok      _/go/snowflake       4.778s

$ GOMAXPROCS=8 go test -v -run NULL -bench .
BenchmarkSavakiSnowflakeGen-8       3000000      523 ns/op
BenchmarkSnowflakeGen-8             5000000      437 ns/op
BenchmarkTwitterSnowflakeGen-8      2000000      659 ns/op
PASS
ok      _/go/snowflake       6.721s

$ GOMAXPROCS=4 go test -v -run NULL  -bench .
BenchmarkSavakiSnowflakeGen-4       3000000      449 ns/op
BenchmarkSnowflakeGen-4             5000000      356 ns/op
BenchmarkTwitterSnowflakeGen-4      3000000      497 ns/op
PASS
ok      _/go/snowflake       6.001s
```

Ok, so now that we've run the benchmark everywhere we see that the problem appears to be unique to Docker for Mac. Which isn't a production environment so no need to get bent out of shape about that. Right? Well, you'll note from the position of your scroll bar that perhaps I decided to get bent out of shape about it anyways, just out of morbid curiosity.

## Profiling

The next step was to take a profiler to both implementations on both my development machine and in Docker for Mac. It would have been neat to see the profiling on MacOS as well by way of comparison but apparently go's profiling is totally broken outside of Linux.

First I ran the benchmark again on my Linux laptop, this time outputting a CPU profile. I'll then run this output through go's `pprof` tool. I first looked at the top 20 calls, sorted by cumulative time. The cumulative time in `pprof` refers to the total amount of time spent in the call including all its callees. As we'd hope and expect for a tight-looping benchmark, we spend the vast majority of our time in the code we're benchmarking. Perhaps unsurprisingly we spend about 70% of our time locking and unlocking a mutex.

```
$ go test -v -run NULL -bench BenchmarkSnowflake -cpuprofile cpu-snowflake.prof
BenchmarkSnowflakeGen-8             5000000      244 ns/op
PASS
ok      snowflake 1.471s

$ go tool pprof cpu-snowflake.prof
Entering interactive mode (type "help" for commands)
(pprof) top20 -cum
2.77s of 3.65s total (75.89%)
Dropped 8 nodes (cum <= 0.02s)
Showing top 20 nodes out of 46 (cum >= 0.21s)
      flat  flat%   sum%        cum   cum%
     0.33s  9.04%  9.04%      2.88s 78.90%  snowflake.(*Server).GetSnowflake
         0     0%  9.04%      2.87s 78.63%  runtime.goexit
     0.01s  0.27%  9.32%      2.86s 78.36%  snowflake.BenchmarkSnowflakeGen.func1
         0     0%  9.32%      2.86s 78.36%  testing.(*B).RunParallel.func1
     0.51s 13.97% 23.29%      1.88s 51.51%  sync.(*Mutex).Lock
     0.83s 22.74% 46.03%      0.83s 22.74%  sync/atomic.CompareAndSwapUint32
     0.01s  0.27% 46.30%      0.74s 20.27%  runtime.mcall
         0     0% 46.30%      0.73s 20.00%  runtime.park_m
         0     0% 46.30%      0.68s 18.63%  runtime.schedule
     0.10s  2.74% 49.04%      0.67s 18.36%  sync.(*Mutex).Unlock
     0.02s  0.55% 49.59%      0.65s 17.81%  runtime.findrunnable
     0.06s  1.64% 51.23%      0.38s 10.41%  runtime.lock
     0.01s  0.27% 51.51%      0.34s  9.32%  sync.runtime_Semacquire
     0.03s  0.82% 52.33%      0.33s  9.04%  runtime.semacquire
     0.31s  8.49% 60.82%      0.31s  8.49%  runtime/internal/atomic.Xchg
     0.27s  7.40% 68.22%      0.27s  7.40%  sync/atomic.AddUint32
     0.02s  0.55% 68.77%      0.25s  6.85%  runtime.semrelease
         0     0% 68.77%      0.25s  6.85%  sync.runtime_Semrelease
     0.05s  1.37% 70.14%      0.21s  5.75%  runtime.unlock
     0.21s  5.75% 75.89%      0.21s  5.75%  runtime/internal/atomic.Xadd
```


But where is that mutex? For clarity we can list the `GetSnowflake` function and see the call times compared to each section of code.

```
(pprof)  list GetSnowflake
Total: 3.65s
ROUTINE ======================== snowflake.(*Server).GetSnowflake in snowflake.go
     330ms      2.94s (flat, cum) 80.55% of Total
         .          .     87:
         .          .     88:// GetSnowflake ...
         .          .     89:func (s *Server) GetSnowflake() Snowflake {
         .          .     90:   // we don't exit scope except in the happy case where we
         .          .     91:   // return, so we can't 'defer s.lock.Unlock()' here
      30ms      1.91s     92:   s.lock.Lock()
     140ms      140ms     93:   sequence := s.id
         .          .     94:   if sequence > maxSequence {
         .          .     95:           s.lock.Unlock()
         .       60ms     96:           return s.GetSnowflake()
         .          .     97:   }
         .          .     98:   s.id++
      10ms       10ms     99:   timestamp := s.lastTimestamp
      10ms      680ms    100:   s.lock.Unlock()
         .          .    101:   return Snowflake(
     140ms      140ms    102:           (timestamp << (workerIDBits + sequenceBits)) |
         .          .    103:                   (uint64(s.workerID) << sequenceBits) |
         .          .    104:                   (uint64(sequence)))
         .          .    105:}
```

This code perhaps makes little sense without the rest of the server code. Here's that.

``` go
// Server ...
type Server struct {
        workerID      workerID
        ticker        *time.Ticker
        lock          sync.Mutex
        lastTimestamp uint64
        id            uint16
}

// NewServer ...
func NewServer() *Server {
        ticker := time.NewTicker(time.Millisecond)
        server := &Server{
                ticker:        ticker,
                workerID:      0,
                lock:          sync.Mutex{},
                id:            0,
                lastTimestamp: uint64(time.Now().UnixNano()/nano - idEpoch),
        }
        go func() {
                for tick := range ticker.C {
                        server.reset(tick)
                }
        }()
        return server
}

func (s *Server) reset(tick time.Time) {
        s.lock.Lock()
        defer s.lock.Unlock()
        s.lastTimestamp = uint64(tick.UnixNano()/nano - idEpoch)
        s.id = 0
}
```

Let's now do the same for the Twitter port. A bit less time being spent in the mutex and a bit more spent switching in the runtime. Nothing unexpected. I've provided the code listing below again.

```
$ go tool pprof cpu-snowflake-twitter.prof
Entering interactive mode (type "help" for commands)
(pprof) top20 -cum
2.50s of 4s total (62.50%)
Dropped 16 nodes (cum <= 0.02s)
Showing top 20 nodes out of 61 (cum >= 0.25s)
      flat  flat%   sum%        cum   cum%
     0.03s  0.75%  0.75%      2.94s 73.50%  snowflake.BenchmarkTwitterSnowflakeGen.func1
         0     0%  0.75%      2.94s 73.50%  runtime.goexit
         0     0%  0.75%      2.94s 73.50%  testing.(*B).RunParallel.func1
     0.24s  6.00%  6.75%      2.88s 72.00%  snowflake.(*TwitterServer).GetSnowflake
     0.29s  7.25% 14.00%      1.46s 36.50%  sync.(*Mutex).Lock
     0.03s  0.75% 14.75%      0.87s 21.75%  runtime.mcall
     0.03s  0.75% 15.50%      0.84s 21.00%  runtime.park_m
     0.02s   0.5% 16.00%      0.77s 19.25%  runtime.schedule
     0.04s  1.00% 17.00%      0.75s 18.75%  sync.(*Mutex).Unlock
     0.18s  4.50% 21.50%      0.73s 18.25%  runtime.lock
     0.12s  3.00% 24.50%      0.70s 17.50%  runtime.findrunnable
     0.08s  2.00% 26.50%      0.57s 14.25%  runtime.semacquire
         0     0% 26.50%      0.57s 14.25%  sync.runtime_Semacquire
     0.01s  0.25% 26.75%      0.54s 13.50%  sync.runtime_Semrelease
     0.06s  1.50% 28.25%      0.53s 13.25%  runtime.semrelease
     0.11s  2.75% 31.00%      0.44s 11.00%  runtime.systemstack
     0.42s 10.50% 41.50%      0.42s 10.50%  sync/atomic.CompareAndSwapUint32
     0.33s  8.25% 49.75%      0.33s  8.25%  runtime.procyield
     0.26s  6.50% 56.25%      0.26s  6.50%  runtime/internal/atomic.Xchg
     0.25s  6.25% 62.50%      0.25s  6.25%  runtime/internal/atomic.Cas

(pprof) list GetSnowflake
Total: 4s
ROUTINE ======================== snowflake.(*TwitterServer).GetSnowflake in snowflake/snowflake_twitter.go
     240ms      2.88s (flat, cum) 72.00% of Total
         .          .     41:
         .          .     42:// GetSnowflake ...
         .          .     43:func (s *TwitterServer) GetSnowflake() Snowflake {
         .          .     44:
      20ms       40ms     45:   ts := uint64(time.Now().UnixNano()/nano - idTwitterEpoch)
     120ms      1.58s     46:   s.lock.Lock()
      20ms      220ms     47:   defer s.lock.Unlock()
         .          .     48:   // TODO: need to block on clock running backwards
         .          .     49:
      40ms       40ms     50:   if s.lastTimestamp == ts {
      10ms       10ms     51:           s.id = (s.id + 1) & maxSequence
         .          .     52:           if s.id == 0 {
         .          .     53:                   for ts <= s.lastTimestamp {
         .          .     54:                           time.Sleep(10 * time.Microsecond)
         .          .     55:                           ts = uint64(time.Now().UnixNano()/nano - idTwitterEpoch)
         .          .     56:                   }
         .          .     57:           }
         .          .     58:   } else {
         .          .     59:           s.id = 0
         .          .     60:   }
         .          .     61:   s.lastTimestamp = ts
         .          .     62:   id := Snowflake(
      20ms       20ms     63:           (s.lastTimestamp << (workerIDBits + sequenceBits)) |
         .          .     64:                   (uint64(s.workerID) << sequenceBits) |
      10ms       10ms     65:                   (uint64(s.id)))
         .      960ms     66:   return id
         .          .     67:
         .          .     68:}
```

I've replicated the results above under hardware virtualized Linux (the AWS `c4.xlarge`) and bare metal containers (the Triton `g4-highcpu-4G`) without learning anything new. From this point in the interest of space I'm going to abandon analysis of the `SavakiServer` and focus on just my version and the Twitter version. You'll have to trust me when I say that it matches the Twitter version pretty closely except in some small details. But now I'll take the cpu profiling to the Docker for Mac environment.

First my Snowflake server, which is mostly unchanged as we'd expect from the original benchmark values.

```
$ go tool pprof cpu-snowflake.prof
Entering interactive mode (type "help" for commands)
(pprof) top20 -cum
1.51s of 1.85s total (81.62%)
Showing top 20 nodes out of 58 (cum >= 0.04s)
      flat  flat%   sum%        cum   cum%
     0.19s 10.27% 10.27%      1.64s 88.65%  _/go/snowflake.(*Server).GetSnowflake
         0     0% 10.27%      1.39s 75.14%  runtime.goexit
     0.03s  1.62% 11.89%      1.37s 74.05%  _/go/snowflake.BenchmarkSnowflakeGen.func1
         0     0% 11.89%      1.37s 74.05%  testing.(*B).RunParallel.func1
     0.32s 17.30% 29.19%      1.08s 58.38%  sync.(*Mutex).Lock
     0.49s 26.49% 55.68%      0.49s 26.49%  sync/atomic.CompareAndSwapUint32
     0.12s  6.49% 62.16%      0.37s 20.00%  sync.(*Mutex).Unlock
     0.19s 10.27% 72.43%      0.19s 10.27%  sync/atomic.AddUint32
     0.16s  8.65% 81.08%      0.16s  8.65%  runtime.procyield
     0.01s  0.54% 81.62%      0.16s  8.65%  sync.runtime_doSpin
         0     0% 81.62%      0.08s  4.32%  runtime.mcall
         0     0% 81.62%      0.08s  4.32%  runtime.park_m
         0     0% 81.62%      0.08s  4.32%  runtime.semacquire
         0     0% 81.62%      0.08s  4.32%  sync.runtime_Semacquire
         0     0% 81.62%      0.06s  3.24%  runtime.findrunnable
         0     0% 81.62%      0.06s  3.24%  runtime.schedule
         0     0% 81.62%      0.05s  2.70%  runtime.copystack
         0     0% 81.62%      0.05s  2.70%  runtime.morestack
         0     0% 81.62%      0.05s  2.70%  runtime.newstack
         0     0% 81.62%      0.04s  2.16%  runtime.cansemacquire

```

And now for the Twitter server.

```
$ go tool pprof cpu-snowflake-twitter.prof
Entering interactive mode (type "help" for commands)
(pprof) top20 -cum
3.12s of 3.22s total (96.89%)
Dropped 9 nodes (cum <= 0.02s)
Showing top 20 nodes out of 24 (cum >= 0.02s)
      flat  flat%   sum%        cum   cum%
         0     0%     0%      2.84s 88.20%  runtime._System
     2.83s 87.89% 87.89%      2.83s 87.89%  runtime._ExternalCode
     0.01s  0.31% 88.20%      0.35s 10.87%  _/go/snowflake.BenchmarkTwitterSnowflakeGen.func1
         0     0% 88.20%      0.35s 10.87%  runtime.goexit
         0     0% 88.20%      0.35s 10.87%  testing.(*B).RunParallel.func1
     0.02s  0.62% 88.82%      0.34s 10.56%  _/go/snowflake.(*TwitterServer).GetSnowflake
     0.07s  2.17% 90.99%      0.27s  8.39%  sync.(*Mutex).Lock
     0.10s  3.11% 94.10%      0.10s  3.11%  runtime.procyield
         0     0% 94.10%      0.10s  3.11%  sync.runtime_doSpin
     0.02s  0.62% 94.72%      0.04s  1.24%  runtime.deferreturn
     0.04s  1.24% 95.96%      0.04s  1.24%  sync/atomic.CompareAndSwapUint32
         0     0% 95.96%      0.03s  0.93%  runtime.mcall
         0     0% 95.96%      0.03s  0.93%  runtime.park_m
     0.01s  0.31% 96.27%      0.03s  0.93%  runtime.schedule
         0     0% 96.27%      0.03s  0.93%  runtime.semacquire
         0     0% 96.27%      0.03s  0.93%  sync.runtime_Semacquire
     0.01s  0.31% 96.58%      0.03s  0.93%  sync.runtime_canSpin
         0     0% 96.58%      0.02s  0.62%  runtime.findrunnable
     0.01s  0.31% 96.89%      0.02s  0.62%  runtime.runqempty
         0     0% 96.89%      0.02s  0.62%  runtime.runqgrab
```

Finally a clue! But what is `runtime._System` and `runtime._ExternalCode`? Well the golang docs don't give us much help here but this post by [Dmitry Vyukov on a blog for Intel](https://software.intel.com/en-us/blogs/2014/05/10/debugging-performance-issues-in-go-programs) has this to say:

> System means time spent in goroutine scheduler, stack management code and other auxiliary runtime code. ExternalCode means time spent in native dynamic libraries.

We're pushing into areas where I don't have deep expertise, but given that I've got a statically linked binary here, this sounds like we're talking about system calls. So I put `strace` on the benchmark runs and counted up the calls. I've elided all the noise here and taken the top handful of syscalls from each one.

My version, on native Linux.

```
$ strace -cf go test -run NULL -bench BenchmarkSnowflake

% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 74.57    1.592028          42     37914      8385 futex
 25.40    0.542223           9     62410           select
  0.02    0.000493           0     19141           sched_yield
  0.00    0.000053           5        10           wait4
  0.00    0.000015           0      1766           read
  0.00    0.000015           0        80        55 unlinkat
  ...
------ ----------- ----------- --------- --------- ----------------
100.00    2.134836                128426      8858 total
```

The Twitter version, on native Linux.

```
$  strace -cf go test -run NULL -bench BenchmarkTwitter
...
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 78.46    1.642500          35     46413      5088 futex
 20.76    0.434634           9     51023           select
  0.77    0.016064        1606        10           wait4
  0.01    0.000137           0     20525           sched_yield
  0.00    0.000022           0       833           close
  0.00    0.000017           0      1528        13 lstat
  0.00    0.000012           0       604           rt_sigaction
  0.00    0.000012           2         6           rt_sigreturn
  ...
------ ----------- ----------- --------- --------- ----------------
100.00    2.093406                126847      5561 total
```

My version, on Docker for Mac.

```
$ strace -cf go test -run NULL -bench BenchmarkSnowflake
...
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 76.19    4.201271         550      7641      1664 futex
 19.11    1.053696         119      8856           select
  3.27    0.180172         101      1776           read
  0.52    0.028537           2     16723           clock_gettime
  0.37    0.020437        2044        10           wait4
  0.36    0.020000        3333         6           waitid
  0.17    0.009448          12       785           sched_yield
  0.01    0.000342          57         6           rt_sigreturn
  ...
------ ----------- ----------- --------- --------- ----------------
100.00    5.514387                 42723      2108 total
```


The Twitter version, on Docker for Mac.

```
$ strace -cf go test -run NULL -bench BenchmarkTwitter
...
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 67.48    2.568430        1468      1750       258 futex
 21.61    0.822401         707      1164           select
 10.63    0.404649           3    118067           clock_gettime
  0.27    0.010230        1023        10           wait4
  0.00    0.000171           0      1515         9 lstat
  0.00    0.000136           0       749           openat
  ...
------ ----------- ----------- --------- --------- ----------------
100.00    3.806243                129994       702 total
```

We expect to see lots of `futex` calls ("fast user space mutex") because the golang runtime uses it a bunch under the hood for scheduling and our `mutex.Lock`/`mutex.ULock` calls. We can see that my version spends more of its total time in system calls than the Twitter version in both cases. We can also see the large amount of overhead that `strace` is incurring; these benchmarks only take ~1sec without it! If only Linux had dtrace! So we should take the timing with a heaping tablespoon of salt. But now wait a second, where is `clock_gettime` in the native Linux version?

## You know what time it is

It's time to get [vDSO](http://man7.org/linux/man-pages/man7/vdso.7.html)! (That probably sounded cooler in my head.) Virtual dynamic shared objects are a pretty nifty performance optimization. Normally when a userspace program makes a system call, there's a fairly expensive bit of shuffling around of memory where some registers are set, kernel space memory gets swapped in, more registers are set, and kernel memory gets swapped back out. But there are lots of programs that want to make lots and lots of system calls like checking what time it is. So vDSOs map a read-only page of kernel memory associated with certain system call functions into userspace. This way userspace applications can use those functions without going through the context switch of a syscall. I'm not super-familiar with the history of this feature but there's an older mechanism called vsyscall that vDSO has replaced, and it looks like the feature started in ever-practical-but-often-a-bit-gross Linux and got ported or emulated in the BSDs and Illumos family later down the road. (I'm relying on [Cunningham's Law](https://meta.wikimedia.org/wiki/Cunningham's_Law) to correct me on this point.)

We can use `go tool pprof` to disassemble our function but we don't get real assembler in this case. Instead we get a kind of [pseudo-assembly](http://dtrace.org/blogs/wesolows/2014/12/29/golang-is-trash/) that's specific to the golang runtime.

```
# irrelevant sections elided
(pprof) disasm GetSnowflake
Total: 4.15s
ROUTINE ======================== snowflake.(*TwitterServer).GetSnowflake
     230ms      3.17s (flat, cum) 76.39% of Total
         .       20ms     46ee6a: CALL time.Now(SB)
         .          .     46ef47: CALL time.Now(SB)
```

But we can take the test binary built by the CPU profiler and dump it to see how we're calling system calls there to see if vDSO might be a factor here.

```
# machine code bytes elided for space
$  objdump -d snowflake.test| grep gettime
45887e:   mov    0x16935b(%rip),%rax  # 5c1be0 <runtime.__vdso_clock_gettime_sym>
4588b5:   mov    0x139864(%rip),%rax  # 592120 <runtime.__vdso_gettimeofday_sym>
4588ee:   mov    0x1692eb(%rip),%rax  # 5c1be0 <runtime.__vdso_clock_gettime_sym>
45892e:   mov    0x1397eb(%rip),%rax  # 592120 <runtime.__vdso_gettimeofday_sym>

```

A bit of googling leads us to [this bit of golang assembly](https://golang.org/src/runtime/sys_linux_amd64.s#L138) in the golang runtime code. Note that this is *not* x86 assembly, and even it were I'm running into the limits of my useful knowledge. Presumably it's calling into the vDSO page that's being exposed to userspace.

```
// func now() (sec int64, nsec int32)
TEXT time.now(SB),NOSPLIT,$16
    // Be careful. We're calling a function with gcc calling convention here.
    // We're guaranteed 128 bytes on entry, and we've taken 16, and the
    // call uses another 8.
    // That leaves 104 for the gettime code to use. Hope that's enough!
    MOVQ  runtime.__vdso_clock_gettime_sym(SB), AX
    CMPQ  AX, $0
    JEQ   fallback
    MOVL  $0, DI // CLOCK_REALTIME
    LEAQ  0(SP), SI
    CALL  AX
    MOVQ  0(SP), AX// sec
    MOVQ  8(SP), DX// nsec
    MOVQ  AX, sec+0(FP)
    MOVL  DX, nsec+8(FP)
    RET
fallback:
    LEAQ  0(SP), DI
    MOVQ  $0, SI
    MOVQ  runtime.__vdso_gettimeofday_sym(SB), AX
    CALL  AX
    MOVQ  0(SP), AX// sec
    MOVL  8(SP), DX// usec
    IMUL  Q$1000, DX
    MOVQ  AX, sec+0(FP)
    MOVL  DX, nsec+8(FP)
    RET
```

Funny story with the "hope that's enough" comment is that my colleagues at Joyent recently ran into it while investigating a segfaulting golang program on LX. [Spoiler alert: it was not enough.](https://smartos.org/bugview/OS-5637)


## So what's happening here?

So why is there a difference between Docker for Mac and all the other environments? If it's something specific to the Mac why don't we see it without Docker? There's one more wrench in this story, and that's Hypervisor Framework and [xhyve](https://github.com/mist64/xhyve). Unlike all the other cases, in Docker for Mac we're using xhyve, which runs entirely in userspace. Presumably somewhere in the userspace emulation we're losing a lot of our performance to virtualization.

This is unfortunately where my enthusiasm for following this to the end starts to flag. That's probably a disappointing finish, but in fairness I totally warned you! If you want to tackle this further, xhyve is open source and developed on GitHub. The real conclusion is to be careful that you're benchmarking what you really think you are. Even in cases where you've got "pure" CPU benchmarks you can be caught by surprise!
