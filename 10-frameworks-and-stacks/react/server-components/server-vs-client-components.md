# Server vs Client Components

The `'use client'` directive draws the single most consequential line in an RSC application: it marks the boundary where execution crosses from the server into the browser. Everything above the line is a Server Component — runs once on the server, ships no JS, has no state or effects. Everything from a `'use client'` module downward is a Client Component — bundled, shipped, and hydrated, with the full interactive React you already know. Understanding *what may cross that line* is what separates working architectures from cryptic serialization errors.

## The directive marks an entry point, not a file's nature

`'use client'` at the top of a module declares that module — and everything it imports — as the **client boundary**. It does not mean "this one component is client"; it means "from here down the import graph, we are in client land." A module without the directive, in an RSC environment, is a Server Component by default. The directive is a *doorway* the bundler uses to decide where to split.

```jsx
'use client';            // entry point: this subtree ships to the browser
import { useState } from 'react';
export function Counter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```

Because it's a doorway, you place it on the *leaf* islands that genuinely need interactivity, not on a top-level layout — pushing it down keeps the server portion (and the bundle savings) as large as possible.

## What runs where

| | Server Component | Client Component |
|---|---|---|
| Executes | Once, on server, per request/build | Server (SSR pass) **and** browser (hydration + updates) |
| Ships JS | No | Yes |
| `useState`/`useEffect`/refs | No | Yes |
| Event handlers (`onClick`) | No | Yes |
| `async`/`await`, direct DB/fs | Yes | No |
| Can import Server Components | Yes | **No** |

A Client Component still renders on the server during SSR for first paint — `'use client'` does not mean "client-only," it means "also runs on the client." The asymmetry is that Server Components *only* run on the server.

## Serialization constraints across the boundary

When a Server Component renders a Client Component, the props it passes must cross the network as part of the RSC payload, so they must be **serializable**. This is the rule that trips people up:

- **Allowed:** primitives, plain objects/arrays of allowed values, `Date`, `Map`/`Set`, typed arrays, JSX/React elements, Promises, and a special case — **Server Functions** (functions marked `'use server'`), which serialize to a reference, not their body.
- **Forbidden:** arbitrary functions and closures, class instances, `Symbol`s (non-registered), and anything carrying behavior the client can't reconstruct.

```jsx
// Server Component
<Chart data={rows} formatLabel={(x) => x.toFixed(2)} />  // ❌ formatLabel: closure can't cross
<Chart data={rows} onSave={saveAction} />                // ✅ saveAction is a 'use server' fn → reference
```

The reason is structural: a closure captures live server-scope variables that have no meaning in the browser. The boundary is a *serialization* boundary, and serialization can't capture behavior. The fix is to keep the function on the side that uses it — define event handlers *inside* the Client Component, or pass a Server Function reference.

## Composition: passing client components and server children through

The escape hatch from "client can't import server" is **composition via children/props**. A Server Component renders the slow, data-heavy, server-only content and passes it *as a prop* into a Client Component:

```jsx
// ServerPage.jsx (server)
<ClientTabs>          {/* client island */}
  <ServerHeavyPanel/> {/* server-rendered, passed as children */}
</ClientTabs>
```

`ClientTabs` never imports `ServerHeavyPanel`. It receives already-rendered element output (a hole in the payload) and decides *where* to place it — it cannot inspect or re-render it. This lets an interactive client shell wrap server-rendered content without dragging that content's code or dependencies into the client bundle. It's the canonical pattern for "interactive layout, server-fetched contents."

## Interleaving and the boundary's depth

Server and client layers can nest arbitrarily deep — server → client → (server passed as children) → client. But the import direction is one-way: server may import client; client may not import server, only *receive* server output through props. A mental shortcut: data and secrets flow downward through the server tree; interactivity lives in client leaves; the two are stitched by passing rendered server output through client components as children.

## Senior pitfalls

- **Putting `'use client'` too high.** One directive on a root layout pulls the entire subtree into the client bundle, erasing RSC's benefits. Push it to the smallest interactive unit.
- **Trying to pass event handlers from server to client.** They're closures — not serializable. Define them client-side.
- **Expecting a Server Component to re-render on interaction.** It can't; it has no state. Interactivity requires a client island, optionally re-fetching a fresh server payload via a navigation/refresh.
- **Assuming `'use client'` makes a component browser-only.** It still SSRs. "Client-only" rendering needs an explicit effect/dynamic guard.

## See Also

- [[rsc-model]]
- [[actions-and-forms]]
- [[error-boundaries]]
- [[declarative-ui]]
- [[10-frameworks-and-stacks/next-js/app-router/server-and-client-components|Server & Client Components (Next)]] — Next's boundary specifics
