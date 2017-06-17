# Scheduler in F# and Kotlin

Let's take a close look at [Agent Based Scheduler](http://www.fssnip.net/6c/title/Agent-Based-Scheduler),
realize that it's super overcomplicated, we don't need an actor to start an async
computation, so we don't need messages to pack arguments to be able to send them
to the actor. What's left? Just a function that spins up an async:

```fsharp
open System
open System.Threading

let schedule<'a> (msg: 'a) (initialDelay: TimeSpan option) (delayBetween: TimeSpan option) (ct: CancellationToken) (receiver: 'a -> unit) =
    let computation = async {
        match initialDelay with
        | Some delay -> do! Async.Sleep (int delay.TotalMilliseconds)
        | _ -> ()

        do! initialDelay
            |> Option.map (fun delay -> Async.Sleep (int delay.TotalMilliseconds))
            |> Option.defaultValue (async.Return())

        match delayBetween with
        | None -> if not ct.IsCancellationRequested then receiver msg
        | Some delay ->
            while not ct.IsCancellationRequested do
                receiver msg
                do! Async.Sleep (int delay.TotalMilliseconds)

    }
    Async.Start(computation, ct)

[<EntryPoint>]
let main _ = Async.RunSynchronously <| async {
    use cts = new CancellationTokenSource()
    schedule 25 None (Some (TimeSpan.FromSeconds 1.)) cts.Token <| fun msg ->
        printfn "Scheduler got %A" msg

    do! Async.Sleep 5000
    cts.Cancel()
    printfn "cancelled"
    do! Async.Sleep 5000
    return 0
}
```

output

```
Scheduler got 25
Scheduler got 25
Scheduler got 25
Scheduler got 25
Scheduler got 25
cancelled
```

Quite a clean, easily readable code... yes?

Let's port it into Kotlin.

```kotlin
import kotlinx.coroutines.experimental.*
import java.time.Duration

suspend fun <T> schedule(ctx: CoroutineContext = CommonPool, msg: T, initialDelay: Duration? = null,
                         delayBetween: Duration? = null, receiver: (T) -> Unit): Job =
    launch(ctx) {
        initialDelay?.let { delay(it.toMillis()) }

        if (delayBetween == null) {
            if (isActive) receiver(msg)
        } else {
            while (isActive) {
                receiver(msg)
                delay(delayBetween.toMillis())
            }
        }
    }

fun main(args: Array<String>) = runBlocking {
    val job = schedule(CommonPool, 25, delayBetween = Duration.ofSeconds(1)) {
        println("Scheduler got $it")
    }

    delay(5000)
    job.cancel()
    println("cancelled")
    delay(5000)
}
```

output

```
Scheduler got 25
Scheduler got 25
Scheduler got 25
Scheduler got 25
Scheduler got 25
cancelled
```
* Free functions can have optional parameters in Kotlin, they don't have to be the
  last in signature and you can specify default values freely.
* `launch(...)` is equivalent to `Async.Start`, but you can specify a specific
"context" on which a coroutine should be running. Context includes the actual runner
(the common thread pool in this case), an equivalent of the .NET cancellation token
and any user data.
* `isActive` is the analog of .NET's `CancellationToken.IsCancellationRequested`, it's
a property of `Job` which is available implicitly inside `launch`.
* Coroutines do not require special syntax to mark suspending points ("binding" in F#'s
Async computation expression), it allows to write
```
initialDelay?.let { delay(it.toMillis()) }
```
You _cannot_ write
```fsharp
initialDelay |> Option.iter (fun delay -> do! Async.Sleep (int delay.TotalMilliseconds))
```
in F#. You _can_ write
```fsharp
do! initialDelay
    |> Option.map (fun delay -> Async.Sleep (int delay.TotalMilliseconds))
    |> Option.defaultValue (async.Return())
```
but it does not make sense.
* Here we use the `Job` returned by `launch` to cancel itself, which is not the same
thing as passing `CancellationTokenSource` to the `schedule` function in F#. It's not
needed in this case, but if we would like to cancel several `Job`s, I could create
an empty `Job` and "mix" it into coroutine context; it overloads `+` operator, so
`CommonPool + aJob` means a context which runs computations in the common thread pool and
uses the `aJob` as cancellation token source:

```fsharp
val job = Job()

schedule(CommonPool + job, 25, delayBetween = Duration.ofSeconds(1)) {
    println("Scheduler1 got $it")
}

schedule(CommonPool + job, 27, delayBetween = Duration.ofSeconds(2)) {
    println("Scheduler2 got $it")
}

delay(5000)
job.cancel()
println("cancelled")
delay(5000)
```

output

```
Scheduler1 got 25
Scheduler2 got 27
Scheduler1 got 25
Scheduler2 got 27
Scheduler1 got 25
Scheduler1 got 25
Scheduler2 got 27
Scheduler1 got 25
cancelled
```