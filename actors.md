# Actors benchmark: F# vs Kotlin

It turns out Kotlin has a full fledged implementation of CML / Go / Hopac style concurrency.
It has synchronous channels, selective synchronization on send and receive (and it allows to 
mix both in the same select, too), buffered channels, etc; 
[read this article for details](https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md).

Let's pick a [simple benchmark](https://github.com/Hopac/Hopac/blob/master/Benchmarks/CounterActor/CounterActor.fs) 
from [Hopac repository](https://github.com/Hopac/Hopac) and translate it into Kotlin.

### F\# (Hopac)

```fsharp
// Copyright (C) by Housemarque, Inc.
let (^) x = (<|) x

type Msg =
    | Add of int64
    | GetAndReset of IVar<int64>

type CounterActor =
    | CA of Ch<Msg>

let create : Job<CounterActor> = job {
    let inCh = Ch ()
    let state = ref 0L
    do! Job.foreverServer
         (inCh >>= function
           | Add n ->
             state := !state + n
             Job.unit ()
           | GetAndReset replyVar ->
             let was = !state
             state := 0L
             replyVar *<= was)
    return CA inCh
}

let add (CA inCh) (n: int64) = inCh *<- Add n

let getAndReset (CA inCh) =
    inCh *<+=>- GetAndReset :> Job<_>

let run numPerThread =
    printf "ChMsg: "
    let timer = Stopwatch.StartNew ()
    
    ignore ^ run ^ job {
      let! actor = create
      do! seq {1 .. Environment.ProcessorCount}
          |> Seq.Con.iterJob
              (fun _ -> Job.forN numPerThread (add actor 100L))
      return! getAndReset actor
    }
    
    let d = timer.Elapsed
    printf "%d * %8d msgs => %8.0f msgs/s\n"
           Environment.ProcessorCount numPerThread
           (float (Environment.ProcessorCount * numPerThread) / d.TotalSeconds)

for n in [300; 3000; 30000; 300000; 3000000] do
    run n
    GC.Collect(2)
    GC.Collect(2)
    GC.Collect(2)
```
Output:
```
ChMsg: 8 *      300 msgs =>    75259 msgs/s
ChMsg: 8 *     3000 msgs =>  4393351 msgs/s
ChMsg: 8 *    30000 msgs =>  4591289 msgs/s
ChMsg: 8 *   300000 msgs =>  4620537 msgs/s
ChMsg: 8 *  3000000 msgs =>  4721725 msgs/s
```

### F\# (MailboxProcessor)

This version uses the built-in actor-like primitive named `MailboxProcessor`, it's 
combination of a message queue and a async loop that processes the messages in 
unblocking manner.

```fsharp
type Msg =
    | Add of int64
    | GetAndReset of AsyncReplyChannel<int64>

let create() = MailboxProcessor.Start(fun mb ->
    async {
        let mutable state = 0L
        while true do
            let! msg = mb.Receive()
            match msg with
            | Add n -> state <- state + n
            | GetAndReset reply ->
                let was = state
                state <- 0L
                reply.Reply was
    })

let add (mp: MailboxProcessor<Msg>) (n: int64) = mp.Post (Add n)

let getAndReset (mp: MailboxProcessor<Msg>) = mp.PostAndReply GetAndReset

let run numPerThread =
    let timer = Stopwatch.StartNew()
    printf "MailboxProcessor: "
    let actor = create()
    
    let _ =
        [1..Environment.ProcessorCount]
        |> List.map (fun _ ->
             async { for _ in 1..numPerThread do
                        add actor 100L })
        |> Async.Parallel
        |> Async.RunSynchronously
                        
    let _ = getAndReset actor
    let d = timer.Elapsed
    printf "%d * %8d msgs => %8.0f msgs/s\n"
           Environment.ProcessorCount numPerThread
           (float (Environment.ProcessorCount * numPerThread) / d.TotalSeconds)
           
for n in [300; 3000; 30000; 300000; 3000000] do
    run n
    GC.Collect(2)
    GC.Collect(2)
    GC.Collect(2)           
```
Output:
```
8 *      300 msgs =>   224431 msgs/s
8 *     3000 msgs =>  1480677 msgs/s
8 *    30000 msgs =>  1138195 msgs/s
8 *   300000 msgs =>  1074774 msgs/s
8 *  3000000 msgs =>  1159185 msgs/s
```
### Kotlin (plane actor)

```kotlin
sealed class Msg
class Add(val n: Long) : Msg()
class GetAndReset(val reply: SendChannel<Long>) : Msg()

fun create() = actor<Msg>(CommonPool) {
    var state = 0L
    for (msg in channel) {
        when (msg) {
            is Add -> state += msg.n
            is GetAndReset -> {
                val was = state
                state = 0L
                msg.reply.send(was)
            }
        }
    }
}

suspend fun getAndReset(actor: ActorJob<Msg>): Long {
    val reply = Channel<Long>()
    actor.send(GetAndReset(reply))
    return reply.receive()
}

suspend fun run(numPerThread: Int) {
    val processorCount = Runtime.getRuntime().availableProcessors()

    val elapsed = measureTimeMillis {
        val actor = create()
        (1..processorCount).map {
            async(CommonPool) {
                for (n in 1..numPerThread) {
                    actor.send(Add(100L))
                }
                getAndReset(actor)
            }
        }.forEach { it.await() }
    }
    print("%d * %8d msgs => %8.0f msgs/s\n"
            .format(processorCount, numPerThread, processorCount * numPerThread / (elapsed / 1000.0)))
}

fun main(args: Array<String>) = runBlocking {
    for (n in listOf(300, 3_000, 30_000, 300_000, 3_000_000)) {
        run(n)
    }
}
```
Output:
```
8 *      300 msgs =>    60000 msgs/s
8 *     3000 msgs =>   470588 msgs/s
8 *    30000 msgs =>  1012658 msgs/s
8 *   300000 msgs =>  1355167 msgs/s
8 *  3000000 msgs =>  1316222 msgs/s
```

This version code uses the `actor` function that simplify things a bit. If you are curious
what such an actor would look like being implemented from scratch, this is it:

```fsharp
fun create(): Channel<Msg> {
    val ch = Channel<Msg>()
    async(CommonPool) {
        var state = 0L
        for (msg in ch) {
            when (msg) {
                is Add -> state += msg.n
                is GetAndReset -> {
                    val was = state
                    state = 0L
                    msg.reply.send(was)
                }
            }
        }
    }
    return ch
}
```


### Kotlin (select from channels)

```kotlin
class Actor() {
    private var state = 0L
    private val addCh = Channel<Long>()
    private val getAndResetCh = Channel<SendChannel<Long>>()

    // expose `SendChannel`s because client code should not be able to receive our messages.
    val add = addCh as SendChannel<Long>
    val getAndReset = getAndResetCh as SendChannel<SendChannel<Long>>

    init {
        launch(CommonPool) {
            while (true) {
                select<Unit> {
                    addCh.onReceive {
                        state += it
                    }
                    getAndResetCh.onReceive {
                        val was = state
                        state = 0
                        it.send(was)
                    }
                }
            }
        }
    }
}

suspend fun Actor.getAndReset(): Long =
    Channel<Long>().let {
        getAndReset.send(it)
        it.receive()
    }

suspend fun run(numPerThread: Int) {
    val processorCount = Runtime.getRuntime().availableProcessors()

    val elapsed = measureTimeMillis {
        val actor = Actor()
        (1..processorCount).map {
            launch(CommonPool) {
                for (n in 1..numPerThread) {
                    select<Unit> {
                        actor.add.onSend(100L) {}
                        // just to show how timeouts can be added in composable way
                        onTimeout(100) { println("Adding to the actor timed out!") }
                    }
                }
                actor.getAndReset()
            }
        }.forEach { it.join() }
    }
    print("%d * %8d msgs => %8.0f msgs/s\n"
        .format(processorCount, numPerThread, processorCount * numPerThread / (elapsed / 1000.0)))
}

fun main(args: Array<String>) = runBlocking {
    for (n in listOf(300, 3_000, 30_000, 300_000, 3_000_000)) {
        run(n)
    }
}
```
Output:
```
8 *      300 msgs =>    26966 msgs/s
8 *     3000 msgs =>   156863 msgs/s
8 *    30000 msgs =>   342857 msgs/s
8 *   300000 msgs =>   776950 msgs/s
8 *  3000000 msgs =>   763310 msgs/s
```

It's been awhile I used Hopac for the last time and almost forget how hard it is. I find 
the Kotlin and `MailboxProcessor` versions tremendously easier to read and 
it was a pleasure to write.

So, Kotlin's actor is ~3.6 times slower than Hopac and ~10% faster than `MailboxProcessor`.
`select` introduces significant overhead, it's ~40% slower than `actor`.

<div style="text-align:center">
<img src ="https://user-images.githubusercontent.com/873919/27082900-3b1d8580-504f-11e7-8d31-8d0df9ecb7ee.png"/>
</div>

