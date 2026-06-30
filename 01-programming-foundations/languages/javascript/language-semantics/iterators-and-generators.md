# Iterators And Generators

Iteration in JavaScript is a *protocol*, not a built-in loop feature: any object that implements the right method becomes consumable by `for...of`, spread, and destructuring. Generators are the language's ergonomic way to author iterators as pausable functions, and async iterators extend the whole scheme to data that arrives over time. Understanding the protocol explains how lazy, infinite, and streaming sequences all work uniformly.

## The Iteration Protocols

Two cooperating interfaces:

- **Iterable**: an object with a `[Symbol.iterator]()` method returning an iterator. Arrays, strings, `Map`, `Set`, `arguments`, and NodeLists are built-in iterables.
- **Iterator**: an object with a `next()` method returning `{ value, done }`. Optionally `return(value)` (called on early exit — `break`, error, or `return` inside `for...of`) for cleanup, and `throw()`.

```js
const range = {
  from: 1, to: 3,
  [Symbol.iterator](){
    let cur = this.from, last = this.to;
    return { next: () => cur <= last ? {value: cur++, done:false} : {value:undefined, done:true} };
  }
};
[...range];          // [1,2,3]
for (const n of range) {}   // 1,2,3
```

`for...of`, spread `[...x]`, array/parameter destructuring, `Array.from`, `Promise.all`, and `new Map(entries)` all consume the iterable protocol. (`for...in`, by contrast, enumerates *string keys* up the prototype chain — a different, unrelated mechanism.) A clean separation: the *iterable* is the data; the *iterator* is a single cursor over it, so multiple iterations may need fresh iterators.

## Generators As State Machines

A generator function (`function*`) returns a generator object that is *both* an iterator and iterable. Its body does not run on call; it runs incrementally, pausing at each `yield` and resuming on the next `next()`. This is cooperative, single-threaded suspension — the function's entire execution state (local variables, position) is preserved between calls.

```js
function* gen(){
  const x = yield 1;     // pauses here; x receives the value passed to next()
  yield x + 10;
}
const g = gen();
g.next();      // {value:1, done:false}  — runs up to first yield
g.next(5);     // {value:15, done:false} — resumes, x = 5
g.next();      // {value:undefined, done:true}
```

Two directions of communication: `yield` sends values *out*; the argument to `next(v)` sends a value *in*, becoming the result of the paused `yield` expression. The first `next()`'s argument is discarded (nothing is waiting yet). `g.return(v)` forces completion; `g.throw(e)` injects an exception at the pause point, letting a `try/finally` in the body clean up. This bidirectional channel is exactly what made generators the substrate for early coroutine-based async (`co`, redux-saga) before `async/await`.

## yield* — Delegation

`yield*` delegates to another iterable, yielding all its values and transparently forwarding `next()`/`throw()`/`return()` and capturing the delegate's final `return` value as the `yield*` expression's result. It composes generators and flattens recursion cleanly — e.g. a tree traversal that `yield*`s into child traversals.

```js
function* inner(){ yield 'a'; yield 'b'; }
function* outer(){ yield 1; yield* inner(); yield 2; }   // 1, a, b, 2
```

## Async Iterators And for-await-of

For values that arrive asynchronously, the symmetric protocol uses `[Symbol.asyncIterator]()` and a `next()` that returns a **Promise** of `{value, done}`. Async generators (`async function*`) author them: their body may both `await` and `yield`. `for await (const x of source)` awaits each `next()` in turn, sequentially. This is how you consume paginated APIs, Node streams (which implement `Symbol.asyncIterator`), or any push source as a pull sequence with backpressure-friendly, one-at-a-time semantics.

```js
async function* pages(url){
  let next = url;
  while (next){ const r = await fetch(next).then(r=>r.json()); yield r.items; next = r.next; }
}
for await (const items of pages('/api')) process(items);
```

## Lazy Sequences And Infinite Streams

Because nothing is computed until `next()` is pulled, iterators model **lazy** and even **infinite** sequences safely. An infinite generator of naturals is fine as long as the consumer stops pulling:

```js
function* naturals(){ let n=0; while(true) yield n++; }
```

This enables pull-based pipelines — `map`/`filter`/`take` implemented over iterators process one element at a time with O(1) memory regardless of source size, and short-circuit (`break`/`take`) avoids computing the tail. It is the iterator-protocol analog of streams and Haskell-style lazy lists.

## Senior Pitfalls And Mental Model

- Iterators are typically **single-use**: once `done`, they stay done. Spreading or `for...of`-ing a generator twice on the same object yields nothing the second time — re-create it.
- Early exit cleanup depends on `return()` being called; `for...of` does this automatically, manual `next()` loops do not — leak risk with open resources.
- Spreading an infinite iterable (`[...naturals()]`) hangs forever; laziness is only safe with a bounded consumer.
- Async iteration is *sequential* by design; for concurrency, collect promises and `Promise.all` rather than `for await`.

Mental model: an iterable is "ask me for a cursor," an iterator is "a cursor you pump for one value at a time," and a generator is the most natural way to *write* such a cursor — a function you can pause, resume, feed, and abort. Sync vs async is the same shape with a Promise wrapper.

## See Also

- [[symbols]]
- [[scopes-and-closures]]
- [[01-programming-foundations/languages/javascript/asynchrony/promises-internals|Promises Internals]]
- [[01-programming-foundations/paradigms/functional-programming|Functional Programming]]
- [[01-programming-foundations/paradigms/reactive-programming|Reactive Programming]]
