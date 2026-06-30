# Composition Model

React's reuse strategy is **composition, not inheritance**. Components are functions; you build complex UI by nesting and parameterizing those functions — passing elements as `children`, passing behavior as props, wrapping with other components — never by subclassing. This is a deliberate rejection of the class-hierarchy approach to UI reuse, and it is the reason React has no `extends Component` for *your* components, no mixins, and no template inheritance.

## Components as composable functions

A component is `(props) -> element tree`. Composition is therefore ordinary function composition plus a tree-shaped data type (elements). Three composition primitives cover the vast majority of real designs:

**1. Containment via `children`.** A component that doesn't know its content ahead of time accepts it as a prop:

```jsx
function Card({ children }) {
  return <div className="card">{children}</div>;
}
<Card><Avatar /><Bio /></Card>
```

`children` is just `props.children` — an opaque element value the parent produced. `Card` never reaches into it; it places it. You can also pass *named slots* as distinct props (`<Layout sidebar={<Nav/>} main={<Feed/>} />`), which is strictly more flexible than a single children channel.

**2. Specialization via props.** Instead of `DangerButton extends Button`, you write `<Button variant="danger">`. The "subclass" relationship becomes a *configuration argument*. A more specialized component is one that renders a general one with some props fixed:

```jsx
const DangerButton = (p) => <Button {...p} variant="danger" />;
```

This is specialization-by-composition: the general component is a collaborator, not an ancestor.

**3. Inversion of control via render props / function children.** When a component owns *behavior or state* but not *markup*, it hands the markup decision back to the caller as a function. This was the canonical pre-hooks pattern for sharing stateful logic and still matters for ownership-of-rendering cases (virtualized lists, headless components).

## Composition vs configuration

There is a tension every senior eventually feels: do you expose a *prop* (configuration) or a *slot* (composition)? Configuration (`icon="search"`) is closed — it works only for cases you anticipated. Composition (`icon={<SearchIcon/>}`) is open — the caller supplies arbitrary content. The React-idiomatic default is to lean toward composition: accept elements, not flags, so callers extend you without you editing your component. Over-configured components accrete boolean props (`isPrimary`, `isLarge`, `hasIcon`) and collapse under their own combinatorics; composition pushes those decisions to the call site.

## Why React rejected inheritance and mixins

- **Inheritance couples reuse to identity.** Subclassing forces an *is-a* hierarchy and shares the `this` namespace, so a base method and a subclass method can silently collide. UI reuse is almost always *has-a* / *uses* — better modeled as a value (`children`) or a collaborator (a wrapped component) than as an ancestor.
- **Mixins failed concretely.** `React.createClass` mixins shared methods and state into a component, producing implicit name clashes, unclear ordering, and "where did this method come from?" opacity. They didn't compose (mixin A and mixin B touching the same lifecycle was a guessing game) and they obscured data flow. React deprecated them, recommended HOCs/render props, and ultimately replaced the whole category with **hooks**, which compose by *explicit call* rather than implicit merge.
- **HOCs (historical).** A higher-order component `withX(Component)` wrapped a component to inject props. They worked but introduced wrapper-hell, prop-name collisions, ref-forwarding ceremony, and indirection in the tree. Hooks superseded them for logic reuse because a custom hook composes values into a *single* component scope with no wrapper element and no prop plumbing — the same "share stateful logic" goal achieved by function composition instead of component composition.

## Senior pitfalls and misconceptions

- **"Hooks replaced composition."** No — hooks replaced *HOCs/render-props for logic reuse*. Tree-shaped UI composition via `children`/slots is as central as ever. The two operate on different axes: hooks compose *behavior*, JSX composes *markup*.
- **Prop-drilling is not a composition failure.** Often the fix is *component composition* — pass the deep element down as `children` from where the data lives, so intermediate layers never see the prop at all — rather than reaching for context.
- **Wrapping breaks identity.** Wrapping a component in a new component on every render gives it a new element type each time, which makes reconciliation remount it (state loss). Define wrappers at module scope.
- **Composition is structural, not behavioral inheritance.** You can't "override a parent's render." You can only supply what a component invites you to supply. Designing those invitation points (which props are slots) *is* component API design.

The philosophy in one line: React models UI reuse as **building a tree out of small functions and the values you pass them**, which keeps data flow explicit and avoids every failure mode that hierarchical, implicit sharing introduced.

## See Also

- [[declarative-ui]]
- [[elements-vs-components]]
- [[unidirectional-data-flow]]
- [[01-programming-foundations/paradigms/inheritance-vs-composition|Inheritance vs Composition]]
- [[01-programming-foundations/paradigms/higher-order-functions|Higher-Order Functions]]
- [[01-programming-foundations/paradigms/object-oriented-programming|Object-Oriented Programming]]
