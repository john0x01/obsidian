# Module Resolution

Resolution is the step that turns a *specifier* — the string in `import x from 'specifier'` — into a concrete module key (a URL on the web, an absolute path on a host) that the loader can fetch and cache. The language deliberately leaves most of this to the *host*: the ESM spec defines a `HostResolveImportedModule` hook and says almost nothing about how it works, so browsers and Node implement different algorithms over the same syntax. Knowing where the language ends and the host begins is what untangles "works in the bundler, breaks in the browser" bugs.

## The Three Specifier Classes

Hosts distinguish three syntactic shapes, and the distinction is load-bearing:

- **Relative** — begins with `./`, `../`, or `/`. Resolved against the importing module's URL/path. Portable and unambiguous.
- **Absolute** — a full URL (`https://…`) in browsers, or an absolute file path/`file:` URL.
- **Bare** (a.k.a. *bare specifier*) — `react`, `lodash/fp`. It names a package, not a location. Bare specifiers have **no intrinsic meaning** on the web platform; the browser has no built-in `node_modules` walk. They must be mapped by an import map, or they error.

The browser's native resolver, in effect, only knows relative and absolute. Everything else needs an explicit map.

## Import Maps (Browsers)

An import map is a JSON document (inline `<script type="importmap">`) declaring how bare specifiers and prefixes rewrite to URLs:

```html
<script type="importmap">
{ "imports": {
    "lodash": "/node_modules/lodash-es/lodash.js",
    "utils/": "/src/utils/"          // trailing-slash = prefix remap
  },
  "scopes": {
    "/legacy/": { "lodash": "/vendor/lodash@3.js" }
  }
}
</script>
```

Two mechanisms matter. **Prefix mapping** (trailing slash) lets one entry cover a whole subtree. **Scopes** allow the *same* bare name to resolve differently depending on which module is importing it — the browser-native answer to "two versions of one dependency in one app," analogous to nested `node_modules`. There may be only one import map, and it must be present before the first module load, since resolution is otherwise irreversible.

## The Resolution Algorithm, Conceptually

Independent of host, every resolver is a function `resolve(specifier, parentURL) → key`:

```
1. classify specifier (relative | absolute | bare)
2. relative/absolute → URL-resolve against parentURL (or as-is)
3. bare → consult the host's mapping mechanism
          (browser: import map / scopes;  node: package lookup + exports field)
4. canonicalize to a unique key
5. dedupe via the module map; fetch+construct only if absent
```

The output is a *key*, and key identity is what makes a module a singleton. Two specifiers that canonicalize to different keys (e.g. differing query strings, or `lodash` vs `lodash/index.js`) produce *two distinct module instances* — a classic source of duplicated singletons, doubled state, and `instanceof` failures across the "same" library.

Node's concrete bare-specifier algorithm (the `node_modules` walk, `package.json` `"exports"`/`"imports"` maps, conditional exports, extension and directory-index resolution, the dual CJS/ESM hazard) is host-specific and covered separately — see the Node module notes.

## How Bundlers Resolve and Tree-Shake

Bundlers (esbuild, Rollup, webpack, Vite) run their *own* resolver at build time rather than deferring to the runtime host. They typically emulate Node's algorithm but layer configurable extras: `resolve.alias`, multiple `extensions`, the `"browser"`/`"module"`/`"main"` field precedence, and `conditions` for picking ESM vs CJS entry points. This is precisely why a path can resolve in your dev bundle yet 404 in the browser — the bundler's resolver is more forgiving than the platform's.

**Tree-shaking** depends on the *static* nature of ESM (covered in the ESM internals note). The bundler:

1. Builds the full module graph by following static `import`/`export` (which it can do without executing code — the payoff of static structure).
2. Marks which exports are actually *reached* from the entry points, treating the graph as a reachability problem over named bindings.
3. Drops unreached exports as dead code, provided they have no observable **side effects**.

Side effects are the catch. A module whose mere evaluation mutates global state (registers a polyfill, pushes to a registry) cannot be dropped even if its exports are unused — removing it would change behavior. Bundlers cannot prove purity in general, so they rely on the `"sideEffects": false` hint in `package.json` to *assert* that a package's modules are pure and safe to prune. A wrong hint silently breaks polyfills; a missing one defeats tree-shaking. Dynamic `import()` and any CJS interop also blunt shaking, because computed specifiers and dynamic `exports` reintroduce the runtime uncertainty that static analysis cannot see through.

The senior mental model: **resolution decides identity, static structure decides shakeability, side effects decide what's actually removable.**

## See Also
- [[module-systems]]
- [[esm-internals]]
- [[08-quality-and-operations/devops-and-infrastructure/build-systems|Build Systems]]
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime]]
