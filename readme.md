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
    + [WaitGroup](#waitgroup)
    + [Mutex and RWMutex](#mutex-and-rwmutex)
    + [Cond](#cond)
    + [Once](#once)
    + [Pool](#pool)

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
#### WaitGroup
WaitGroup is a great way to wait for a set of concurrent operations to complete when you either don't care about the results or you have other means of collecting the results. Otherwise use channels and a select statement.

[fig-wait-group.go](gos-concurrency-building-blocks%2Fthe-sync-package%2Fwaitgroup%2Ffig-wait-group.go)

It's customary to keep the calls to `WaitGroup.Add` as close as possible to the goroutines they're tracking.

But you can bulk-add goroutines, if you're creating goroutines in a loop and you know the number.

[fig-bulk-add.go](gos-concurrency-building-blocks%2Fthe-sync-package%2Fwaitgroup%2Ffig-bulk-add.go)

#### Mutex and RWMutex
A Mutex (mutual exclusion) provides a concurrent-safe way to express exclusive access to shared resources.

Here's an example that starts 5 goroutines to increment a variable and 5 goroutines to decrement it.

[fig-mutex.go](gos-concurrency-building-blocks%2Fthe-sync-package%2Fmutex-%26-rwmutex%2Ffig-mutex.go)

The `count` variable is locked with `lock.Lock()` and unlocked by `defer lock.Unlock()`, an idiomatic way of making sure unlocks are always called even if the program exits prematurely.

RWMutex lets you request a lock for reading, which is granted unless the lock is being held for writing. Gives you a bit more control.

[fig-rwlock.go](gos-concurrency-building-blocks%2Fthe-sync-package%2Fmutex-%26-rwmutex%2Ffig-rwlock.go)

#### Cond
A way of making a goroutine sleep until a condition is met.

[fig-cond-based-queue.go](gos-concurrency-building-blocks%2Fthe-sync-package%2Fcond%2Ffig-cond-based-queue.go)

[fig-cond-broadcast.go](gos-concurrency-building-blocks%2Fthe-sync-package%2Fcond%2Ffig-cond-broadcast.go)

#### Once
You can pass a function to `sync.Once`'s `Do()`. This ensures that function is only ever run once, even if called multiple times by multiple goroutines.

[fig-sync-once.go](gos-concurrency-building-blocks%2Fthe-sync-package%2Fonce%2Ffig-sync-once.go)

In this example [fig-sync-once-diff-funcs.go](gos-concurrency-building-blocks%2Fthe-sync-package%2Fonce%2Ffig-sync-once-diff-funcs.go), `once.Do` is called twice, with a different function passed in for each one.

But only the first one is called because `sync.Once` only counts the number of times `Do` is called, not how many times unique functions passed into `Do` are called.

This is an example of a deadlock caused by `sync.Once`: [fig-sync-once-do-deadlock.go](gos-concurrency-building-blocks%2Fthe-sync-package%2Fonce%2Ffig-sync-once-do-deadlock.go)

#### Pool
A way of creating a fixed number of things, usually expensive things like database connections. Multiple goroutines can access this pool of things but only a limited number of things will be created.

This code [fig-sync-pool-basic.go](gos-concurrency-building-blocks%2Fthe-sync-package%2Fpool%2Ffig-sync-pool-basic.go) shows that the first time we call `Get` on a pool, a new object is created. It isn't assigned to a variable so the returned `struct{}{}` leaves the pool and is lost.

Then `Get()` is called again. This makes the number of items in the pool go from 0 to 1.

Then it is assigned to a variable, meaning the pool item is removed from the pool, and the items in the pool goes from 1 to 0.

`Put` puts an object retrieved from the pool back in the pool, changing the number of items in the pool from 0 back to 1.

When `Get` is called again we reuse the instance previously allocated and immediately put it back in the pool.

This code [fig-sync-pool.go](gos-concurrency-building-blocks%2Fthe-sync-package%2Fpool%2Ffig-sync-pool.go) restricts the number of items created in the pool. 

It creates 1,048,576 goroutines, each one calling `Get` on a pool, swiftly followed by `Put`. 

Because a lot of goroutines are getting but then replacing items from the pool, there are previously allocated items in the pool for the next goroutine to use, as opposed to having a create a new one.

When running the above code multiple times you get different numbers of created items, typically 10-16.

If we weren't reusing objects then we could create 1,048,576 instances, one for each goroutine!