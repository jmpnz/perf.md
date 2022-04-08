# perf.md

This is a collection of notes on software performance, optimizations and mistakes
I learned over the years.

While there are several great [ressources](#ressources) you can read, this one tries
to be pragmatic and focus on a few rules of thumbs and heuristics.

If there's one thing you take away from this document *Don't guess, measure. Then measure it again.*.

## Reach for low hanging fruits

One of the hallmarks of software optimization is trying to rewrite core parts of an application
either by redesigning it or sometimes going for the extreme of [rewriting it in rust](https://discord.com/blog/why-discord-is-switching-from-go-to-rust).

Whatever you pick, as long as it works it's a good option. But sometimes, you want to start easy
you've just got a ticket "Application slow make fast pls" and crunching a couple of weeks on a
redesign and reimplementation is probably not an option.

One thing to do is to go through the basics, look for hotpaths, refactor hotpaths, look for other
hotpaths.

In the simplest case the goal here is to shave a few seconds, you can do it by adjusting preallocations
avoid the [accidental O(n^2)](https://accidentallyquadratic.tumblr.com/) but you should avoid doing any refactoring or redesigns.

# ressources

- [Brendan Greg - Home Page and Books](https://www.brendangregg.com/)
- [Daniel Lemire - Home Page and Papers](https://lemire.me/blog/)
- [Damian Gryski - Go Perf Book](https://github.com/dgryski/go-perfbook)
- [Sergey Slotin - Algorithms for Modern Hardware](https://en.algorithmica.org/hpc/)
