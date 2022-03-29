# perf.md

This is a collection of notes on software performance, optimizations and mistakes
I learned over the years.

While there are several great [ressources](#ressources) you can read, this one tries
to be pragmatic and focus on a few rules of thumbs and heuristics.

If there's one thing you take away from this document *Don't guess, measure. Then measure it again.*.

## Reach for low hanging fruits

One of the hallmarks of software optimization is trying to rewrite core parts of an application
either by redesigning it or sometimes going for the extreme of [rewriting it in rust]().

Whatever you pick, as long as it works it's a good option. But sometimes, you want to start easy
you've just got a ticket "Application slow make fast pls" and crunching a couple of weeks on a
redesign and reimplementation is probably not an option.

One thing to do is to go through the basics, look for hotpaths, refactor hotpaths, look for other
hotpaths.

In the simplest case the goal here is to shave a few seconds, you can do it by adjusting preallocations
avoid the [accidental O(n^2)]() but you should avoid doing any refactoring or redesigns.

But the most troubling optimizations to untangle is the ones where you're [locking]() especially
in batch jobs.

One time while looking at some code, I noticed an interesting case of a very long lock.
Upon further investigation it turned out the batch job was concurrent (supposedly) but because
of the lock, the trace profile showed the goroutine were running sequentially.

```go

data := sourceReader.Read()

summary.mu.Lock()
defer summary.mu.Unlock()

for _, row := range data {
  for _, item := range row {
    if item.Type == Potatos {
      summary.add_type(Potatos)
    } else {
      summary.add_type(NotPotatos)
    }
  }
}

```

Looks "not clever" by Go standards, except if you have 10 million rows and 20 columns and maybe
you're doing more than sorting through potatos.

One might notice the amenability of this operation to parallelism,(using concurrency of course 
so it's like random parallelism with extra steps and the you might pull a [mapreduce]() on it.

```go

data := sourceReader.Read()

processRow := func(row *Row) rowSummary {
  for _,item := range row {
    if item.Type == Potatos {
      rowSummary.add_type(Potatos)
    } else {
      rowSummary.add_type(NotPotatos)
    }
  }
}

// lock the append
summaries := []rowSummary

for _,row := range data {
  summary := go processRow(row[:n])
  summaries = append(summaries, summary)
  }
}

```

Of course in a real world setting you need context handling a better way to do the "reduce"
probably with a [lock-free queue]() and a few sprinkles of logging, error handling et al.

In real life, this change reduced one of my batch jobs duration from 7 minutes on a 25GB dataset
to approximately 1m19sec.

Of course the numbers you need to collect should be bias free (different profiles have different times)
for reasons such as heat, background jobs, I/O utilization, I/O speed...

Avoiding loading the data in memory in one Go and doing async Reads with a buffer might even improve
performance.

For a batch job, where there's no data dependency a MapReduce should be your first pick, it might not
appear to be performant at first sight, but MapReduce is scalable and scalability is where performance
matter.

# ressources

- [Brendan Greg - Home Page and Books](https://www.brendangregg.com/)
- [Daniel Lemire - Home Page and Papers](https://lemire.me/blog/)
- [Damian Gryski - Go Perf Book](https://github.com/dgryski/go-perfbook)
- [Sergey Slotin - Algorithms for Modern Hardware](https://en.algorithmica.org/hpc/)
