# The Stream API

The `java.util.stream` API (Java 8) is a declarative pipeline for processing sequences of elements — `source.filter(...).map(...).collect(...)` — that lets you express *what* transformation you want and leaves the *how* (iteration, laziness, parallelism) to the library. It is Java's functional data-processing layer, and despite the shared word it has nothing to do with the I/O streams of [[io-streams-and-readers]].

## Laziness: Intermediate Versus Terminal

A stream pipeline has three parts: a **source** (collection, array, `Stream.iterate`/`generate`, lines of a file), zero or more **intermediate operations**, and exactly one **terminal operation**. The defining property is **laziness**: intermediate ops (`map`, `filter`, `sorted`, `distinct`, `limit`) only *describe* work and return a new stream; nothing executes until the terminal op (`collect`, `reduce`, `forEach`, `count`, `findFirst`) is invoked. When it fires, elements are pulled through the whole pipeline **one at a time**, not stage-by-stage over the full dataset — so a `filter` then a `map` make a single pass, and no intermediate collections are materialized.

Laziness enables two things eager iteration cannot: **fusion** of operations into one traversal (see [[07-performance-engineering/lazy-evaluation|Lazy Evaluation]]) and **short-circuiting** — `findFirst`, `anyMatch`, `limit`, `takeWhile` can stop early, which is what makes an infinite source like `Stream.iterate(1, n -> n + 1)` usable at all.

Streams are also **single-use** (one terminal op per stream) and **non-interfering**: they do not mutate the source, and lambdas should be **stateless** and side-effect-free. A stream is not a data structure — it stores nothing and owns no elements.

## Spliterators: The Engine

Under every stream is a `Spliterator` — a traversal abstraction combining `tryAdvance` (consume one element) with `trySplit` (hand off a chunk for parallel work). Its **characteristics** — `SIZED`, `ORDERED`, `SORTED`, `DISTINCT`, `IMMUTABLE`, `CONCURRENT` — are metadata the pipeline exploits: a `SIZED` source lets `distinct` presize buffers, an already-`SORTED` source skips a re-sort. The quality of a source's spliterator, especially how evenly `trySplit` divides, is what determines whether parallelism pays off. Arrays and `ArrayList` split cleanly by index; `LinkedList` and I/O sources split poorly.

## Collectors And Reduction

Terminal aggregation comes in two flavors. `reduce` is an *immutable* fold requiring an associative operator and an identity. `collect` is a **mutable reduction**: a `Collector` supplies a container, an accumulator, and a combiner (`Collectors.toList`, `groupingBy`, `partitioningBy`, `joining`, `toMap`). `groupingBy` with a downstream collector — `groupingBy(Order::customer, summingInt(Order::total))` — is where the API's expressiveness shows, replacing loops of nested maps with one declarative statement. Because primitives box expensively inside a `Stream<Integer>`, the specialized `IntStream`/`LongStream`/`DoubleStream` exist to avoid that overhead.

## Parallel Streams: Handle With Care

`.parallel()` (or `Collection.parallelStream()`) splits the source via `trySplit` and runs the pipeline on the **common `ForkJoinPool`** (see [[03-computer-systems/concurrency-and-parallelism/thread-pools|Thread Pools]]). It is a one-word change with sharp edges:

- The common pool is **shared process-wide** and sized to `cores − 1`. A blocking or long-running task inside a parallel stream starves *every other* parallel stream and `CompletableFuture` in the JVM.
- It pays off only with a **large N**, a **cheap-to-split** source (arrays, not `LinkedList`), and a **CPU-bound, associative** operation. For I/O-bound work or small datasets, the split/merge overhead loses outright.
- Reducers must be **associative** or results are non-deterministic; ordered pipelines pay extra to preserve encounter order (`forEachOrdered`), and any shared mutable state is a race.

The honest senior default is *sequential unless a benchmark proves parallel wins* — and even then, prefer an explicit executor over the shared common pool.

## Philosophy

Streams brought a **functional, declarative** idiom to an imperative language (see [[01-programming-foundations/paradigms/functional-programming|Functional Programming]] and [[01-programming-foundations/paradigms/higher-order-functions|Higher-Order Functions]]): you compose higher-order operations over immutable views instead of writing index loops. The payoff is readability and the *option* of parallelism from the same code; the price is a real learning curve, harder debugging through lazy pipelines, and stack traces that obscure the actual line. Used for genuine data-transformation pipelines it is a clear win; contorted into a replacement for a simple `for` loop it costs more than it gives.

## See Also
- [[io-streams-and-readers]]
- [[01-programming-foundations/paradigms/functional-programming|Functional Programming]]
- [[01-programming-foundations/paradigms/higher-order-functions|Higher-Order Functions]]
- [[07-performance-engineering/lazy-evaluation|Lazy Evaluation]]
- [[03-computer-systems/concurrency-and-parallelism/thread-pools|Thread Pools]]
- [[fork-join-and-parallel-streams]] — how parallel streams execute
