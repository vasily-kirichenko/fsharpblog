# Why ideomatic F# APIs are _good_ #1

### [Akka.NET](https://github.com/akkadotnet/akka.net) stashing actor

```fsharp
type Message =
    | InitializationDone
    | Print of string

type StashingActor() as self =
    inherit UntypedActor()
    
    do self.Become(self.Initializing)
    
    interface IWithUnboundedStash with
        member val Stash = null with get, set

    override this.OnReceive _ = ()
    
    member this.Initializing(msg: obj) : unit =
        match msg with
        | :? Message as msg ->
            match msg with
            | InitializationDone ->
                (this :> IWithUnboundedStash).Stash.UnstashAll()
                this.Become(this.Operating)
            | Print s ->
                printfn "%s stashed" s
                (this :> IWithUnboundedStash).Stash.Stash()
        | _ -> ()
    
    member this.Operating(msg: obj) : unit =
        match msg with
        | :? Message as msg ->
            match msg with
            | Print s -> printfn "Actually printing: %s" s
            | _ -> ()
        | _ -> ()

[<EntryPoint>]
let main _ = 
    let system = System.create "test" (Configuration.defaultConfig())
    let a = system.ActorOf<StashingActor>()
    a.Tell (Print "foo")
    a.Tell (Print "bar")
    a.Tell InitializationDone
    a.Tell (Print "buzz")
    Console.ReadKey() |> ignore
    0

```

### [Akkling](https://github.com/Horusiath/Akkling) stashing actor

```fsharp
type Message =
    | InitializationDone
    | Print of string

module rec StashingActor =

    let initializing (a: Actor<Message>) = function
        | InitializationDone ->
            a.UnstashAll()
            become (operating a)
        | Print s ->
            printfn "%s stashed" s
            a.Stash() |> ignored

    let operating (a: Actor<Message>) = function
        | Print s -> printfn "Actually printing: %s" s |> ignored
        | _ -> unhandled()

[<EntryPoint>]
let main _ = 
    let system = System.create "test" (Configuration.defaultConfig())
    let a = spawnAnonymous system (props (actorOf2 StashingActor.initializing)) 
    
    
    a <! Print "foo"
    a <! Print "bar"
    a <! InitializationDone
    a <! Print "buzz"
    
    Console.ReadKey() |> ignore
    0
```