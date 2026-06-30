# The Client/Server Boundary

`'use client'` is not a runtime switch — it's a **bundler directive that cuts the module graph**. It marks the point where modules transition from server-only to client-shipped. Everything above the cut renders only on the server and never reaches the browser as code; everything at and below the cut is compiled into the client bundle. Internalizing that the boundary is a *graph partition*, not a per-component flag, resolves most "why is this running where I didn't expect" confusion.

## How the directive splits the graph

The compiler walks the module dependency graph. A module with `'use client'` at the top becomes an **entry point into the client bundle**: that module *and everything it imports* is bundled for the browser. Crucially, the directive is **transitive downward but bounded by re-entry**:

- A `'use client'` module and its imports ship to the client.
- A Server Component is the default (no directive needed in the App Router). It runs only on the server.
- You can render a Server Component *inside* a Client Component **only by passing it as a prop/children** (it's already-rendered output in the RSC payload), not by importing it — importing a Server Component into a client module would pull it into the client graph and effectively make it client.

So the boundary is the set of edges where a server module references a client module. Each such edge becomes a **client reference** in the RSC payload.

```
server graph                 │  client graph (shipped JS)
  page.tsx (server)          │
   ├─ Header (server)        │
   └─ Cart  'use client' ────┼─▶ Cart.js  + its imports
        └─ <Server/> as prop │     (Server output arrives via payload, not imported)
```

## Client references in the payload

When a server render hits a Client Component, it can't execute it (the client code may use hooks, browser APIs, event handlers). Instead it emits a **module reference**: an identifier `{ chunk, exportName }` that the client runtime resolves to the real component, plus the component's **serialized props**. At hydration (or on navigation), the client loads that chunk and mounts the component with those props. This is the concrete link between the boundary and the wire format described in [[rsc-and-flight]]: every `'use client'` edge is one `$L` reference row.

## What JS actually ships

The browser receives JavaScript for: every `'use client'` module, everything those modules import (including shared utilities and any third-party libs they pull in), and the React/Next client runtime. It does **not** receive: Server Component code, server-only imports (DB clients, secrets, heavy formatting libs used only server-side), or the data-fetching logic that ran on the server. This asymmetry is the headline benefit — a date library or markdown renderer used only in a Server Component adds **zero** bytes to the client. The boundary is therefore a *bundle-size lever*: push interactivity-only code below `'use client'`, keep everything else above it.

A common bloat source: importing a large library into a `'use client'` module "just for one helper" drags the whole library client-side. And a `'use client'` placed too high (e.g. at the layout root) drowns the benefit by pulling the whole subtree into the client graph. Place the boundary as **low and as narrow** as the interactivity requires — "islands," not a client shell.

## Serialization rules and constraints

Because props on a Client Component cross the boundary *inside the RSC payload*, they are subject to Flight serialization. The allowed set is roughly: primitives, plain objects/arrays of serializable values, `Date`, `Map`/`Set`, `BigInt`, `TypedArray`, `Promise` (it streams and resolves on the client), React elements (including already-rendered Server Components passed as children), and **Server Action references** (functions marked `'use server'`). The forbidden set: arbitrary functions, class instances, symbols (except registered ones), and anything with behavior that can't be reconstructed. Practically:

- **You cannot pass an event handler or callback** from a Server Component to a Client Component. Functions don't serialize — except Server Actions, which serialize as an encrypted reference (see [[server-actions]]).
- **Passing children/slots is the escape hatch.** Render Server Components on the server and hand them to a Client Component via `children`; the client just slots the pre-rendered output in.
- **Props are a one-way, render-time snapshot** server→client. There's no live binding back.

## Pitfalls and misconceptions

- **"`'use client'` means it only runs on the client."** No — Client Components still **SSR** on the first request (they produce initial HTML) and then **hydrate**. "Client component" means "in the client bundle and hydratable," not "skips the server."
- **"`'use client'` opts the whole route out of server rendering."** No — it marks a boundary; the server tree above it still renders to RSC and the client subtree still SSRs once.
- **Importing vs. composing.** Importing a Server Component into a Client Component breaks the model; passing it as a prop/children is the supported pattern.
- **Over-marking.** Reflexively topping files with `'use client'` defeats RSC's bundle savings. The default should be server; promote to client only at genuine interactivity leaves.

## See Also

- [[rsc-and-flight]]
- [[router-internals]]
- [[server-actions]]
- [[data-fetching]]
- [[10-frameworks-and-stacks/react/philosophy/composition-model|Composition Model]]
- [[10-frameworks-and-stacks/react/hooks/hooks-internals|Hooks Internals]]
