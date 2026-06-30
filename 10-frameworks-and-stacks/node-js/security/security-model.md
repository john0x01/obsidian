# Security Model

Node's foundational security assumption is the inverse of a browser's: **the code you run is trusted, the data it processes is not.** A browser sandboxes the script; Node hands the script full ambient authority over the filesystem, network, environment, and child processes the moment it starts. There is no privilege boundary between your code and a transitive dependency — `require('left-pad')` runs with exactly the same power as your `main`. Understanding this is the whole game: most Node CVEs are not bugs in V8, they are your trusted code doing something dangerous with untrusted input, or a dependency you trusted turning out to be hostile.

## The Node threat model (what core defends against)

The Node project publishes a threat model that draws a sharp line. **In scope** (a vulnerability core will fix): data the program treats as untrusted but Node mishandles — network input, untrusted file *contents*, etc. **Out of scope** (your responsibility, not a Node bug): if you pass attacker-controlled data to an API that is documented to require *trusted* input. Concretely, things like `child_process` command strings, `vm` (which is explicitly **not a security sandbox**), `require`/import paths, and the contents of `package.json` scripts are all assumed-trusted. Knowing which side of this line an API sits on tells you where *you* must validate.

## The permission model (`--permission`)

Node 20+ ships an experimental, opt-in **Permission Model** that finally introduces an in-process privilege boundary. Enable with `--permission` (the flag was promoted from the earlier `--experimental-permission`); it denies by default and you grant capabilities explicitly:

```bash
node --permission \
  --allow-fs-read=/app --allow-fs-read=/etc/ssl \
  --allow-fs-write=/app/tmp \
  --allow-child-process \
  --allow-worker \
  server.js
```

Granted permissions cover filesystem read/write (path-scoped), spawning child processes, creating workers, and addons. At runtime `process.permission.has('fs.read', '/path')` lets code check before acting. Critical caveats: it is **experimental**, it does **not** sandbox network access, it is bypassable by native addons (which is why `--allow-addons` is separate), and it is a defense-in-depth layer to limit blast radius — *not* a substitute for OS-level isolation (containers, seccomp, users). Treat it as "reduce what a compromised dependency can reach," not "safely run untrusted code."

## Common vulnerability classes

- **Prototype pollution.** JS's mutable prototype chain means writing to `__proto__`/`constructor.prototype` via a recursive merge, `JSON`-driven object assignment, or a buggy query-string parser can inject properties onto *every* object, escalating to RCE or auth bypass. Defenses: `Object.create(null)` for maps, `Object.freeze(Object.prototype)`, reject `__proto__`/`constructor`/`prototype` keys, validate with a schema, and prefer `Map` over plain objects for untrusted keys.
- **ReDoS (regex denial of service).** Catastrophic backtracking in a regex applied to attacker input (classically nested quantifiers like `(a+)+$`) makes one request peg the single event-loop thread, stalling the whole process. Defenses: avoid ambiguous patterns, use linear-time engines (RE2) for untrusted input, set timeouts, and audit dependency regexes.
- **Path traversal.** Concatenating user input into a file path lets `../../etc/passwd` escape the intended directory. Defenses: `path.resolve` then verify the result is *prefixed* by the allowed root, never trust the input string; leverage `--allow-fs-read` scoping as a backstop.
- **Command injection.** `child_process.exec`/`execSync` run a *shell*, so any interpolated input is shell-parsed (`; rm -rf`, `$(...)`). Defenses: use `execFile`/`spawn` with an **argument array** and `shell: false` (the default) so arguments are never re-parsed; never build shell strings from input.
- **Supply-chain.** The dominant modern risk: a malicious or compromised npm package (typosquat, hijacked maintainer, malicious postinstall script) executing in your trust context. Defenses: lockfiles + `npm ci`, `npm audit`, `--ignore-scripts` for installs, Sigstore/provenance attestation, minimizing dependency count, and runtime egress controls.

## Secure defaults and where Node helps

Modern Node has shifted toward safer defaults: ESM is `strict mode`; `--frozen-intrinsics` hardens built-in prototypes against tampering; the permission model exists; `--disable-proto=delete|throw` neutralizes the `__proto__` accessor; `node --run` executes `package.json` scripts *without* shell features that the older npm path exposed; and the built-in test runner / `node:sqlite` reduce the dependency surface. But defaults only go so far — `child_process` still spawns a shell on request, `vm` still isn't a sandbox, and the permission model is still off unless you flip it.

## Senior pitfalls and philosophy

The recurring senior mistake is treating the permission model or `vm` as a sandbox for *untrusted code* — neither is, and architecting around that misconception is how breaches happen; isolate genuinely untrusted code in a separate process/container/VM with OS enforcement. The second is under-weighting supply chain: your own code is usually a small fraction of what executes, so dependency hygiene buys more security than hardening handlers. The unifying philosophy: **validate at the trust boundary** (every API that says "trusted input"), **minimize ambient authority** (fewer dependencies, scoped permissions, no shell), and **assume breach** — design so a single compromised module can reach as little as possible.

## See Also
- [[profiling-and-diagnostics]]
- [[08-quality-and-operations/security/supply-chain-security|Supply Chain Security]]
- [[08-quality-and-operations/security/threat-modeling|Threat Modeling]]
- [[08-quality-and-operations/security/secure-coding|Secure Coding]]
- [[01-programming-foundations/languages/javascript/language-semantics/prototypes-and-inheritance|Prototypes and Inheritance]]
- [[08-quality-and-operations/security/owasp-top-10|OWASP Top 10]]
