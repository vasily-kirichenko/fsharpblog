# Regex Redux benchmark: F# vs Kotlin

OK, it's time for another F# -> Kotlin port experiment, it's [regex-redux benchmark game](https://benchmarksgame.alioth.debian.org/u64q/regexredux-description.html#regexredux) :) 
It exposes Kotlin's Coroutines a bit, compared to F# Async expressions. My version of F# code:

### F\#

```fsharp
open System.Text.RegularExpressions
open System
open System.IO
open System.Diagnostics

[<EntryPoint>]
let main args = 
    let sw = Stopwatch.StartNew()
    let regex pat = Regex (pat, RegexOptions.Compiled ||| RegexOptions.ExplicitCapture)
    let input = File.ReadAllText args.[0]
    let text = (regex ">.*\n|\n").Replace(input, "")

    async {
        let! newTextAsync =
            async {
               return
                   ["tHa[Nt]", "<4>"
                    "aND|caN|Ha[DS]|WaS", "<3>"
                    "a[NSt]|BY", "<2>"
                    "<[^>]*>", "|"
                    "\\|[^|][^|]*\\|" , "-"]
                   |> List.fold (fun s (code, alt) -> 
                        (regex code).Replace(s, alt)) text
            } |> Async.StartChild
        
        let! results =
            [| "agggtaaa|tttaccct"
               "[cgt]gggtaaa|tttaccc[acg]"
               "a[act]ggtaaa|tttacc[agt]t"
               "ag[act]gtaaa|tttac[agt]ct"
               "agg[act]taaa|ttta[agt]cct"
               "aggg[acg]aaa|ttt[cgt]ccct"
               "agggt[cgt]aa|tt[acg]accct"
               "agggta[cgt]a|t[acg]taccct"
               "agggtaa[cgt]|[acg]ttaccct" |]
            |> Array.map (fun pat -> async { return pat, ((regex pat).Matches text).Count })
            |> Async.Parallel
        
        for pat, count in results do printfn "%s %i" pat count
        
        let! newText = newTextAsync
        printfn "\n%i\n%i\n%i\n%d ms" input.Length text.Length newText.Length sw.ElapsedMilliseconds
    } 
    |> Async.RunSynchronously
    0

```
It start a number of async computations that matches a large text with regular expressions, 
all in parallel, then waits them to finish. Nothing special, just ordinary F# async code.

### Kotlin

```kotlin
import kotlinx.coroutines.experimental.*
import java.io.File
import java.util.regex.Pattern

fun main(args: Array<String>) = runBlocking {
    val start = System.nanoTime()
    val input = File(args.firstOrNull() ?: "d:\\input5000000.txt").readText()
    val text = ">.*\n|\n".toRegex().replace(input, "")

    val newTextAsync = async(CommonPool) {
        listOf(
            "tHa[Nt]" to "<4>",
            "aND|caN|Ha[DS]|WaS" to "<3>",
            "a[NSt]|BY" to "<2>",
            "<[^>]*>" to "|",
            "\\|[^|][^|]*\\|" to "-"
        ).fold(text) { s, (code, alt) -> code.toRegex().replace(s, alt) }
    }

    listOf("agggtaaa|tttaccct",
        "[cgt]gggtaaa|tttaccc[acg]",
        "a[act]ggtaaa|tttacc[agt]t",
        "ag[act]gtaaa|tttac[agt]ct",
        "agg[act]taaa|ttta[agt]cct",
        "aggg[acg]aaa|ttt[cgt]ccct",
        "agggt[cgt]aa|tt[acg]accct",
        "agggta[cgt]a|t[acg]taccct",
        "agggtaa[cgt]|[acg]ttaccct")
        .map { async(CommonPool) { it to Pattern.compile(it).splitAsStream(text).count() - 1 }}
        .map { it.await() }
        .map { (pat, count) -> println("$pat, $count") }

    val newText = newTextAsync.await()

    println(
        """
        ${input.length}
        ${text.length}
        ${newText.length}
        elapsed: ${(System.nanoTime().toDouble() - start.toDouble()) / 1E6} ms
        """.trimIndent())
}

```
The code is very, very similar to the F# one thanks to the new Kotlin feature - coroutines.
I don't have a solid understanding of then to decide if they are better or worse than F# 
computation expressions, but I can make a several remarks already:

* Coroutins are a general approach to generate continuation-passing style code out from
regular code, like F# does for computation expressions.
* Both allow to implement any kind of such expressions, like async computations, lazy 
sequence generators, etc.
* The main visual difference is that continuation points are not immediately visible
in source code. Compare F#
```fsharp
async {
    ...
    let! newText = newTextAsync
    ...
}
```
to Kotlin
```kotlin
runBlocking {
   ...
   val newText = newTextAsync.await()
   ...
}
```
However, Intellij IDEA shows all such point in a gutter, like this:

![image](https://user-images.githubusercontent.com/873919/27023475-6250ef06-4f5b-11e7-85d4-bfea8d3ec9f5.png)

which removes the problem (if it's a problem at all. I'm not sure if `let!`, `do!`, 
`yeild!` and `return!` keywords make F# code more or less readable. I mean Kotlin
coroutine code looks exactly the same as ordinary one and it may make it more
approachable for beginners).

As for performance of this benchmark, Kotlin code is faster (on a file generated with
Fasta benchmark for 5M entries): 5.6 seconds vs 7.6 seconds. 