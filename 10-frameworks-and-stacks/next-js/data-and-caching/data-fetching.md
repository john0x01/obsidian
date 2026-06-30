# Data Fetching In Server Components

In the App Router, data fetching is just code that runs on the server during render. A Server Component is an `async` function; you `await` your data inline and return JSX. There is no `getServerSideProps`, no separate data-loading lifecycle, no serialization contract you author by hand. This collapses two formerly distinct concerns — "fetch the data" and "render the markup" — into a single function, and most of the senior-level subtlety lives in the consequences of that collapse.

## The mental model: render *is* the fetch

Because components are async and rendering happens server-side, the component tree *is* your data dependency graph. React renders the tree, suspends on any component that is awaiting, and resumes when the promise resolves. The shape of your fetches therefore mirrors the shape of your JSX. This is powerful (data lives next to where it's used, no prop-drilling of loaders) but it means the *structure* of your components silently dictates whether fetches run in parallel or in series.

## Sequential vs parallel, and the waterfall trap

A request waterfall is when fetch B cannot start until fetch A resolves, even though B does not depend on A's result. Two flavors exist and they are easy to conflate:

- **Intra-component waterfall** — sequential `await`s in one function:

```ts
const user = await getUser(id);      // blocks
const posts = await getPosts(id);    // only starts after user resolves
```

If `getPosts` doesn't need `user`, this is a pure latency tax. Fix by kicking off both before awaiting:

```ts
const userP = getUser(id);
const postsP = getPosts(id);
const [user, posts] = await Promise.all([userP, postsP]);
```

- **Inter-component (tree) waterfall** — a parent component `await`s before rendering a child that also fetches. The child's fetch can't begin until the parent's resolves *and the parent finishes rendering*. This is the more insidious one because it's invisible: nothing in the code looks sequential, but the parent/child nesting serializes the network.

The cure for tree waterfalls is to **move fetches up to a common ancestor and fan them out**, or to use Suspense boundaries so siblings stream independently. Sibling Server Components that each fetch will run their fetches concurrently — React does not block one sibling on another. The waterfall only appears across parent→child edges where the parent awaits before rendering the child.

## Preloading: breaking the parent→child edge

When a child genuinely needs the parent's data but you don't want to serialize them, the **preload pattern** starts the request eagerly without awaiting:

```ts
// item.ts
export const preload = (id: string) => { void getItem(id); };
export const getItem = cache(async (id: string) => { /* fetch */ });
```

The parent calls `preload(id)` (fire-and-forget) before doing its own work; by the time the child calls `getItem(id)`, the request is already in flight. This leans on **Request Memoization** (see [[caching-layers]]) — `fetch` and React's `cache()` dedupe identical calls within a single render pass, so `preload` and the later real read coalesce into one network request. Without memoization, preload would just double the load.

## Where data fetching should live

The senior heuristic is **fetch at the point of use, dedupe at the boundary**. Don't centralize everything in the page and drill props; co-locate each component's fetch with the component, and rely on memoization to prevent the duplicate requests that co-location would otherwise cause. Wrap shared data accessors in `cache()` (for non-`fetch` sources like an ORM) so two components asking for the same record hit the database once. Put a Suspense boundary around any subtree whose data is slow or independent, so the shell streams immediately and the slow part fills in — this also converts an accidental waterfall into visible, parallel streaming.

## Pitfalls and misconceptions

- **`Promise.all` does not parallelize HTTP across the wire by magic** — it parallelizes *awaiting*. The requests still must have been *started*. `await a; await b;` is sequential; `Promise.all([a, b])` where `a`/`b` are already-invoked promises is parallel. Starting the promise is what matters, not the `await`.
- **`fetch` in Server Components is patched, not native.** Next wraps it for memoization and the Data Cache. A non-`fetch` data source (DB driver, SDK) gets *no* automatic dedupe — you must opt in with `cache()`.
- **Don't fetch in a Client Component just to render server data.** That ships the data and the fetching logic to the browser, creates a client-side waterfall (component mounts → effect runs → request), and loses the streaming benefit. Fetch on the server, pass plain serializable props down.
- **`loading.js` is a Suspense boundary in disguise.** Adding it doesn't speed anything up; it lets the route shell render while the page awaits, which is a UX trade (instant skeleton vs. slightly later complete page), not a performance free-lunch.

## See Also

- [[caching-layers]]
- [[revalidation]]
- [[rsc-and-flight]]
- [[10-frameworks-and-stacks/react/hooks/hooks-internals|Hooks Internals]]
- [[11-applied/web-platform/critical-rendering-path|Critical Rendering Path]]
