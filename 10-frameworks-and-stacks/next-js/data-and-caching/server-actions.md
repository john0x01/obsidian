# Server Actions

A Server Action is a function marked `'use server'` that runs **only on the server** but can be called from client or server code as if it were local. It is React's built-in RPC mechanism: you write an async function, hand it to a `<form action={fn}>` or call it from an event handler, and React arranges the network round-trip, serialization, and re-render for you. In Next.js, Server Actions are the canonical way to perform mutations — the App Router's answer to "where does the POST go."

## The RPC mechanism

`'use server'` does not bundle the function to the client. Instead, the bundler replaces it at the call site with a **reference**: a stable, opaque **action ID** (a hash) plus a small client stub. When the client invokes the action, React serializes the arguments and POSTs them — to the **same URL the page is on**, not a separate API route — with headers identifying the action ID. The server router maps the ID back to the real function, deserializes the args, runs it, and streams back the result *plus* any re-rendered RSC payload triggered by revalidation. So an action call is simultaneously a mutation and (often) a render request.

```
client: action(formData)
   │  POST current-url   { actionId, serialized args }
   ▼
server: lookup actionId ─▶ run fn(args) ─▶ revalidate? ─▶ re-render
   │  RSC payload (updated tree) + return value
   ▼
client: apply payload, resolve the promise
```

## Forms, mutations, and revalidation

The progressive-enhancement story is the whole point of binding actions to `<form action={...}>`: the form is real HTML that **works before JS hydrates**. Submit early and the browser does a native form POST to the action; once hydrated, React intercepts and does the RPC instead — same function, no full page reload. `useFormStatus` and `useActionState` (React 19) layer on pending state and returned errors without abandoning the no-JS baseline.

After a mutation you almost always invalidate caches so the UI reflects the new state:

```ts
'use server';
export async function addItem(formData: FormData) {
  await db.insert(/* ... */);
  revalidatePath('/items');     // or revalidateTag('items')
}
```

Because the action's response includes the re-rendered RSC payload, the client patches the affected segments in place — no manual refetch, no client-side store to keep in sync. This is the mutation-then-revalidate loop that replaces the old `mutate()` + manual cache-update dance (see [[revalidation]]).

## Security model — treat actions as public endpoints

This is the part seniors underestimate. **An action ID is a publicly reachable POST endpoint.** Anyone can craft a request with that ID and arbitrary arguments. Therefore:

- **Always authenticate/authorize inside the action.** The fact that you only rendered the button for admins is irrelevant; the endpoint exists regardless. Re-check the session and permissions in the function body.
- **Validate and never trust arguments.** Parse `FormData`/args with a schema (e.g. Zod) on the server. The client controls them.
- **Action IDs are stable hashes**, not secrets, and they are deterministic across builds for the same function — don't rely on obscurity.
- **Closed-over variables are encrypted.** When an action closes over values from the enclosing scope (e.g. an `id` captured in a component), Next serializes those bound arguments and **encrypts** them with a per-deployment key before sending the reference to the client, so the client can't read or tamper with them. This is why a leaked closure value isn't trivially exposed — but it is *not* a substitute for authorization, because the *caller* still supplies the explicit arguments. Across multiple instances/deployments you must keep the encryption key stable (Next derives it; for self-hosting set a consistent secret) or older references break.
- **Dead-code-eliminate carefully.** An unused action can still be referenced if imported; only actions that are actually wired up get IDs registered.

## Progressive enhancement nuances

- Inline `'use server'` actions defined in a Server Component can close over render-time data; module-level actions in a `'use server'` file are reusable but receive everything as explicit args.
- Non-form invocation (calling the action from an `onClick`) loses the no-JS fallback by definition — there's no form to submit. Reserve form binding for the cases where enhancement matters.
- Actions run on the server runtime you configured (Node or Edge); long work should be offloaded, since the action holds the request open and the client awaits the re-render.

## Pitfalls and misconceptions

- **"It's hidden so it's safe."** No — it's an open RPC endpoint. Authorize in-body.
- **Confusing action IDs with CSRF protection.** Next checks `Origin`/`Host` to mitigate cross-site POSTs, but your own authz is still mandatory.
- **Expecting return values to update the UI by themselves.** The return value resolves the promise; UI freshness comes from `revalidatePath`/`revalidateTag` re-rendering the tree, or from `useActionState` wiring the result into state.
- **Mutating then reading stale data on the client** because you forgot the revalidation call — the server changed, the Router Cache didn't.

## See Also

- [[revalidation]]
- [[caching-layers]]
- [[rsc-and-flight]]
- [[client-server-boundary]]
- [[06-software-design/api-design/idempotency|Idempotency]]
- [[06-software-design/software-architecture/cqrs|CQRS]]
