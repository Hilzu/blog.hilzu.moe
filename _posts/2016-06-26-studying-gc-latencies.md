---
layout: post
title: Studying garbage collection latencies
excerpt: >
  A brief look in the worst case garbage collection latencies in Java Virtual Machine, V8 and GHC.
date: 2016-06-26 13:43:00 # UTC
---

Some time ago I read a great [blog post][original-post] by [Gabriel Scherer][gabriel] about measuring Garbage Collector
latencies in Haskell, OCaml and Racket. The blog post is already an expansion of a [different blog post][ghc-post]
that studies GHC so I thought it would be interesting to expand further to Java Virtual Machine and V8 (VM of Chrome
and Node.js).

The benchmark creates a total of 1 million 1 KiB elements in a map indexed by integers. The newest 200 000 items are
kept in memory and the older ones are removed. It has a large working set and only old objects are released which is
bad for most GC algorithms. They are usually optimized for work loads that generate lot's of short lived objects.
This means that the benchmark should exercise worst case GC latencies.

All benchmarks were run on the latest versions as of writing this blog post:

* Node.js: v6.2.2 (Immutable.js v3.8.1)
* Java: 1.8.0_92
* Scala: 2.11.8 (JDK 1.8.0_92)
* Haskell: GHC 7.10.3

The code can be found from my [GitHub repo][my-code]. The `build-all.sh` script is used to build all benchmarks and
the `benchmark.py` script runs them, calculates statistics and generates plots. It runs all the benchmarks
five times in a random order and aggregates the results. The numbers below are averages from these runs. You can find
the plots and table with all results and the end of this post.

## Haskell

As a baseline let's take the Haskell benchmark from the original post and see some results on my machine.

```haskell
module Main (main) where

import qualified Control.Exception as Exception
import qualified Control.Monad as Monad
import qualified Data.ByteString as ByteString
import qualified Data.Map.Strict as Map

type Msg = ByteString.ByteString

type Chan = Map.Map Int Msg

windowSize = 200000
msgCount = 1000000

message :: Int -> Msg
message n = ByteString.replicate 1024 (fromIntegral n)

pushMsg :: Chan -> Int -> IO Chan
pushMsg chan highId =
  Exception.evaluate $
    let lowId = highId - windowSize in
    let inserted = Map.insert highId (message highId) chan in
    if lowId < 0 then inserted
    else Map.delete lowId inserted

main :: IO ()
main = Monad.foldM_ pushMsg Map.empty [0..msgCount]
```

    $ ./src/haskell/Main +RTS -S

The maximum pause time was 50ms with a median of 26ms.
The pause times are quite high (deemed excessive in the original post).
The GHC GC is optimized for throughput and not latency so these numbers are not that surprising.
Now that we have our baseline let's move on to the JVM!

## Java Virtual Machine

To test JVM GC I ported the benchmark to plain Java and ran it with a 1 GiB heap that fits our working set comfortably.

```java
import java.util.Arrays;
import java.util.HashMap;

public class Main {

    private static final int windowSize = 200_000;
    private static final int msgCount = 1_000_000;
    private static final int msgSize = 1024;

    private static byte[] createMessage(final int n) {
        final byte[] msg = new byte[msgSize];
        Arrays.fill(msg, (byte) n);
        return msg;
    }

    private static void pushMessage(final HashMap<Integer, byte[]> map, final int id) {
        final int lowId = id - windowSize;
        map.put(id, createMessage(id));
        if (lowId >= 0) {
            map.remove(lowId);
        }
    }

    public static void main(String[] args) {
        final HashMap<Integer, byte[]> map = new HashMap<>();
        for (int i = 0; i < msgCount; i++) {
            pushMessage(map, i);
        }
    }
}
```

    $ java -Xmx1G -verbosegc -cp src/java Main

With this the maximum pause time was 126ms and the median 57ms. This is even worse than with Haskell! The default JVM
GC is also optimized for throughput and not latency. This is evident it the run time (1.2s) which is almost the same as
Haskell's even while JVM uses a lot more time paused (704ms vs. 235ms).

Unlike GHC, JVM has several collection algorithms to choose from. One of the most interesting ones is the
[Garbage-First (G1) collector][g1-collector] that is optimized for large heap sizes and predictable pause times.
It can be enabled using the `-XX:+UseG1GC` flag. The pause time targets of G1 can be adjusted to fit the needs of the
program. 50 ms was considered excessive in the original blog post so let's set that as the maximum target.

    $ java -Xmx1G -XX:+UseG1GC -XX:MaxGCPauseMillis=50 -verbosegc -cp src/java Main

The median pause time dropped to 15 ms while the maximum was still quite high: 84ms. The maximum target is a soft one
so the collector doesn't guarantee that it's met. There is just a couple of outliers in the results with pause times
this high. See the box plots and the end of this post for more details.

Now this benchmark isn't completely fair. The map in the Haskell version is immutable while Java's HashMap is mutable.
Modifying immutable data structures usually happens by creating a new structure with the desired modifications. Behind
the scenes most of the structure is usually shared by the original and modified structures but it still causes more
garbage to be generated compared to modifying them inline.

I figured the easiest way to create a version of the benchmark with immutable data structures was to use Scala. It's a
functional programming language that runs on the JVM and ships with them built-in. I ran this
Scala program with the same 1GiB heap as the Java one.

```scala
object Main {
  val windowSize = 200000
  val msgCount = 1000000
  val msgSize = 1024

  def createMessage(n: Int): Array[Byte] = {
    Array.fill(msgSize){ n.toByte }
  }

  def pushMessage(map: Map[Int, Array[Byte]], id: Int): Map[Int, Array[Byte]] = {
    val lowId = id - windowSize
    val inserted = map + (id -> createMessage(id))
    if (lowId >= 0) inserted - lowId else inserted
  }

  def main(args: Array[String]): Unit = {
    val map = Map[Int, Array[Byte]]()
    (0 until msgCount).foldLeft(map) { (m, i) => pushMessage(m, i) }
  }
}
```

    $ scala -cp src/scala -J-Xmx1G -J-verbosegc Main

Collections were shorter but more frequent when using Scala. The median pause time with Scala was 18.2ms and the
maximum was 129.4ms. The more frequent collections are explained by the larger amount of garbage generated by Scala.
The load generated by Scala seems to fit the JVM default GC better than the Java one.

## V8

Next up is the V8 JavaScript engine used by Chrome and Node.js.

```javascript
"use strict";

const windowSize = 200000
const msgCount = 1000000
const msgSize = 1024

function createMessage (n) {
  return Buffer.alloc(msgSize, n % 256)
}

function pushMessage (map, id) {
  const lowId = id - windowSize
  map[id] = createMessage(id)
  if (lowId >= 0) {
    delete map[lowId]
  }
}

const map = {}
for (let i = 0; i < msgCount; i++) {
  pushMessage(map, i)
}
```

    $ time node --trace-gc src/node/main.js
          359.59 real       356.27 user         3.42 sys

Woah! That is really slow! More than 100 times slower than Java or Haskell. Something is
really wrong in the code. Objects are commonly used as maps in JavaScript because there hasn't
been any other built-in option. Most JS VMs toggle objects to a hashmap mode when objects are used like maps. For some
reason it seem that the mode isn't being triggered here. Forcing the keys to be strings and values JS native
[Int8Arrays][int8array] instead of Node.js Buffers didn't help. V8 might just not be optimized for this
specific use case yet.

Luckily the [ES2015 standard][es2015] (formerly known as ES6 or harmony) includes a native [Map][map] type that makes
the debugging above unnecessary. Node.js 6 supports it so let's take it for a spin.

```javascript
function pushMessage (map, id) {
  const lowId = id - windowSize
  map.set(id, createMessage(id))
  if (lowId >= 0) {
    map.delete(lowId)
  }
}

const map = new Map()
for (let i = 0; i < msgCount; i++) {
  pushMessage(map, i)
}
```

    $ time node --trace-gc src/node/main.js
            4.86 real         3.39 user         1.62 sys

Much better. Seems like objects as maps is a lot costlier than I thought and really should be avoided if possible.

V8 GC pauses seem to be all over the place. The median pause was 64ms and the maximum a whopping 52 ms! The box plot
really shows how uneven the times are.

The same unfairness as with the Java benchmark exists here. Map is mutable. One of the most popular implementation
of immutable data structures for JavaScript is [Immutable.js][immutable-js] so let's port our benchmark to use it.

```javascript
"use strict";

const Immutable = require('immutable')

const windowSize = 200000
const msgCount = 1000000
const msgSize = 1024

function createMessage (n) {
  return Buffer.alloc(msgSize, n % 256)
}

function pushMessage (map, id) {
  const lowId = id - windowSize
  const inserted = map.set(id, createMessage(id))
  return lowId >= 0
    ? inserted.delete(lowId)
    : inserted
}

Immutable.Range(0, msgCount)
  .reduce(
    (map, i) => pushMessage(map, i),
    Immutable.Map())
```

The same thing happens as with Scala. The pauses are more frequent, shorter and more predictable. Median pause time
drops to 14ms and maximum to 157ms.

## Results and graphs

The benchmarks we're run for 5 times and the following values are aggregates from those runs.

**GC latencies (milliseconds)**

|                        | Java  | Java G1 | Scala  | Node   | Node Imm | Haskell
| ---                    | ---:  | ---:    | ---:   | ---:   |     ---: | ---:
| **Min**                |  23.8 |     0.8 |    2.7 |    0.5 |      0.4 |   1.0
| **Median**             |  56.5 |    15.0 |   18.2 |   63.6 |     13.8 |  26.0
| **Average**            |  58.7 |    15.0 |   24.1 |  111.7 |     17.2 |  20.6
| **Max**                | 126.1 |    83.8 |  129.4 |  529.1 |    157.3 |  50.0
| **Avg total pause**    | 704.2 |   380.7 | 1119.3 | 3240.3 |   3574.9 | 235.4
| **Avg pauses**         |    12 |    25.4 |   46.4 |     29 |      208 |  11.4
| **Avg clock time (s)** |   1.2 |     1.0 |    2.6 |    5.2 |      9.3 |   1.3

On JVM and V8 immutability seems to cause more but shorter GCs probably due to larger amount of garbage generated by
modifying persistent data structures.

<div class="gc-figures-container">
    <figure>
        <a href="/assets/gc-latencies.svg">
            <img src="/assets/gc-latencies.svg" alt="GC latencies">
        </a>
        <figcaption>GC latencies with V8 outliers visible</figcaption>
    </figure>

    <figure>
        <a href="/assets/gc-latencies-zoomed-in.svg">
            <img src="/assets/gc-latencies-zoomed-in.svg" alt="GC latencies zoomed in">
        </a>
        <figcaption>Latencies zoomed in</figcaption>
    </figure>
</div>

V8 has a lot of outliers. The large amount of outliers in the immutable Node.js benchmark is explained by the amount
of pauses that the GC does (235 on average per run). The couple of outliers with Java G1 are also visible here.
All other collections with it we're clearly under our maximum target of 50ms.

That concludes the look into JVM and V8 garbage collection latencies. Let me know in the comments below what you
thought. You can also open a pull request or issue in the GitHub repository if you found some problems with the
code.

[gabriel]: http://gallium.inria.fr/~scherer/
[original-post]: http://prl.ccs.neu.edu/blog/2016/05/24/measuring-gc-latencies-in-haskell-ocaml-racket/
[original-code]: https://gitlab.com/gasche/gc-latency-experiment
[ghc-post]: https://blog.pusher.com/latency-working-set-ghc-gc-pick-two/
[my-code]: https://github.com/Hilzu/gc-latency-experiment
[buffer]: https://nodejs.org/api/buffer.html
[map]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map
[immutable-js]: https://facebook.github.io/immutable-js/
[int8array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int8Array
[es2015]: http://www.ecma-international.org/ecma-262/6.0/
[g1-collector]: http://www.oracle.com/technetwork/tutorials/tutorials-1876574.html
