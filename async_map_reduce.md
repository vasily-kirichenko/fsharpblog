# Async MapReduce: F# to Kotlin

Another day, another [victim](http://www.fssnip.net/73/title/Async-based-MapReduce)

### F\#

```fsharp
let noiseWords = [|"a"; "about"; "above"; "all"; "along" ... |]

let links =
    [|  "http://www.cs.ukzn.ac.za/~hughm/ap/data/shakespeare/allswellthatendswell.txt"
        "http://www.cs.ukzn.ac.za/~hughm/ap/data/shakespeare/amsnd.txt"
        ...
    |]

let Download(url : Uri) =
    async {
        let client = new WebClient()
        let! html = client.AsyncDownloadString(url)
        return html
    }

let (<||>) first second = async { let! results = Async.Parallel([|first; second|]) in return (results.[0], results.[1]) }

let mapReduce (mapF : 'T -> Async<'R>) (reduceF : 'R -> 'R -> Async<'R>) (input : 'T []) : Async<'R> =
    let rec mapReduce' s e =
        async {
            if s + 1 >= e then return! mapF input.[s]
            else
                let m = (s + e) / 2
                let! (left, right) =  mapReduce' s m <||> mapReduce' m e
                return! reduceF left right
        }
    mapReduce' 0 input.Length


// Example: the classic map/reduce word-count

let mapF uri =
    async {
        let! text = Download(new Uri(uri))
        let words = text.Split([|' '; '.'; ','|], StringSplitOptions.RemoveEmptyEntries)
        return
            words
            |> Seq.map (fun word -> word.ToUpper())
            |> Seq.filter (fun word -> not (noiseWords |> Seq.exists (fun noiseWord -> noiseWord.ToUpper() = word)) && Seq.length word > 3)
            |> Seq.groupBy id
            |> Seq.map (fun (key, values) -> (key, values |> Seq.length)) |> Seq.toList
    }

let reduceF (left : (string * int) list) (right : (string * int) list) =
    async {
        return
            left @ right
            |> Seq.groupBy fst
            |> Seq.map (fun (key, values) -> (key, values |> Seq.sumBy snd))
            |> Seq.toList
    }

mapReduce mapF reduceF links
|> Async.RunSynchronously
|> List.sortBy (fun (_, count) -> -count)
```

### Kotlin

```kotlin
val noiseWords = listOf("a", "about", "above", "all", "along" ...)

val links = listOf(
    "http://www.cs.ukzn.ac.za/~hughm/ap/data/shakespeare/allswellthatendswell.txt",
    "http://www.cs.ukzn.ac.za/~hughm/ap/data/shakespeare/amsnd.txt"
    ...)

// Wrap Fuel's continuation passing async API to a coroutine.
suspend fun download(url: String): String = suspendCoroutine { cont ->
    try {
        url.httpGet().timeout(5000).responseString { _, _, result ->
            when (result) {
                is Result.Failure -> cont.resumeWithException(result.getException())
                is Result.Success -> cont.resume(result.value)
            }
        }
    } catch(e: Throwable) {
        cont.resumeWithException(e)
    }
}

suspend fun <T, R> mapReduce(mapF: suspend (T) -> R,
                             reduceF: suspend (R, R) -> R,
                             input: List<T>,
                             s: Int = 0,
                             e: Int = input.size): R {

    if (s + 1 >= e) return mapF(input[s])
    else {
        val m = (s + e) / 2
        // Start two coroutines in parallel and...
        val left = async(CommonPool) { mapReduce(mapF, reduceF, input, s, m) }
        val right = async(CommonPool) { mapReduce(mapF, reduceF, input, m, e) }
        // ...await them both.
        return reduceF(left.await(), right.await())
    }
}

// Example: the classic map/reduce word-count

suspend fun mapF(uri: String): List<Pair<String, Int>> =
    // Note that we don't have to bind the result of `download` to an intermidite variable.
    download(uri)
        .split(' ', '.', ',', '\t', '\n', '\r').map { it.trim() }.filterNot { it.isBlank() }
        .map { it.toUpperCase() }
        .filter { it.length > 3 }
        .filterNot { word -> noiseWords.any { it.toUpperCase() == word } }
        .groupBy { it }
        .map { (key, values) -> key to values.count() }

fun reduceF(left: List<Pair<String, Int>>, right: List<Pair<String, Int>>) =
    (left + right)
        .groupBy { it.first }
        .map { (key, values) -> key to values.sumBy { it.second } }

fun main(args: Array<String>) = runBlocking {
    println(
        // Unfortunately, passing suspend functions to HOFs is not supported yet,
        // so we have to wrap `mapF` and `reduceF` into lambdas.
        mapReduce(
            { mapF(it) },
            { x, y -> reduceF(x, y) },
            links
        ).sortedBy { (_, count) -> count }
    )
}
```

My feelings as I was writing this Kotlin code were: "that's what asynchronous programming should be".
The both versions look very similar for the first glance, but Kotlin code is just simpler
because asynchronousy is deeper integrated into the language. As I'm starting to see,
everything in Kotlin is designed in such a way that it helps to make code as simple,
natural and no-surprising as possible. It's not the most powerful language out there, but
it's the most pleasant to write in for sure.