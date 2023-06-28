# Notes on _Concurrency in Go_ by Katherine Cox-Buday (O'Reilly Books)

Forked the source code for this book.

These are explanatory notes linking to code examples.

## Contents
<!-- `make toc` to generate https://github.com/jonschlinkert/markdown-toc#cli -->

<!-- toc -->

- [Go's Concurrency Building Blocks](#gos-concurrency-building-blocks)
  * [Goroutines](#goroutines)
    + [Fork-Join Model](#fork-join-model)
    + [Goroutine Closures](#goroutine-closures)
    + [Goroutine Closures in Loops](#goroutine-closures-in-loops)
    + [Goroutine Size](#goroutine-size)
    + [Goroutine Context Switching](#goroutine-context-switching)
  * [The sync Package](#the-sync-package)

<!-- tocstop -->

## Go's Concurrency Building Blocks
### Goroutines
#### Fork-Join Model
Every Go program has at least one goroutine: the _main goroutine_.

When a new goroutine is launched it _forks_ or splits a child branch of execution from the parent.

_Join_ is when concurrent branches of execution join back together.

[fig-join-point.go](gos-concurrency-building-blocks%2Fgoroutines%2Ffig-join-point.go)

#### Goroutine Closures
Goroutines close over variables to operate on the original reference.

For example, in [fig-goroutine-closure.go](gos-concurrency-building-blocks%2Fgoroutines%2Ffig-goroutine-closure.go), the `salutation` variable is set outside a goroutine. Inside the goroutine, `salutation` is reset to something different. After the goroutine, the updated value is seen. So the goroutine _closed over_ the variable and captured it, and was able to update it.

#### Goroutine Closures in Loops
In this code [fig-goroutine-closure-loop.go](gos-concurrency-building-blocks%2Fgoroutines%2Ffig-goroutine-closure-loop.go), we loop over a slice of strings and run a goroutine on each value.

The code prints out `good day` 3 times, instead of each of the slice's values. Why is this?

The goroutine closes over the iteration variable `salutation`.

Usually, the loop finishes before the goroutines inside the lop have a chance to start. So what is the value of `salutation`? It's the last value in the slice because it looped over everything and the last value was `good day`.

Then the goroutines start and they hold onto the closed over `salutation` variable, which are all `good day`.

To fix this, pass the value of `salutation` to each goroutine. This works because we are passing a copy of `salutation` into the closure sp that by the time the goroutine is running, it will be operating on the data from its iteration of the loop.

Here's the fixed code: [fig-goroutine-closure-loop-correct.go](gos-concurrency-building-blocks%2Fgoroutines%2Ffig-goroutine-closure-loop-correct.go)

#### Goroutine Size
Goroutines are lightweight, starting at around 3Kb.

[fig-goroutine-size.go](gos-concurrency-building-blocks%2Fgoroutines%2Ffig-goroutine-size.go)

#### Goroutine Context Switching
Go can context switch between goroutines very quickly. Typical times are 225 ns per context switch.

Here's some code that times it: [fig-ctx-switch_test.go](gos-concurrency-building-blocks%2Fgoroutines%2Ffig-ctx-switch_test.go)

### The sync Package