# Unidirectional Data Flow

React enforces that data moves **one direction**: down the component tree as props, while change requests move **up** as events (callbacks). State lives at a single owning node; children receive read-only snapshots and ask the owner to change them. This "props down, events up" discipline is a deliberate rejection of two-way binding, and it is the structural reason React apps are debuggable: at any instant the tree is a pure function of the state at its roots.

## The shape of the flow

```
        state (owner)
           │ props (down)
           ▼
        Child  ── onChange(value) ──┐  event (up)
           │ props (down)            │
           ▼                         │
       Grandchild ── onClick() ──────┘
```

A child cannot mutate the data it receives. Props are immutable from the child's perspective; the only lever a child has is to invoke a callback the owner gave it. The owner decides whether and how to update its own state, which re-renders the subtree with new props. Data and the authority to change data are intentionally separated: you *have* the value, but only the *owner* can change it.

## Single source of truth & lifting state

Every piece of mutable state should have exactly one authoritative home. When two components need the same state, you **lift state up** to their nearest common ancestor and pass it back down. The components below become "controlled" — their displayed value is dictated entirely by the prop, and their changes are forwarded up. The canonical example is a controlled `<input value={x} onChange={e => setX(e.target.value)} />`: the DOM node is *not* the source of truth; `x` is. The input merely renders `x` and reports keystrokes upward.

This is the same idea as normalizing a database: avoid duplicated state that can drift out of sync. Derived data should be *computed during render* from the single source, never stored as a second copy of state. Two `useState`s that must always agree is a smell — collapse them or derive one.

## Why two-way binding was rejected

Frameworks with two-way binding (Angular's classic `ng-model`, Knockout, Vue's `v-model` sugar) let a view node and a model field mutate each other automatically. It feels convenient for forms, but it scales badly:

- **Cycles and cascades.** A↔B bindings can trigger each other; the framework needs dirty-checking / digest loops / change-detection passes to reach a fixed point, and those passes are where the famous performance cliffs and "$digest already in progress" classes of bugs live.
- **Diffuse causality.** When any binding can write back to the model, the answer to "what changed this value?" is "anything bound to it." Unidirectional flow makes the answer always "the owner's setState, triggered by some event" — a single, greppable choke point.
- **Hidden mutation breaks `UI = f(state)`.** Two-way binding lets the *view* be a writer of state, blurring the input/output boundary. React keeps the view strictly an *output*; inputs arrive only through explicit handlers. (Note: `v-model` and React's controlled inputs are actually the same one-way mechanism under the hood — value down, event up; React just refuses to hide the event leg in sugar.)

React's stance: two-way binding is convenient sugar that conceals a cycle, and cycles are where predictability dies. Make every write explicit and the system stays traceable.

## Why this yields predictability and debuggability

- **Deterministic replay.** Because the view is downstream of state and writes only happen through dispatched updates, replaying a list of actions reproduces the UI exactly — the premise behind Redux/Flux, time-travel devtools, and state-preserving hot reload.
- **Localized reasoning.** To understand why a node renders what it does, you read the props flowing in; you never have to audit who else might be writing to its DOM. Bugs reduce to "the state was wrong" or "the wrong callback fired," both inspectable at the owner.
- **Flux/Redux generalize the same arrow.** action → reducer → new state → re-render is unidirectional flow scaled to app level: a strict, auditable pipeline with no back-edges. Even when state is centralized in a store, the data-flow shape is unchanged.

## Senior pitfalls and misconceptions

- **Refs and `useImperativeHandle` are the sanctioned back-channel** for genuinely imperative needs (focus, scroll, media playback) — but they're parent→child *commands*, not data flow, and overusing them recreates the spaghetti unidirectional flow was meant to kill.
- **Context is still unidirectional.** It lets data skip intermediate levels, but it still flows *down*; updates still happen via a provider's setState higher up. It's a transport optimization, not a new direction.
- **Lifting too high is a real cost.** State lifted above where it's needed forces wide re-renders and prop-drilling. The art is lifting to the *nearest* common owner, not the root.
- **Derived-state duplication.** Copying a prop into state in an initializer and then letting them drift is the most common single-source-of-truth violation. Prefer deriving in render or fully controlling the value.

## See Also

- [[declarative-ui]]
- [[composition-model]]
- [[reconciliation]]
- [[01-programming-foundations/paradigms/dataflow-programming|Dataflow Programming]]
- [[01-programming-foundations/paradigms/reactive-programming|Reactive Programming]]
- [[01-programming-foundations/paradigms/immutability|Immutability]]
