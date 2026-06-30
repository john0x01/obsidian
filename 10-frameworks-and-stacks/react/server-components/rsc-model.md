# The RSC Model

React Server Components (RSC) are components that run **only on the server**, during the request (or build), and render to a serialized description of UI — the **RSC payload**, sometimes called the Flight format — rather than to HTML or to a live DOM. The shift is fundamental: a Server Component's *code* never ships to the browser, executes once on the server, and is reduced to data. This collapses the old "fetch in `useEffect`, ship the whole component, hydrate" pipeline into "run on the server with direct data access, send only the result."

## The payload is an element tree, not HTML

This is the most misunderstood point. The RSC payload is **not** HTML and **not** a JSON blob of props you reassemble by hand. It is a serialized React element tree — the same conceptual thing `<App/>` produces in memory, but encoded as a streamable format that the React client runtime knows how to deserialize back into elements and reconcile.

```
Conceptual payload (simplified):
  ["$","div",null,{"className":"card","children":[
     "Hello",
     ["$","@1",null,{"count":3}]        // @1 = reference to a Client Component
  ]}]
  M1: {id:"./Counter.js", chunks:[...], name:"Counter"}  // module ref for @1
```

Server Components are rendered to inline elements (text, host tags, resolved data). Client Components appear as **module references** — a pointer to a bundle chunk plus their serialized props — *holes* the server leaves for the client to fill by importing and rendering the real component. Because it's an element tree and not HTML, RSC composes with reconciliation: a re-fetched payload can be diffed against the current UI and applied without blowing away client state. That is why RSC can power navigations and refreshes, not just an initial paint.

## The server/client component graph

A React app under RSC is a single tree split into two execution domains:

```
  ServerLayout            (server: runs once, becomes payload data)
  ├─ ServerSidebar        (server)
  ├─ 'use client' Tabs    (client: shipped as JS, hydrates)
  │    └─ ServerPanel     (server, passed as children → rendered server-side)
  └─ ServerList           (server)
```

Server Components are the default in an RSC environment. Crossing into client land requires a `'use client'` boundary. Crucially, **a Client Component cannot import a Server Component** — once you're in client code, you're past the server's execution window — but a Server Component can *render* a Client Component, and can pass Server-rendered JSX into it as `children` or props. The graph is therefore a server shell with client islands, and the islands can themselves contain server-rendered slots passed through from above.

## Why RSC exists

- **Data access without waterfalls.** A Server Component can `await db.query(...)` or hit an internal service directly — no API endpoint, no client fetch, no round trip. Data fetching moves *into* the component, server-side, where it's close to the source. `async` Server Components and the `use` hook make this first-class.
- **Zero client bundle for server code.** Markdown renderers, date libraries, syntax highlighters, ORM types — none of it ships. The component's dependencies stay on the server; only the rendered result and any client islands' JS reach the browser. Bundle size decouples from app complexity for the server portion.
- **Security and secrets.** API keys, tokens, and direct datastore access live in code the client never receives. The serialization boundary is also a *trust* boundary.

## Mental model

Think of RSC as **`UI = f(state)` split across the network**, where the server computes the static-for-this-request portion and streams it, leaving typed holes for interactivity. Server Components are not "SSR with extra steps." Classic SSR renders your *client* components to an HTML string on the server and then ships and hydrates all of that same JS. RSC renders *server-only* components to a data payload and **never ships their code at all** — only the client islands hydrate. RSC and SSR are complementary: a request can both stream HTML (for first paint) and stream the RSC payload (the source of truth the client reconciles against).

The payload is also **streamable and suspense-aware**: the server can flush the shell immediately and push in slower data chunks as `Suspense` boundaries resolve, so a slow query doesn't block the whole response. This is the same `<Suspense>` you use on the client, now coordinating server streaming.

## Senior pitfalls

- **"RSC means it renders on the server" — incomplete.** Server Components render *only* on the server, with no re-render, no state, no effects, no event handlers. They are one-shot. Interactivity is exclusively a client concern.
- **Confusing the payload with HTML or with props JSON.** It is a serialized element tree with module references; that's what enables diffing and state-preserving navigation.
- **Assuming everything must be a Server Component.** The art is drawing the boundary: keep data and heavy deps on the server, push the smallest possible interactive leaves to the client.
- **Forgetting RSC is a *protocol*, not a framework feature.** React defines the model and payload; a framework/bundler (Next.js App Router, etc.) wires up the server runtime, routing, and bundler integration that make it usable.

## See Also

- [[server-vs-client-components]]
- [[actions-and-forms]]
- [[state-architecture]]
- [[declarative-ui]]
- [[10-frameworks-and-stacks/next-js/rendering-internals/rsc-and-flight|RSC Wire Format (Flight)]] — how Next serializes RSC
