# JSX

JSX is **syntax sugar that compiles to function calls returning element objects** — it has no runtime of its own. A build step (Babel, SWC, TypeScript, esbuild) transforms each tag into a call that produces a plain element (see [[elements-vs-components]]). Understanding JSX as "an expression-level DSL for `createElement`" — and knowing exactly what the compiler emits in React 19's automatic runtime — removes essentially all of JSX's mystery.

## What the compiler emits

There are two transforms. The **classic** transform (pre-React 17, still selectable) lowers tags to `React.createElement`, which is why `React` had to be in scope in every JSX file:

```jsx
<div className="x">{a}</div>
// classic →
React.createElement('div', { className: 'x' }, a);
```

The **automatic** runtime (React 17+, default in React 19) imports dedicated helpers from a separate entry point and no longer needs `React` in scope:

```jsx
// automatic → compiler injects:
import { jsx as _jsx, jsxs as _jsxs } from 'react/jsx-runtime';
_jsx('div', { className: 'x', children: a });
```

Key differences in the automatic runtime:

- **`children` becomes a prop**, not a trailing argument. Single child → `children: child`; multiple → `children: [...]`.
- **`jsx` vs `jsxs`.** The compiler calls `jsxs` ("s" = static) when children are a *statically known array* (multiple literal children), and `jsx` for a single/dynamic child. The split lets React skip a runtime check about whether children is a stable array — a small key-validation/perf optimization, not something you ever call yourself.
- **`key` is hoisted out of props** and passed specially, because `key` is reconciliation metadata, not data for the component (see [[reconciliation]]). In dev there's also a `jsxDEV` variant carrying source location and validation.

Lowercase tags (`div`) compile to **string** types (host elements); capitalized tags (`Counter`) compile to the **identifier** type — this is purely a casing convention the compiler uses to decide string vs reference. `<Foo.Bar/>` and `<obj.member/>` likewise resolve to property access on a value.

## Expressions, not statements

`{ }` in JSX embeds a JavaScript **expression** — something that evaluates to a value — never a statement. This is why you write `cond ? <A/> : <B/>` and `list.map(...)` but cannot write `if`/`for`/`switch` inline: statements don't produce a value to splice into the `children` argument. Everything in JSX is ultimately *building an argument list*, and arguments are expressions. The values that render are elements, strings, numbers, arrays of those, fragments, portals, and (React 19) Promises/`null`/`false`; objects and `true` do not render.

Whitespace, `{' '}`, and adjacent string/expression children all become entries in the `children` argument exactly as the compiler serializes them — a frequent source of "why is there a space here" confusion that dissolves once you picture the emitted call.

## Design rationale: colocation and the "logic + markup" stance

JSX is React's most contested design decision and its most deliberate one. The traditional web orthodoxy was **separation of concerns by technology**: HTML in one file, JS in another, CSS in a third. React's counter-thesis is that those are *separation by technology, not by concern* — the real unit of concern is a component, which inherently couples its markup and the logic that drives it. A dropdown's template and its open/close logic change together and should live together. JSX therefore colocates markup *inside* the language, so the full power of JS (variables, functions, composition, type-checking) applies to the view with no template sublanguage.

Crucially JSX is **not string templating**. It produces typed object trees, not concatenated strings, which means: no injection by default (children are escaped), static analysis and TypeScript can check element types and props, and editors get real autocomplete. Compare to template DSLs (Handlebars, Angular templates) that invent their own loop/conditional/expression mini-languages — JSX deliberately has *none* of that, deferring entirely to JavaScript expressions. The tradeoff React accepted: a mandatory compile step and a learning curve, in exchange for one language, full type safety, and no template/logic impedance mismatch.

## Senior pitfalls and misconceptions

- **JSX is not HTML.** It's an expression DSL: `class` → `className`, `for` → `htmlFor`, attributes are camelCased, and `{}` holds JS, not interpolation strings. (React 19 relaxed several of these, e.g. accepting `class`-like custom attributes and many lowercase/`data-*`/`aria-*` passthroughs more permissively.)
- **`<X/>` does not call `X`.** It builds an element whose `type` is `X`; React calls it later. Confusing creation with invocation underlies most "extra renders" and "hooks order" confusion.
- **Fragments compile to a real type** (`React.Fragment` / the `jsx`-emitted fragment symbol), so they participate in reconciliation as a keyless wrapper — `<></>` can't take a `key`, but `<Fragment key=...>` can.
- **You're not limited to JSX.** Because it's just sugar over `jsx`/`createElement`, hand-written calls are equivalent — useful for understanding what the compiler does, and for the rare programmatic-element case.

## See Also

- [[elements-vs-components]]
- [[virtual-dom]]
- [[reconciliation]]
- [[declarative-ui]]
- [[01-programming-foundations/paradigms/declarative-programming|Declarative Programming]]
