# The RSC Wire Format (Flight)

When a Server Component renders, React does not produce HTML for the client to diff — it produces a serialized description of the rendered tree called the **RSC payload**, transmitted in a format informally known as **Flight**. This payload is the bridge between server rendering and the client React runtime: it is *not* HTML and *not* JavaScript, but a streamable, line-oriented serialization of "what the UI is" that the browser's React reconstructs into a real element tree. Understanding its shape demystifies almost everything about App Router behavior.

## What the payload actually contains

The RSC payload is a description of the rendered output of Server Components, with three kinds of contents woven together:

- **Rendered host elements and text** — the already-evaluated output of Server Components (e.g. the resolved `<div>`, `<p>`, text). Server Component *code* never reaches the client; only its *output* does.
- **References to Client Components** — Server Components can't run on the client, but Client Components must. Where a Client Component appears, the payload contains a **module reference**: a pointer (chunk id + export name) to the client JS bundle that holds that component, plus its **serialized props**. It's a placeholder saying "here goes the client component identified by this reference, with these props."
- **Promises / suspended holes** — for streaming, unresolved subtrees are emitted as references to chunks that arrive later.

It's helpful to think of it as a tree of React elements where leaves are either concrete host nodes or "load-and-mount-this-client-module-here" markers.

## The format up close

Flight is a row-based, reference-counted serialization. Conceptually each line is `id:payload`, where ids let later rows refer back to earlier ones, enabling deduplication and out-of-order streaming:

```
// illustrative, not exact byte syntax
1:I["./Button.js",["chunk-abc"],"default"]   // a client module reference
0:["$","div",null,{"children":[["$","h1",null,{"children":"Hi"}],
   ["$","$L1",null,{"label":"Go"}]]}]          // element tree; $L1 -> row 1
```

`["$","div",null,{props}]` is the serialized form of a React element (the `$` tag, type, key, props). `$L1` is a *lazy reference* to the module declared on row `1` — that's the client component placeholder. Rows can stream independently: the server flushes row `0` (the shell) immediately and flushes a suspended subtree's row later when its data resolves. This row/reference design is what makes the payload both **streamable** and **deduplicated** — a value used twice is serialized once and referenced.

## Streaming and reconstruction

The server streams the payload as a chunked response while rendering. The client React runtime parses it incrementally:

1. As rows arrive, the runtime builds React elements. Host elements become normal elements; `$L` references trigger loading the named client chunk.
2. Client component references resolve to the actual component functions from the bundle; their serialized props are deserialized and passed in.
3. Suspended holes show their fallback until the corresponding later row streams in, at which point that subtree is filled — this is RSC streaming and it's the same mechanism `<Suspense>` and `loading.js` ride on.

On the *initial* page load there's a subtlety: the server also produces **HTML** (via SSR) so the user sees content immediately, *and* it inlines the RSC payload (as `self.__next_f` script chunks) so the client can **hydrate** and continue. On *subsequent* navigations there's no HTML — the client fetches *only* the RSC payload for the new segments and reconciles it into the existing tree (see [[router-internals]]). This is why navigation feels instant and preserves client state: you're patching a tree with a new serialized description, not replacing a document.

## Why a tree, not HTML

The reason RSC ships a serialized *React tree* rather than HTML is that React must be able to **reconcile** it against the client's existing virtual tree. HTML would force a destroy-and-rebuild that wipes client component state (form inputs, scroll, focus). A serialized element tree lets React diff old vs new at the element level and keep untouched Client Components — and their state — alive across navigations. The Flight format is the serialization that makes server-driven UI *reconcilable* on the client.

## Senior pitfalls and misconceptions

- **"The RSC payload is HTML."** It isn't; HTML is a *separate* SSR output produced only on first load. Confusing the two leads to wrong mental models of navigation.
- **"Server Component code is sent to the client."** Never. Only its rendered output plus client *references* travel. This is the actual source of the bundle-size win — server libraries stay server-side.
- **Props at the boundary must be serializable.** Because Client Component props live *in the payload*, they go through Flight serialization. Functions (other than Server Actions, which serialize as references), class instances, and other non-serializable values can't cross — see [[client-server-boundary]].
- **Streaming order ≠ source order.** Rows arrive as data resolves; don't assume top-to-bottom. This is a feature (fast shell, late slow parts), not a bug.

## See Also

- [[client-server-boundary]]
- [[router-internals]]
- [[data-fetching]]
- [[server-actions]]
- [[10-frameworks-and-stacks/react/fiber/fiber-architecture|Fiber Architecture]]
- [[10-frameworks-and-stacks/react/philosophy/composition-model|Composition Model]]
- [[10-frameworks-and-stacks/react/server-components/rsc-model|RSC Model (React)]] — the underlying React model
