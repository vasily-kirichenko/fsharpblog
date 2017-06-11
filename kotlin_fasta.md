# Fasta benchmark: F# vs Kotlin

Let's look at a small F# program ported to Kotlin. It's a [Fasta](https://benchmarksgame.alioth.debian.org/u64q/fasta-description.html#fasta)
benchmark. The [current F# code](https://benchmarksgame.alioth.debian.org/u64q/program.php?test=fasta&lang=fsharpcore&id=1): 

### F\#

```fsharp
// The Computer Language Benchmarks Game
// http://benchmarksgame.alioth.debian.org/
//
// Contributed by Valentin Kraevskiy

let im = 139968
let ia = 3877
let ic = 29573
let mutable seed = 42;

let inline random max =
    seed <- (seed * ia + ic) % im
    max * float seed / float im

let alu =
    "GGCCGGGCGCGGTGGCTCACGCCTGTAATCCCAGCACTTTGG\
     GAGGCCGAGGCGGGCGGATCACCTGAGGTCAGGAGTTCGAGA\
     CCAGCCTGGCCAACATGGTGAAACCCCGTCTCTACTAAAAAT\
     ACAAAAATTAGCCGGGCGTGGTGGCGCGCGCCTGTAATCCCA\
     GCTACTCGGGAGGCTGAGGCAGGAGAATCGCTTGAACCCGGG\
     AGGCGGAGGTTGCAGTGAGCCGAGATCGCGCCACTGCACTCC\
     AGCCTGGGCGACAGAGCGAGACTCCGTCTCAAAAA"B

let makeCumulative = 
    List.fold (fun (cp, res) (c, p) -> cp + p, (c, cp + p) :: res) (0.0, []) 
    >> snd >> List.toArray

let homoSapiens = 
    makeCumulative
        ['a'B, 0.3029549426680
         'c'B, 0.1979883004921
         'g'B, 0.1975473066391
         't'B, 0.3015094502008]
     
let iub = 
    makeCumulative 
        ['a'B, 0.27; 'c'B, 0.12; 'g'B, 0.12
         't'B, 0.27; 'B'B, 0.02; 'D'B, 0.02
         'H'B, 0.02; 'K'B, 0.02; 'M'B, 0.02
         'N'B, 0.02; 'R'B, 0.02; 'S'B, 0.02
         'V'B, 0.02; 'W'B, 0.02; 'Y'B, 0.02]

let inline selectRandom (f: _ [])  =     
    let r = random 1.0 
    let rec find = function
        | 0 -> fst f.[0]
        | n when r < snd f.[n] -> fst f.[n]
        | n -> find (n - 1)
    find <| f.Length - 1
   
let width = 60 
let stream = System.Console.OpenStandardOutput ()
let buffer = Array.create 1024 0uy
let mutable index = 0

let inline flush () =    
    stream.Write (buffer, 0, index)
    index <- 0
    
let inline write b = 
    buffer.[index] <- b
    index <- index + 1
    if index = buffer.Length then flush ()        
    
let randomFasta desc table n =
    Array.iter write desc
    for i in 1 .. n do
        write <| selectRandom table
        if i % width = 0 then write '\n'B
    if n % width <> 0 then write '\n'B

let repeatFasta desc (table : byte []) n =
    Array.iter write desc 
    for i in 1 .. n do
        write <| table.[(i - 1) % table.Length]
        if i % width = 0 then write '\n'B
    if n % width <> 0 then write '\n'B
        
[<EntryPoint>]
let main args =
    let n = try int args.[0] with _ -> 1000
    repeatFasta ">ONE Homo sapiens alu\n"B alu (2 * n)
    randomFasta ">TWO IUB ambiguity codes\n"B iub (3 * n)
    randomFasta ">THREE Homo sapiens frequency\n"B homoSapiens  (5 * n)
    flush ()
    0
```  
It's a typical optimized F# code.

### Kotlin:

```kotlin
val im = 139968
val ia = 3877
val ic = 29573
var seed = 42

fun random(max: Double): Double {
    seed = (seed * ia + ic) % im
    return max * seed.toDouble() / im.toDouble()
}

val alu = """
    GGCCGGGCGCGGTGGCTCACGCCTGTAATCCCAGCACTTTGG
    GAGGCCGAGGCGGGCGGATCACCTGAGGTCAGGAGTTCGAGA
    CCAGCCTGGCCAACATGGTGAAACCCCGTCTCTACTAAAAAT
    ACAAAAATTAGCCGGGCGTGGTGGCGCGCGCCTGTAATCCCA
    GCTACTCGGGAGGCTGAGGCAGGAGAATCGCTTGAACCCGGG
    AGGCGGAGGTTGCAGTGAGCCGAGATCGCGCCACTGCACTCC
    AGCCTGGGCGACAGAGCGAGACTCCGTCTCAAAAA
"""
    .trimIndent()
    .replace("\n", "")
    .toByteArray()

fun makeCumulative(xs: Array<Pair<Char, Double>>) =
    mutableListOf<Pair<Byte, Double>>().apply {
        var cp = 0.0
        for ((c, p) in xs) {
            cp += p
            add(c.toByte() to cp)
        }
        reverse()
    }

val homoSapiens =
    makeCumulative(arrayOf(
        'a' to 0.3029549426680,
        'c' to 0.1979883004921,
        'g' to 0.1975473066391,
        't' to 0.3015094502008))

val iub =
    makeCumulative(arrayOf(
        'a' to 0.27, 'c' to 0.12, 'g' to 0.12,
        't' to 0.27, 'B' to 0.02, 'D' to 0.02,
        'H' to 0.02, 'K' to 0.02, 'M' to 0.02,
        'N' to 0.02, 'R' to 0.02, 'S' to 0.02,
        'V' to 0.02, 'W' to 0.02, 'Y' to 0.02))

fun selectRandom(f: List<Pair<Byte, Double>>): Byte {
    fun find(n: Int): Byte = when {
        n == 0 -> f[0].first
        random(1.0) < f[n].second -> f[n].first
        else -> find(n - 1)
    }
    return find(f.size - 1)
}

val width = 60
val buffer = ByteArray(1024) { 0 }
var index = 0

fun flush() {
    System.out.write(buffer, 0, index)
    index = 0
}

fun write(b: Byte) {
    buffer[index] = b
    index += 1
    if (index == buffer.size) flush()
}

fun write(c: Char) = write(c.toByte())

fun randomFasta(desc: String, table: List<Pair<Byte, Double>>, n: Int) {
    desc.toByteArray().forEach(::write)
    for (i in 1..n) {
        write(selectRandom(table))
        if (i % width == 0) write('\n')
    }
    if (n % width != 0) write('\n')
}

fun repeatFasta(desc: String, table: ByteArray, n: Int) {
    desc.toByteArray().forEach(::write)
    for (i in 1..n) {
        write(table[(i - 1) % table.size])
        if (i % width == 0) write('\n')
    }
    if (n % width != 0) write('\n')
}

fun main(args: Array<String>) {
    val n = args.firstOrNull()?.toIntOrNull() ?: 1000
    repeatFasta(">ONE Homo sapiens alu\n", alu, 2 * n)
    randomFasta(">TWO IUB ambiguity codes\n", iub, 3 * n)
    randomFasta(">THREE Homo sapiens frequency\n", homoSapiens, 5 * n)
    flush()
}
``` 
I've replaced the `fold` with a loop plus a couple of mutable variables for two 
reasons: 

* There's no `ImmutableList` type in standard library and passing mutable one
 as `fold`s state would be weird.
* The `apply` method makes dealing with the mutable list quite nice, as a result 
`makeCumulative` reads easily, better than the original one from F# version.

I personally find the Kotlin code a little bit easier to read because the 
types are visible almost everywhere, there's no cryptic pipes (`<|`), `if..else..` reads better than
 `if..then..else..`, `when (..) { }` does the job better than `function...` in case of 
 number of boolean conditions. However, the two versions are quite similar, to be
 honest.

As for performance, this benchmark is largely IO bounded, as I see it (it writes ~250MB of 
data into a file in several seconds), so both versions showed approximately 
same time (~6 seconds).