# Synchronous channels based object pool: F# vs Kotlin

### F\#

This one I wrote [long time ago](https://github.com/Hopac/Hopac.Extras/blob/master/src/Hopac.Extras/ObjectPool.fs)
in Hopac:

```fsharp
type private PoolEntry<'a> = 
    { Value: 'a
      mutable LastUsed: DateTime }
    static member Create(value: 'a) = { Value = value; LastUsed = DateTime.UtcNow }
    interface IDisposable with
        member x.Dispose() = 
            // Mute exceptions those may be raised in instance's Dispose method to prevent the pool 
            // to stop looping.
            try 
                match box x.Value with
                | :? IDisposable as d -> d.Dispose()
                | _ -> ()
            with _ -> ()

/// Bounded pool of disposable objects. If number of given objects is equal to capacity then client will be blocked as it tries to get an instance. 
/// If an object in pool is not used more then inactiveTimeBeforeDispose period of time, it's disposed and removed from the pool. 
/// When the pool is disposing itself, it disposes all objects it caches first.
type ObjectPool<'a>(createNew: unit -> 'a, ?capacity: uint32, ?inactiveTimeBeforeDispose: TimeSpan) =
    let capacity = defaultArg capacity 50u
    let inactiveTimeBeforeDispose = defaultArg inactiveTimeBeforeDispose (TimeSpan.FromMinutes 1.)
    let reqCh = Ch<Promise<unit> * Ch<Choice<PoolEntry<'a>, exn>>>()
    let releaseCh = Ch<'a PoolEntry>()
    let maybeExpiredCh = Ch<'a PoolEntry>()
    let doDispose = IVar()
    let hasDisposed = IVar()
    
    let rec loop (available: 'a PoolEntry list, given: uint32) = Job.delay <| fun _ ->
        // an instance returns to pool
        let releaseAlt() =
            releaseCh ^=> fun instance ->
                instance.LastUsed <- DateTime.UtcNow
                Job.start (timeOut inactiveTimeBeforeDispose >>=.
                           (maybeExpiredCh *<+ instance)) >>=.
                loop (instance :: available, given - 1u)
        // request for an instance
        let reqAlt() =
             reqCh ^=> fun (nack, replyCh) ->
                let ok available instance =
                    replyCh *<- Ok instance ^=>. loop (available, given + 1u) <|>
                    nack ^=>. loop (instance :: available, given)
                match available with
                | instance :: available -> ok available instance
                | [] -> try createNew () |> PoolEntry.Create |> ok available
                        with e -> (replyCh *<- Fail e <|> nack) ^=>. loop (available, given)
        // an instance was inactive for too long
        let expiredAlt() =
            maybeExpiredCh ^=> fun instance ->
                if DateTime.UtcNow - instance.LastUsed > inactiveTimeBeforeDispose 
                   && List.exists (fun x -> obj.ReferenceEquals(x, instance)) available then
                    dispose instance
                    loop (available |> List.filter (fun x -> not <| obj.ReferenceEquals(x, instance)), given)
                else loop (available, given)

        // the entire pool is disposing
        let disposeAlt() = 
            doDispose ^=> fun _ ->
                // dispose all instances that are in pool
                available |> List.iter dispose
                // wait until all given instances are returns to pool and disposing them on the way
                Job.forN (int given) (releaseCh >>- dispose) >>=. hasDisposed *<= ()

        if given < capacity then
            // if number of given objects has not reached the capacity, synchronize on request channel as well
            releaseAlt() <|> expiredAlt() <|> disposeAlt() <|> reqAlt()
        else
            releaseAlt() <|> expiredAlt() <|> disposeAlt()
    
    do start (loop ([], 0u)) 
     
    let get() = reqCh *<+->- fun replyCh nack -> (nack, replyCh)

    /// Applies a function, that returns a Job, on an instance from pool. Returns `Alt` to consume 
    /// the function result.
    member __.WithInstanceJobChoice (f: 'a -> #Job<Choice<'r, exn>>) : Alt<Choice<'r, exn>> =
        get() ^=> function
            | Ok entry ->
               Job.tryFinallyJobDelay
                   <| fun _ -> f entry.Value
                   <| releaseCh *<- entry
            | Fail e -> Job.result (Fail e)

    interface IAsyncDisposable with
        member __.DisposeAsync() = IVar.tryFill doDispose () >>=. hasDisposed

    interface IDisposable with 
        /// Runs disposing asynchronously. Does not wait until the disposing finishes. 
        member x.Dispose() = (x :> IAsyncDisposable).DisposeAsync() |> start
```

### Kotlin

As always with me and Kotlin at this time, I'm not sure the following code is totally idiomatic:

```kotlin
class ObjectPool<out T>(
    createNew: () -> T,
    capacity: Int = 50,
    inactiveTimeBeforeDisposeMillis: Long = 10_000
) : AutoCloseable {

    private class PoolEntry<T>(val value: T, var lastUsed: ZonedDateTime = ZonedDateTime.now()) : AutoCloseable {
        override fun close() {
            if (value is AutoCloseable)
                try {
                    value.close()
                } catch (_: Throwable) {
                }
        }
    }

    private val reqCh = Channel<Channel<PoolEntry<T>>>()
    private val releaseCh = Channel<PoolEntry<T>>()
    private val maybeExpiredCh = Channel<PoolEntry<T>>()
    private val doCloseCh = Channel<Unit>()
    private val hasClosed = Channel<Unit>()
    private val available = Stack<PoolEntry<T>>()
    private var given = 0

    private fun <T> Stack<T>.tryPop() = if (empty()) null else pop()

    private fun <T> close(x: T) = (x as? AutoCloseable)?.close()

    init {
        launch(CommonPool) {
            whileSelect {
                // An instance returns to pool.
                releaseCh.onReceive { instance ->
                    instance.lastUsed = ZonedDateTime.now()
                    launch(CommonPool) {
                        delay(inactiveTimeBeforeDisposeMillis)
                        maybeExpiredCh.send(instance)
                    }
                    available.add(instance)
                    given--
                    true
                }
                // If number of given objects has not reached the capacity, synchronize on request channel as well.
                if (given < capacity) {
                    // request for an instance
                    reqCh.onReceive { replyCh ->
                        replyCh.send(available.tryPop() ?: PoolEntry(createNew()))
                        given++
                        true
                    }
                }
                // An instance was inactive for too long time, dispose and remove it.
                maybeExpiredCh.onReceive { instance ->
                    if (Duration.between(instance.lastUsed, ZonedDateTime.now()).toMillis() > inactiveTimeBeforeDisposeMillis
                        && available.any { it === instance }) {
                        println("${instance.value} expired")
                        close(instance)
                        available.removeIf { it === instance }
                    }
                    true
                }
                // The entire pool is closing.
                doCloseCh.onReceive {
                    // dispose all instances that are in pool
                    available.forEach { close(it) }
                    // wait until all given instances are returned to the pool and disposing them on the way
                    (1..given).forEach { close(releaseCh.receive()) }
                    hasClosed.send(Unit)
                    false
                }
            }
        }
    }

    suspend fun withInstance(f: suspend (T) -> Unit) {
        val replyCh = Channel<PoolEntry<T>>()
        reqCh.send(replyCh)
        val entry = replyCh.receive()
        try {
            f(entry.value)
        } finally {
            releaseCh.send(entry)
        }
    }

    override fun close() = runBlocking {
        doCloseCh.send(Unit)
        val timeoutSeconds = 10L
        select<Unit> {
            hasClosed.onReceive {}
            onTimeout(timeoutSeconds, TimeUnit.SECONDS) {
                throw TimeoutException("${javaClass.name} has not closed in $timeoutSeconds s")
            }
        }
    }
}
```

Usage:

```kotlin

fun main(args: Array<String>) = runBlocking {
    val startTime = ZonedDateTime.now()

    fun <T> ObjectPool<T>.useInstance(name: String) {
        fun log(msg: String) =
            println("%4d | %s | %s".format(Duration.between(startTime, ZonedDateTime.now()).toMillis(), name, msg))

        launch(CommonPool) {
            log("get...")
            withInstance {
                log("got $it!")
                delay(1000)
            }
            log("released.")
        }
    }

    var n = 0
    ObjectPool<Int>({ n++ }, 2, 1000).use {
        it.useInstance("1")
        it.useInstance("2")
        it.useInstance("3")
        it.useInstance("4")
        delay(5000)
    }
    Unit
}
```
Output
```
  13 | 3 | get...
  13 | 4 | get...
  13 | 2 | get...
  13 | 1 | get...
  28 | 1 | got 0!
  32 | 3 | got 1!
1042 | 1 | released.
1043 | 3 | released.
1043 | 2 | got 1!
1043 | 4 | got 0!
2043 | 2 | released.
2044 | 4 | released.
1 expired
0 expired
```

Kotlin's version does not use the Either monad to track errors, it just uses
exceptions, which I think is the proper way to do in any language which has built-in
support for them. It doesn't use negative acknowledgements either because they are not
needed.

The structure of both programs are similar, but Kotlin's one reads much easier thanks
to deep coroutine integration and avoiding custom operators. 