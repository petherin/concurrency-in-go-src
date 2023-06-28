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
  * [Channels](#channels)
    + [Closed Channels](#closed-channels)
    + [Ranging Over a Channel](#ranging-over-a-channel)
    + [Unblocking Multiple Goroutines at Once](#unblocking-multiple-goroutines-at-once)
    + [Buffered and Unbuffered Channels](#buffered-and-unbuffered-channels)
    + [Nil Channels](#nil-channels)
    + [Channel Owners Must Create, Write and Close the Channel](#channel-owners-must-create-write-and-close-the-channel)
    + [Channel Consumers (Non-Owners)](#channel-consumers-non-owners)
  * [The select Statement](#the-select-statement)
    + [What Happens When Multiple Cases Are Ready?](#what-happens-when-multiple-cases-are-ready)
    + [What Happens If None of the Channels Are Ever Ready?](#what-happens-if-none-of-the-channels-are-ever-ready)
    + [What Happens if No Channels Are Ready But We Need to Do Something Else?](#what-happens-if-no-channels-are-ready-but-we-need-to-do-something-else)
    + [Empty Select](#empty-select)

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

### Channels
A channel is a conduit, river, stream, pipeline of information. Values are passed along the channel and read out downstream.

It can be bidirectional, ie data can be read and written from it.

Or unidirectional, ie it can only be read or written from but not both.

This is read-only: `<-chan string`.

This is write-only: `chan<- string`.

Here is an example that writes a string to a bidirectional `chan string` and then reads from it later on.

[fig-simple-chan.go](gos-concurrency-building-blocks%2Fchannels%2Ffig-simple-chan.go)

Channels are blocking - any goroutine that tries to write to a full channel will wait, and any goroutine that tries to read from an empty channel will wait until at least one item is placed on it.

#### Closed Channels
[fig-chan-recv-multi-value.go](gos-concurrency-building-blocks%2Fchannels%2Ffig-chan-recv-multi-value.go)

The second `ok` value is a way for a read operation to indicate whether the value from the channel was generated by a write somewhere else, or a default value because the channel is closed.

`close(chan)` is a way to say no more values are going to be added to the channel.

We can continue reading from a closed channel forever but it will just keep returning false in its second return value. This is so multiple goroutines can see a channel is closed.

[fig-reading-from-closed-channel.go](gos-concurrency-building-blocks%2Fchannels%2Ffig-reading-from-closed-channel.go)

#### Ranging Over a Channel
[fig-iterating-over-channel.go](gos-concurrency-building-blocks%2Fchannels%2Ffig-iterating-over-channel.go)

We can range over a channel and it will stop ranging when it sees the channel has been closed.

#### Unblocking Multiple Goroutines at Once
Because we can read from a closed channel infinitely, we can unblock multiple goroutines by closing the channel they're listening to.

[fig-unblocking-goroutines.go](gos-concurrency-building-blocks%2Fchannels%2Ffig-unblocking-goroutines.go)

This example has 5 goroutines all blocked on the line `<-begin`.

That's because no values are being added to the `begin` channel, it's empty.

When we `close(begin)`, each `<-begin` is unblocked. They're getting the default value from the channel because it's closed, but the point is, they're unblocked.

#### Buffered and Unbuffered Channels
A buffered channel can be created by passing a number when making it.

`intStream := make(chan int, 4)`

[fig-using-buffered-chans.go](gos-concurrency-building-blocks%2Fchannels%2Ffig-using-buffered-chans.go)

This channel can hold 4 values but if a fifth value is added, the code will block on that line.

If a value is read from the channel, that makes a space available, and the line adding the fifth value unblocks.

An unbuffered channel is what we've already created in code examples above.

These two statements are the same:

```go
intStream := make(chan int, 0)
intStream := make(chan int)
```
#### Nil Channels
Reading from and writing to a nil channel return a fatal error because of a deadlock.

Closing a nil channel results in a panic.

#### Channel Owners Must Create, Write and Close the Channel
To make working with channels easier, it helps to make it clear which goroutines own which channels.

If a goroutine owns a channel it should:

* instantiate the channel
* performs writes, or pass ownership to another goroutine
* close the channel
* encapsulate the previous three things and exposes them via a reader channel

By doing this we avoid problems like:

* deadlock when writing to a nil channel
* panic when closing a nil channel
* panic when writing to a closed channel
* panic when closing a channel more than once
* compilation errors from writing improper types to the channel

#### Channel Consumers (Non-Owners)
Goroutines that consume from a channel only have two things to worry about:

* knowing when a channel is closed
* handling blocks on the channel

Here's an example clarifying ownership and consumption of channels.

[fig-chan-ownership.go](gos-concurrency-building-blocks%2Fchannels%2Ffig-chan-ownership.go)

The lifecycle of the `resultStream` channel is encapsulated inside the `chanOwner` function.

### The select Statement
A `select` statement waits for channel reads and writes. It considers all cases simultaneously.

[fig-select-blocking.go](gos-concurrency-building-blocks%2Fthe-select-statement%2Ffig-select-blocking.go)

It blocks until one of the cases is ready.

#### What Happens When Multiple Cases Are Ready?
[fig-select-uniform-distribution.go](gos-concurrency-building-blocks%2Fthe-select-statement%2Ffig-select-uniform-distribution.go)

This code example has two cases in the `select` and both are ready to be read from.

Roughly half the time, the `select` reads from the `c1` channel, the other half from `c2`. This is by design. Go doesn't know which are the most important cases in a `select`, so if more than one is ready, it gives roughly equal precedence to them all.

#### What Happens If None of the Channels Are Ever Ready?
If this happens then the `select` will block forever.

This isn't very useful but we can add a timeout.

`time.After` returns a channel which becomes ready to read after the provided interval. An example of this is:

[fig-select-timeouts.go](gos-concurrency-building-blocks%2Fthe-select-statement%2Ffig-select-timeouts.go)

#### What Happens if No Channels Are Ready But We Need to Do Something Else?
We can add a `default` case so something can be done while waiting for the channels in the `select` to be ready.

Here's an example.

[fig-select-default-clause.go](gos-concurrency-building-blocks%2Fthe-select-statement%2Ffig-select-default-clause.go)

Usually you'll see a `select` containing a `default` case inside a `for` loop, so we can keep checking a channel but carry on past the `select` to do some other work.

Here's an example.

[fig-select-for-select-default.go](gos-concurrency-building-blocks%2Fthe-select-statement%2Ffig-select-for-select-default.go)

We drop into the `default` case that leaves the `select`, we do our work, then because we're inside a `for`, we go back up and run the `select` again. Eventually the channel will be ready and we'll run that case and break out of the loop.

#### Empty Select
`select {}` will block forever. May be useful in some contexts.