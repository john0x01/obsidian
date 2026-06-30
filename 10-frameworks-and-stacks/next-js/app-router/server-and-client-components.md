# Server And Client Components

The `'use client'` directive is the most consequential single line in an App Router codebase, and the most misunderstood. It is not "make this run in the browser." It is a **boundary marker that splits the module graph**: everything reachable from a `'use client'` module becomes part of the client bundle, while everything above the boundary renders only on the server. Mastery here is mostly about understanding *where the boundary falls* and *what crosses it*.

## Where Code Actually Runs

- **Server Components** (the default in `app/`) execute on the server only — at build time for static segments, at request time for dynamic ones. Their *code is never sent to the browser*. They can be `async`, `await` data, read env secrets, and import server-only libraries.
- **Client Components** (a file with `'use client'` at the top) are still **rendered on the server first** (SSR produces their initial HTML), *and* their JavaScript is shipped and **hydrated** in the browser, where they then re-render in response to state and events. So "client component" means "ships to and runs in the browser," not "only runs in the browser."

The directive is needed only **once, at the boundary**. You do not put `'use client'` in every leaf — marking a module makes that module *and every module it imports* part of the client graph automatically.

## The Module-Graph Split

```
Server graph (default)
  app/page.tsx ........... Server Component
    imports <Chart/> ← 'use client'  ──┐
                                       │  boundary
  Client graph ────────────────────────┘
    Chart.tsx (and everything Chart imports)
    ships to the browser
```

Key rule: **once you cross into the client graph, you cannot import a Server Component back in the normal way.** A `'use client'` module that does `import ServerThing from './server-thing'` pulls that module into the client bundle — it stops being a Server Component. This is why naive "just add use client to fix the error" cascades the boundary upward and bloats the bundle.

## Passing Server Data to Client Components

Two mechanisms, and the distinction is everything:

1. **Props.** A Server Component can render a Client Component and pass it props. Those props are **serialized** into the RSC payload and rehydrated on the client. Therefore props must be serializable: plain objects, arrays, strings, numbers, `Date`, `Map`/`Set`, and — uniquely — **Server Functions** (`'use server'` actions) and **JSX/Server Components passed as `children`**. You **cannot** pass functions (other than server actions), class instances, or symbols.

2. **`children` (the composition pattern).** A Server Component can be passed *as the `children` prop* of a Client Component. The Server Component still renders on the server; the Client Component just renders a "hole" where the already-rendered server output is slotted in. This is the canonical escape hatch:

```tsx
// ClientShell.tsx  — 'use client'
export function ClientShell({ children }) {
  const [open, setOpen] = useState(false)
  return <div>{children}</div>   // children rendered on the server
}

// page.tsx — Server Component
<ClientShell>
  <ServerOnlyDataView />   {/* stays a Server Component! */}
</ClientShell>
```

This lets interactive client wrappers (theme toggles, accordions, providers) contain server-rendered, data-fetching content without dragging that content into the client bundle.

## Composition Patterns

- **Push client boundaries to the leaves.** Keep the boundary as deep as possible so the maximum amount of tree stays server-rendered. A "client island" approach beats a "client root."
- **Providers at the root.** Context providers (theme, query client) must be Client Components. Wrap them once in the root layout *as a client wrapper that takes `children`*, so the rest of the tree stays server-rendered inside them.
- **`server-only` / `client-only` packages.** Importing `server-only` in a module throws a build error if that module ever ends up in a client bundle — a guardrail to prevent leaking secrets across the boundary.

## Common Mistakes (Senior-Level)

- **Marking a data-fetching component `'use client'` then trying to `await` in it.** Client components can't be async server-style; you lose direct data access and fall back to effects/`use()`.
- **Accidental boundary creep.** A shared util that imports a client component drags consumers client-side. Audit what your `'use client'` files import.
- **Passing a non-serializable prop** (a callback, a Prisma model instance) from server to client — silent breakage or serialization errors. Pass IDs/primitives, or a server action.
- **Assuming `'use client'` disables SSR.** It doesn't; the component still server-renders for the initial HTML. To truly skip SSR you opt out per-component (e.g. dynamic import with SSR disabled).
- **Reading `window`/`localStorage` in module scope** of a client component — runs during SSR where it's undefined. Guard in effects.

The mental model that scales: *the server tree is the trunk; client components are islands grafted onto it at explicit boundaries, and data flows down across those boundaries only as serializable props or as pre-rendered `children`.*

## See Also

- [[app-router-model]]
- [[file-conventions]]
- [[route-handlers]]
- [[10-frameworks-and-stacks/react/fiber/fiber-architecture|Fiber Architecture]]
- [[10-frameworks-and-stacks/react/philosophy/declarative-ui|Declarative UI]]
- [[10-frameworks-and-stacks/react/server-components/server-vs-client-components|Server vs Client Components (React)]] — the React-level rules
