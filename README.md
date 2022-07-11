# perf.md

This is a collection of notes on software performance, optimizations and mistakes
I learned over the years.

While there are several great [resources](#resources) you can read, this one tries
to be pragmatic with a particular focus towards software design. 

If there's one thing you take away from this document *Don't guess, measure. Then measure it again.*.

## Beyond all those numbers you were told you needed to know.

Almost everyone is told there's a list of numbers every programmer should know, the numbers are
timings of various computer actions such as how long it takes to fetch from the various caches
or how much it takes for a single packet to make a cross-ocean round trip.

Here's a distilled version from 2012 courtesy of [jboner](https://gist.github.com/jboner/2841832)

```sh

Latency Comparison Numbers (~2012)
----------------------------------
L1 cache reference                           0.5 ns
Branch mispredict                            5   ns
L2 cache reference                           7   ns                      14x L1 cache
Mutex lock/unlock                           25   ns
Main memory reference                      100   ns                      20x L2 cache, 200x L1 cache
Compress 1K bytes with Zippy             3,000   ns        3 us
Send 1K bytes over 1 Gbps network       10,000   ns       10 us
Read 4K randomly from SSD*             150,000   ns      150 us          ~1GB/sec SSD
Read 1 MB sequentially from memory     250,000   ns      250 us
Round trip within same datacenter      500,000   ns      500 us
Read 1 MB sequentially from SSD*     1,000,000   ns    1,000 us    1 ms  ~1GB/sec SSD, 4X memory
Disk seek                           10,000,000   ns   10,000 us   10 ms  20x datacenter roundtrip
Read 1 MB sequentially from disk    20,000,000   ns   20,000 us   20 ms  80x memory, 20X SSD
Send packet CA->Netherlands->CA    150,000,000   ns  150,000 us  150 ms

Notes
-----
1 ns = 10^-9 seconds
1 us = 10^-6 seconds = 1,000 ns
1 ms = 10^-3 seconds = 1,000 us = 1,000,000 ns

```

Since then the numbers improved quite dramatically, on a fresh 2021 box it takes 4,000 ns to make
a 1 MB sequentially read from memory, here's a [great visualization](https://colin-scott.github.io/personal_website/research/interactive_latency.html)
of how the numbers changed in the last 20 years.

The numbers are important as a reference but they don't offer much insight into how to leverage them
for engineering decisions.

Consider the time it takes for acquiring a mutex lock, 25 ns is quite small unless you are in
a constrained or critical environments there are probably a few locks here and there in different
hotpaths of your application.

But then you add up how many records you're processing, the complexity of your computation and the
difference between a lock and no-lock becomes in the seconds.

## Reach for low hanging fruits

One of the hallmarks of software optimization is trying to rewrite core parts of an application
either by redesigning it or sometimes going for the extreme of [rewriting it in rust](https://discord.com/blog/why-discord-is-switching-from-go-to-rust).

In the simplest case the goal here is to shave a few seconds, you can do it by adjusting preallocations
avoid the [accidental O(n^2)](https://accidentallyquadratic.tumblr.com/), and more importantly stay
sane.

### Granular Locks in Go ( `defer` considered harmful)

This is quite popular in Golang applications, take a look at this snippet from [Kubernetes's
code](https://github.com/kubernetes/kubernetes/blob/6609899398d35d22a7482f687ed05fb19093b762/staging/src/k8s.io/client-go/plugin/pkg/client/auth/exec/metrics.go#L69)


```go

ressources.mu.Lock()
defer ressources.mu.Unlock()

```

This pattern is pretty pervasive in any application where a ressource needs to be shared and we
seldom think of the cost of the pattern.

It is highly recommended to use `defer`, you can't think of it as a post-it note that tells the
runtime to run some code before returning.

if you're implementing a cache where reads > writes then using `RWMutex` is faster, even so
when you're locking the code for the entire computation.

In the example above the ressource will be locked for the entire computation, in this case
the cost is small but if your iteration takes time to run and the only *critical ressource*
is the cache, then using more granular locks **will always be faster**.

```go

for _,item := range items {
  if item.Category() == "Retail" {
    // do something
    cache.mu.Lock()
    cache[item.Name()]++
    cache.mu.Unlock()
  }

```

If you have two goroutines filtering through two lists of items in a shared cache, the first
executing goroutine will hold a lock for the entire iteration when the lock is only needed
for increment operation.

Look at your code, are your goroutines suffering from lock contention due to a simplistic quote ?


# ressources

- [Brendan Greg - Home Page and Books](https://www.brendangregg.com/)
- [Daniel Lemire - Home Page and Papers](https://lemire.me/blog/)
- [Damian Gryski - Go Perf Book](https://github.com/dgryski/go-perfbook)
- [Sergey Slotin - Algorithms for Modern Hardware](https://en.algorithmica.org/hpc/)
