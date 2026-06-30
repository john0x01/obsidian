# Actions And Forms

Server Actions (declared with `'use server'`) are functions that run on the server but can be **called from client components as if they were local async functions**. They are React 19's mutation model: where RSC made *reading* data a server concern, Actions make *writing* it one, without you hand-authoring an API endpoint, a fetch call, or request/response plumbing. Underneath, an Action is an RPC — React and the bundler generate the wire protocol; you write a function.

## The RPC mechanism underneath

`'use server'` at the top of a module (or inside a function body) marks its exports as Server Functions. The bundler does **not** ship their code to the client. Instead, each Server Function becomes a *reference* — an opaque ID plus an endpoint. When client code "calls" it, React serializes the arguments, POSTs them to the server, the real function executes there against your database/services, and the result (or a fresh RSC payload) streams back.

```
client: action(formData)
   │  serialize args ─────────────▶ POST /<action-id>
   │                                 server: run real fn, mutate, return value/payload
   └◀──────────── deserialize result / re-render with new RSC payload
```

This is why a Server Function can be *passed as a prop* across the `'use client'` boundary (it serializes to a reference, unlike an ordinary closure) and why its arguments must be serializable. The function you wrote and the call site the client sees are two ends of an automatically generated RPC.

## Form integration and progressive enhancement

React 19 lets you pass a function directly to `<form action={fn}>`. When the action is a Server Function, the form **works before any JavaScript loads or hydrates**: the browser's native form POST hits the action endpoint, the server processes it, and responds. Once hydrated, React intercepts the submit and performs the same call via fetch instead — no full navigation, with the action receiving a `FormData` object either way.

```jsx
// app/post.js
'use server';
export async function createPost(formData) {
  const title = formData.get('title');
  await db.posts.insert({ title });
  // optionally revalidate / return state
}
```

```jsx
<form action={createPost}>
  <input name="title" />
  <button>Save</button>
</form>
```

This is genuine **progressive enhancement** restored to React: the baseline works with HTML semantics; JS upgrades the experience. The mental shift is that the `action` prop now accepts a *function*, not just a URL string.

## The new hooks

React 19 ships three hooks that turn Actions into a complete mutation UX:

- **`useActionState(action, initialState)`** wraps an action and returns `[state, wrappedAction, isPending]`. The action's signature gains a leading `previousState` argument, and its return value becomes the new `state`. This is how you surface validation errors and results from a server mutation back into the UI, while `isPending` drives disabled/spinner states — all without manual `useState` bookkeeping.

```jsx
'use client';
const [error, submit, pending] = useActionState(async (prev, formData) => {
  const res = await createPost(formData);
  return res.ok ? null : res.message;
}, null);
return <form action={submit}>{/* ... */} {pending && '…'} {error}</form>;
```

- **`useFormStatus()`** reads the pending state of the **nearest parent `<form>`** from a child component, without prop-drilling. A reusable `<SubmitButton/>` can disable itself during submission by calling `useFormStatus()` internally — it must be rendered *inside* the form whose status it reads.

- **`useOptimistic(state, updateFn)`** shows an immediate, optimistic UI while the action is in flight. You apply the expected result locally; if the action eventually returns different data (or errors and the component re-renders), React **reconciles back to the real state automatically**. The optimistic value is transient — scoped to the pending transition — so you never have to manually roll it back.

```jsx
const [optimisticTodos, addOptimistic] = useOptimistic(
  todos,
  (state, newTodo) => [...state, { ...newTodo, sending: true }]
);
// in the action handler: addOptimistic(draft) then await the real mutation
```

## The mutation model

Actions are dispatched through React's **transition** machinery: a pending action is a non-blocking transition, which is why `isPending`/`useFormStatus` are available and why the UI stays responsive. The intended cycle is: user submits → optimistic update paints instantly → Server Function runs the mutation → server returns a fresh RSC payload (or the framework revalidates affected data) → React reconciles the real result, discarding the optimistic placeholder. Reads and writes thus share one model: a write produces a new server-rendered view, which the client diffs in, rather than the client manually patching local cache.

## Senior pitfalls

- **Trusting the boundary.** A Server Function is a public POST endpoint. Arguments arrive from the network; **authenticate and validate every input inside the action** — the `'use client'` caller is not a security boundary.
- **Non-serializable arguments.** Args cross the wire; closures, class instances, and functions can't be passed (a Server Function reference can).
- **`useFormStatus` outside the form.** It reads the *nearest ancestor* form; placing the button as a sibling of the form returns no status.
- **Hand-rolling rollback with `useOptimistic`.** The hook reverts automatically on reconciliation; manual rollback fights the model.
- **Forgetting revalidation.** A mutation that succeeds but doesn't trigger the framework's data revalidation leaves stale reads on screen — the write/read loop must close.

## See Also

- [[rsc-model]]
- [[server-vs-client-components]]
- [[error-boundaries]]
- [[state-architecture]]
