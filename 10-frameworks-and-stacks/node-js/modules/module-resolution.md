# Module Resolution

Resolution is the algorithm that turns a specifier string (`'express'`, `'./util.js'`, `'#config'`) into a concrete file on disk. Node runs *two* related-but-distinct resolvers — one for CommonJS `require`, one for the ESM loader — and the differences between them (extension guessing, directory indexes, URL semantics) are a frequent source of "works in CJS, breaks in ESM" bugs.

## Specifier Categories

- **Relative** (`./`, `../`): resolved against the importing file.
- **Absolute / URL** (`/abs/path`, `file:///…`): used as-is.
- **Bare** (`express`, `lodash/fp`): resolved through `node_modules` and package metadata.
- **Builtin** (`node:fs`, `fs`): mapped to core modules; `node:` prefix is the unambiguous form and is mandatory for some newer cores.

## The `node_modules` Walk

For a bare specifier, both resolvers walk upward: check `./node_modules/`, then `../node_modules/`, then `../../node_modules/`, up to the filesystem root, taking the first match. This is why hoisting (flattening duplicate deps to a shared ancestor `node_modules`) works and why nested duplicates produce multiple instances of the same package — each is found at a different point in the walk. The walk is the structural reason `node_modules` resolution is location-dependent rather than global.

## Package Entry Points

Once a package directory is found, its `package.json` chooses the entry:

- Legacy: `"main"` points at the entry file; for ESM, a bare `"main"` is still honored.
- Modern: `"exports"` defines an **encapsulated public API** and, critically, *blocks* deep imports of internal files not listed there. With `"exports"` present, `require('pkg/lib/secret.js')` fails unless explicitly exposed.

`"exports"` is the single biggest change in modern resolution: packages went from "every file is importable" to "only declared subpaths are importable," which lets maintainers refactor internals without breaking consumers.

## Conditional Exports

`"exports"` values can branch on **conditions** the resolver matches in order:

```json
{
  "exports": {
    ".": {
      "types": "./index.d.ts",
      "node-addons": "./native.node",
      "node": { "import": "./index.mjs", "require": "./index.cjs" },
      "browser": "./index.browser.js",
      "default": "./index.js"
    }
  }
}
```

Conditions are matched top-to-bottom; order in the object matters. Common keys: `import`/`require` (which syntax loaded it), `node`/`browser`/`deno` (environment), `types` (must come first, for TypeScript), `development`/`production` (via `--conditions`), and `default` (fallback). This is how dual packages route ESM vs CJS callers to different files and how a package ships a browser build.

## Subpath Patterns and `imports`

- **Subpath exports** expose multiple entry points: `"./utils": "./src/utils.js"`.
- **Patterns** use a single `*` wildcard to map a family of paths: `"./features/*": "./src/features/*.js"` lets `import 'pkg/features/foo'` resolve to `src/features/foo.js`. The `*` is a literal substring substitution, not a glob.
- **`"imports"`** defines *internal* private specifiers, conventionally prefixed `#`: `{ "imports": { "#config": "./config.prod.js" } }`. These are resolvable only from inside the package, support the same conditions, and are the clean way to do internal aliasing without `node_modules` hacks or build-time path rewriting.

## Self-Referencing

Inside a package, you may import the package *by its own name* (`import { x } from 'my-pkg'`), and Node routes it through the package's own `"exports"`. This keeps internal imports going through the public API surface, so tests and examples exercise exactly what consumers see — and it removes brittle `../../..` relative chains.

## CJS vs ESM Resolver Differences

| Behavior | CJS (`require`) | ESM (loader) |
|---|---|---|
| Extension guessing | Yes (`./x` → `x.js`, `x.json`, `x.node`) | No — extension required (unless via `exports`) |
| Directory index | Yes (`./dir` → `dir/index.js`) | No |
| Identity model | Filesystem paths | `file:` URLs (percent-encoding, query strings) |
| `exports`/`imports` | Honored | Honored |
| Loading | Synchronous | Asynchronous, hookable |

The ESM resolver is **stricter and URL-based**: no magic extensions, no implicit `index`, and specifiers are real URLs (so `?` and `#` have URL meaning). This strictness is the trade Node accepted for spec-compliance and static analyzability. ESM also exposes programmable resolution/load **hooks** (registered via `module.register`, running off-thread), which CJS lacked short of monkey-patching `Module._resolveFilename`.

## Senior Pitfalls

- Deep-importing a path not in `"exports"` — silently worked for years, now throws.
- Forgetting `types` must be the first condition or TypeScript won't see your declarations.
- Assuming ESM will guess extensions like CJS — it won't.
- Condition ordering bugs: a broad `default` placed above `node` shadows the specific one.

## See Also

- [[commonjs-internals]]
- [[esm-in-node]]
- [[01-programming-foundations/languages/javascript/modules/module-resolution|JS Module Resolution]]
- [[03-computer-systems/operating-systems/filesystems|Filesystems]]
