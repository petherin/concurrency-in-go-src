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
- [Concurrency Patterns in Go](#concurrency-patterns-in-go)
  * [Safe Operation of Concurrent Code](#safe-operation-of-concurrent-code)
    + [Immutable Data](#immutable-data)
    + [Confinement](#confinement)
      - [Ad Hoc](#ad-hoc)
      - [Lexical](#lexical)
  * [The for-select Loop](#the-for-select-loop)
    + [Sending Iteration Variables Out on a Channel](#sending-iteration-variables-out-on-a-channel)
    + [Looping Infinitely Waiting to Be Stopped](#looping-infinitely-waiting-to-be-stopped)
  * [Preventing Goroutine Leaks](#preventing-goroutine-leaks)
  * [The or-channel](#the-or-channel)
  * [Error Handling](#error-handling)
  * [Pipelines](#pipelines)
    + [Best Practices for Constructing Pipelines](#best-practices-for-constructing-pipelines)
    + [Some Handy Generators](#some-handy-generators)
      - [repeat](#repeat)
      - [take](#take)
      - [Type Assertion Stage](#type-assertion-stage)
  * [Fan-Out, Fan-In](#fan-out-fan-in)
  * [The or-done-channel](#the-or-done-channel)

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

Usually, the loop finishes before the goroutines inside the loop have a chance to start. So what is the value of `salutation`? It's the last value in the slice because it looped over everything and the last value was `good day`.

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

Because a lot of goroutines are getting but then replacing items from the pool, there are previously allocated items in the pool for the next goroutine to use, as opposed to having to create a new one.

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

## Concurrency Patterns in Go
Before we get to patterns, here are some things to consider when working with concurrent code.

### Safe Operation of Concurrent Code
There are several ways to safely work with concurrent code.

* synchronisation primitives for sharing memory (e.g. `sync.Mutex`)
* synchronisation via communicating (e.g. channels)
* immutable data
* confinement

The first two have been covered.

#### Immutable Data
If our concurrent processes only read data and never modify it, then it is implicitly concurrent-safe.

If we want to modify data we can copy its value (not its pointer reference) and modify the copy.

#### Confinement
Confinement ensures data is only ever available from one concurrent process.

When this is done, concurrent code is implicitly safe and synchronisation is unnecessary.

This can be done in two ways:

* ad hoc 
* lexical

##### Ad Hoc
This is where a convention is agreed on where data is only accessed from one place/function/loop/etc, and everyone has to agree this is how it will be done.

Here is an example where the `data` slice of integers is available from both the `loopData` function and the loop over the `handleData` channel. But by convention it's only accessed from the `loopData` function.

[fig-confinement-ad-hoc.go](concurrency-patterns-in-go%2Fconfinement%2Ffig-confinement-ad-hoc.go)

This breaks down the second someone commits code that breaks the convention.

##### Lexical
This involves scope in the code so you can't access the data from outside, say, a function, which restricts access to it.

Here's an example. It creates a channel inside a function and writes to it. Nothing else can write to it. The `consumer` function can only read from it because the channel is passed as `results <-chan int`.

[fig-confinement-ownership.go](concurrency-patterns-in-go%2Fconfinement%2Ffig-confinement-ownership.go)

This isn't the best example because channels are concurrent-safe anyway.

Here is another example without channels, using a data structure that isn't concurrent-safe.

[fig-confinement-structs.go](concurrency-patterns-in-go%2Fconfinement%2Ffig-confinement-structs.go)

The `printData` function has no access to the `data` variable declared outside it. Each concurrent instance of `printData` is passed a copy of some of the data instead. No synchronisation is required.

Concurrent code that uses lexical confinement is simpler than concurrent code that has channels, mutexes, etc.

### The for-select Loop
This is used a lot in concurrent code. There are a couple of scenarios it's used for.

#### Sending Iteration Variables Out on a Channel
Often you'll want to convert a set of values, usually in a slice, into values on a channel, like this:

```go
for _, s := range []string{"a", "b", "c", "d"} {
    select {
    case <-done:
        return
    case stringStream <- s:
    }
}
```

#### Looping Infinitely Waiting to Be Stopped
It's common to create goroutines that loop infinitely until stopped.

The first of a couple of variations of this is below. It keeps the `select` as short as possible.

```go
for {
    select {
    case <-done:
        return
    default:
    }

    // Do non-preemptable work
}
```

The second variation puts the work in the `default` case.

```go
for {
    select {
    case <-done:
        return
    default:
        // Do non-preemptable work
    }
}
```

It's not complex but it's worth mentioning because it shows up all over concurrent code.

### Preventing Goroutine Leaks
Goroutines are not garbage-collected so we don't want to leave them lying around. How do we ensure they're cleaned up?

Goroutines terminate under three conditions:

* It's completed its work
* It cannot continue due to an unrecoverable error
* It is told to stop

We get the first two for free when our algorithms end, or we get an error.

The parent or main goroutine should be able to tell its child goroutines to terminate.

Here's an example of a goroutine that remains in memory for the lifetime of the process.

[fig-goroutine-leaks-example.go](concurrency-patterns-in-go%2Fpreventing-goroutine-leaks%2Ffig-goroutine-leaks-example.go)

`nil` is passed to `doWork` so no strings are put on the `strings` channel.

To fix this we pass a `done` channel into `doWork`. By convention, it's a read-only channel called `done`.

[fig-goroutine-leaks-cancellation.go](concurrency-patterns-in-go%2Fpreventing-goroutine-leaks%2Ffig-goroutine-leaks-cancellation.go)

`doWork` listens for `done` on a `select` that also has a `default` so it can continue working.

The `done` channel is closed by the main goroutine after 1 second has passed.

The main goroutine blocks on `<-terminated`. When `doWork` returns it closes `terminated` and then `<-terminated` becomes unblocked and the program ends.

Here is another example where a goroutine doesn't terminate because it can't write to a channel because nothing is receiving from it.

[fig-leak-from-blocked-channel-write.go](concurrency-patterns-in-go%2Fpreventing-goroutine-leaks%2Ffig-leak-from-blocked-channel-write.go)

`newRandStream` function creates a goroutine that writes to the `randStream` channel. It will block on `randStream <- rand.Int()` until the main goroutine loops over the returned `randStream` and reads off 3 values.

When `newRandStream` tries to write a fourth value it blocks. Its deferred `Println` message is never done.

The fix is to send `newRandStream` a channel to tell it to stop.

[fig-leak-from-blocked-channel-write-solved.go](concurrency-patterns-in-go%2Fpreventing-goroutine-leaks%2Ffig-leak-from-blocked-channel-write-solved.go)

As before, a `done` channel is passed into the function and the function checks it in a `select`. 

The other case is adding values into the `randStream` channel.

In the main goroutine, `randStream` is read from 3 times and then `close(done)` is called.

This hits the `case <-done` part of `newRandStream`'s `select`, and we return from the function, ending the goroutine.

### The or-channel
Sometimes you could have more than one `done` channel. You could check them all in a `select` but you might not know how many there are at runtime, and doing it in a `select` could be quite verbose.

You can combine several `done` channels into an `or` channel, which can take a variadic number of channels and close all of them as soon as just one closes.

Here's an example.

[fig-or-channel.go](concurrency-patterns-in-go%2Fthe-or-channel%2Ffig-or-channel.go)

It takes a variadic slice of channels and returns a single channel. If there are 0 it returns nil, or the single channel if there's just one.

Otherwise the `or` function is called recursively until all channels have been added, and a single channel is returned.

The code example blocks until one of the `done` channels is closed.

### Error Handling
It can be hard to reason about what a goroutine should do with an error. Here's an example where the goroutine just prints the error and continues through its loop.

[fig-patterns-imporoper-err-handling.go](concurrency-patterns-in-go%2Ferror-handling%2Ffig-patterns-imporoper-err-handling.go)

A better way to handle errors is to have concurrent processes send their errors to another part of the program that has complete information about the program's state.

In the example below, the `checkStatus` function returns a `<-chan Result` where `Result` contains the response and an error.

[fig-patterns-proper-err-handling.go](concurrency-patterns-in-go%2Ferror-handling%2Ffig-patterns-proper-err-handling.go)

The main goroutine can range over the returned `<-chan Result` and handle any errors itself, rather than relying on the function that's doing the work.

We can be more intelligent in our error handling. Here's an example of the code above that stops ranging over the returned channel if it receives 3 errors.

[fig-stop-after-three-errors.go](concurrency-patterns-in-go%2Ferror-handling%2Ffig-stop-after-three-errors.go)

The point of all this is that errors should be considered to be as important as the actual values returned from goroutines.

### Pipelines
A pipeline is a series of stages that take data in, process it, and pass the data back out.

Here is a non-concurrent example where the stages are `multiply` and `add`, and the two stages are combined to act on a slice of ints.

[fig-functional-pipeline-combination.go](concurrency-patterns-in-go%2Fpipelines%2Ffig-functional-pipeline-combination.go)

The `add` and `multiply` stages both consume and return the same type of `[]int`, meaning they can be combined to form a pipeline. The results of `add` can be fed into `multiply` and vice versa.

This allows any combination of the stages and add as many as we want, e.g.

[fig-adding-additional-stage-to-pipeline.go](concurrency-patterns-in-go%2Fpipelines%2Ffig-adding-additional-stage-to-pipeline.go)

Because these functions take and return slices of data, the stages are performing _batch processing,_ as in they act on chunks of data all at once instead of one value at a time.

_Stream processing_ is where the stages receive and emit one element at a time.

This is how the above code looks if we use stream processing and process one value at a time.

[fig-pipelines-func-stream-processing.go](concurrency-patterns-in-go%2Fpipelines%2Ffig-pipelines-func-stream-processing.go)

In this method we are calling the pipeline _per iteration_, whereas before we were ranging over the pipeline's results.

#### Best Practices for Constructing Pipelines
Here's the above pipeline rewritten to use channels.

[fig-pipelines-chan-stream-processing.go](concurrency-patterns-in-go%2Fpipelines%2Fbest-practices-for-constructing-pipelines%2Ffig-pipelines-chan-stream-processing.go)

The `generator` function is a mainstay of pipeline code. It converts a discrete set of values into a stream of data on a channel.

A `done` channel is passed in, as the idiomatic way of signalling a goroutine to stop.

A variadic slice of `int` is also passed in , and a `<-chan int` is returned.

Inside, a goroutine is started which ranges over the passed data, which has a `select` to listen to the `done` channel and to push integers onto the `chan int`. 

The `multiply` and `add` functions have been changed to accept a `done` channel and an `<-chan int`, and they return `<-chan int` too.

Inside, they start a goroutine which ranges over the incoming `<-chan int`. Inside the loop is a  `select` to listen for `done`. There is also a case where the current `int` is either multiplied or added to, which is then pushed onto the `<-chan int` to be returned.

In the main goroutine, we get the `<-chan int` from `generator`. This is used to set a `pipeline` variable which is set to the result of stringing together the pipeline stages.

The stages are, for a stream of numbers, multiply by two, add one, then multiply the result by two.

Finally, we range over the `pipeline` variable to print out the results.

This approach provides two benefits:

1. Because channels are being used, we can range over the results, and at each stage we are running concurrent-safe code because the inputs and outputs are themselves concurrent-safe.
2. Each stage of the pipeline is executing concurrently. So any stage only needs to wait for its inputs, and to be able to send its outputs.

Below is a table showing what values are on which channels, at each stage of the `for` loop. Multiple events occur during a single iteration - these are the steps.

| Iteration | Step | Generator (from incoming _[]int_) | Multiply (from _<-chan int_ <br/>returned by Generator) | Add (from _<-chan int_ returned<br/>from Multipler which multiples by 2) | Second Multiply (from _<-chan int_ returned <br/>by Add which adds 1) | Value (from _<-chan int_ returned by second <br/>Multiply which multiples by 2) |
|-----------|------|--------------------------------------------------------------|---------------------------------------------------------|--------------------------------------------------------------------------|-----------------------------------------------------------------------|---------------------------------------------------------------------------------|
| 0         | 1    | 1                                                            |                                                         |                                                                          |                                                                       |                                                                                 |
| 0         | 2    |                                                                  | 1                                                       |                                                                          |                                                                       |                                                                                 |
| 0         | 3    |2                                                                 |                                                         | 2                                                                        |                                                                       |                                                                                 |
| 0         | 4    |                                                                 | 2                                                       |                                                                          | 3                                                                     |                                                                                 |
| 0         | 5    |3                                                                 |                                                         | 4                                                                        |                                                                       | 6                                                                               |
| 1         | 6    |                                                                  | 3                                                       |                                                                          | 5                                                                     |                                                                                 |
| 1         | 7    |4                                                                 |                                                         | 6                                                                        |                                                                       | 10                                                                              |
| 2         | 8    |(closed)                                                          | 4                                                       |                                                                          | 7                                                                     |                                                                                 |
| 2         | 9    |                                                                  | (closed)                                                | 8                                                                        |                                                                       | 14                                                                              |
| 3         | 10   |                                                                  |                                                         | (closed)                                                                 | 9                                                                     |                                                                                 |
| 3         | 11   |                                                                  |                                                         |                                                                          | (closed)                                                              | 18                                                                              |

All of the above can't start until we start reading from the pipeline.

The chain of channels is:

`generator -> multiply -> add -> multiply`

Nothing can be added to `generator` because the entire pipeline is blocked until something is ready to read from `multiply` at the end.

This happens when we start ranging over `pipeline`, which contains the channel returned by the last `multiply`.

Here are the first 5 steps, covering the first iteration of the main goroutine's `for` loop.

1. The first value (1) in the slice of `ints` sent to the `generator` function can now be pushed onto `generator`'s `intStream`.
2. The range loop in `multiplier` can now get the 1, and add it to `multiplier`'s channel.
3. Now that the `generator` channel is free, the next number, 2, can be pushed to it. **Concurrently**, the 1, now multiplied to 2, moves to the `add` channel.
4. The first `multiply` channel gets the 2 from `generator`. **Concurrently**, the range in the second `multiply` gets the value (now 3) from the `add` channel.
5. The next value 3 can go onto the `generator` channel. **Concurrently**, the `add` function gets 4 from the `multiplier` channel. **Also Concurrently**, the 3 (now multiplied to 6) is read from the second `multiply` channel and is the first number to be printed out.

The remaining iterations are more of the same.

The pipeline can be stopped at any stage by calling `close(done)`. All stages are checking `done`, and the channels they range over will `break `when an upstream stage closes its channel.

#### Some Handy Generators
A generator for a pipeline is any function that converts a set of discrete values into a stream of values on a channel.

##### repeat
A `repeat` generator repeats the values passed to it infinitely, until told to stop. An example is included in the next section.

##### take
This will only take the specified number of items from its incoming channel, and then exit. With `repeat`, it can be very powerful.

[fig-take-and-repeat-pipeline.go](concurrency-patterns-in-go%2Fpipelines%2Fsome-handy-generators%2Ffig-take-and-repeat-pipeline.go)

The lines that construct the pipeline are:

```
for num := range take(done, repeat(done, 1), 10) {
    fmt.Printf("%v ", num)
}
```
The `repeat` _could_ generate an infinite number of values, but because it is blocked waiting for `take` to read from its channel, and `take` is getting a set number of items, this makes `repeat` very efficient.

##### Type Assertion Stage
The `repeat` and `take` examples consume and return `chan interface{}` because the author of the book states it's useful to have a library of generators where they can handle any type.

With generics in Go, it feels like we could still have a general library of generators but they could be a constrained set of generic types.

For completeness, here's an example from the book that introduces a type assertions stage to convert a `chan interface{}` to a specific type, in this case `chan string`.

[fig-utilizing-string-stage.go](concurrency-patterns-in-go%2Fpipelines%2Fsome-handy-generators%2Ffig-utilizing-string-stage.go)

It's used like this, as the outer stage of a pipeline:

```go
for token := range toString(done, take(done, repeat(done, "I", "am."), 5)) {
    message += token
}
```

### Fan-Out, Fan-In
If a stage of your pipeline is slow, then upstream stages can become blocked and the entire pipeline can slow down.

A slow stage can be re-used on multiple goroutines to speed it up - this is fan-out, fan-in.

Fan-out is starting multiple goroutines to handle input from the pipeline.

Fan-in is combining multiple results into one channel.

You could consider this pattern if:

* the stage doesn't rely on values that the stage before has calculated, in other words, the stage is order-independent
* it takes a long time to run

Here is an inefficient prime number finder with a slow `primeFinder` stage.

[fig-naive-prime-finder.go](concurrency-patterns-in-go%2Ffan-out-fan-in%2Ffig-naive-prime-finder.go)

The pipeline bit is:

```go
randIntStream := toInt(done, repeatFn(done, rand))
fmt.Println("Primes:")
for prime := range take(done, primeFinder(done, randIntStream), 10) {
    fmt.Printf("\t%d\n", prime)
}
```

`repeatFn` is a stage that takes `rand`, a function that returns a random number as an `interface{}`. 

`repeatFn` returns `<-chan interface{}` so we wrap it in `toInt` which returns `<-chan int`.

We then loop over a pipeline of `take(done, primeFinder(done, randIntStream), 10)`.

Inside the pipeline, the `randIntStream <-chan int` is passed to `primeFinder`, which feeds into `take` which takes 10 results.

`primeFinder` is slow so it would improve the speed of the pipeline if several concurrent `primeFinder`s were running.

Here's the fan-out, fan-in version that does that.

[fig-fan-out-naive-prime-finder.go](concurrency-patterns-in-go%2Ffan-out-fan-in%2Ffig-fan-out-naive-prime-finder.go)

There is a new `fanIn` stage that takes in a variadic slice of channels `...<-chan interface{}`.

It declares a `chan interface{}` and returns it as `<-chan interface{}`.

It declares a function that takes `<-chan interface{}`, ranges over it, and pushes to the channel the function returns.

For each `<-chan interface{}` we call the function that gets the values from a channel.

A WaitGroup waits for the `...<-chan interface{}` to all be processed and closes the returned channel.

The bit in the main goroutine that constructs the pipeline is different from the previous version.

```go
numFinders := runtime.NumCPU()
fmt.Printf("Spinning up %d prime finders.\n", numFinders)
finders := make([]<-chan interface{}, numFinders)
fmt.Println("Primes:")
for i := 0; i < numFinders; i++ {
    finders[i] = primeFinder(done, randIntStream)
}

for prime := range take(done, fanIn(done, finders...), 10) {
    fmt.Printf("\t%d\n", prime)
}
```

Here's the fan-out. Instead of `primeFinder` being included in the pipeline and ranged over, several `primeFinder`s are called and added to a slice of finders - `[]<-chan interface{}`.

In the pipeline that we range over, we replace `primeFinder(done, randIntStream)` with `fanIn(done, finders...)` which converts the multiple channels from the slice of finders into a single channel.

This speeds things up.

### The or-done-channel
Sometimes you will be using channels from all over the place in your code. You can't be sure that if you cancel a goroutine using a `done` channel, the goroutine will behave properly and cancel the channels _it_ returns.

You could write cumbersome code to wrap the channel in a `for` and `select` to check for `done` and to check if the channel is returning `false` for its second return value. But it gets untidy if there are further channels you need to check.

The `or-done-channel` is a function that encapsulates this logic so you can just write:

```go
for val := range orDone(done, myChan) {
	// Do something with val
}
```

This is the `orDone` function.

```go
orDone := func(done, c <-chan interface{}) <-chan interface{} {
    valStream := make(chan interface{})
    go func() {
        defer close(valStream)
        for {
            select {
            case <-done:
                return
            case v, ok := <-c:
                if ok == false {
                    return
                }
                select {
                case valStream <- v:
                case <-done:
                }
            }
        }
    }()
    return valStream
}
```