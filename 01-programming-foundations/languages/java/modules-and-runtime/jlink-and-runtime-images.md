# jlink And Runtime Images

`jlink` assembles a *custom runtime image*: a trimmed JVM containing only the modules your application actually uses, plus your application's own modules, linked into one self-contained directory. It exists because Project Jigsaw made the JDK modular (see [[jpms-module-system]]) — once the platform is a set of resolvable modules, you can compute the closure your program needs and discard the rest, attacking Java's two chronic complaints: fat runtimes and slow startup.

## What jlink Produces

Given a set of root modules (`--add-modules`) and a module path, `jlink` resolves the module graph and copies the reachable modules into an image with `bin/`, `conf/`, and a `lib/modules` file — the *jimage*, a single memory-mappable container of the linked classes. Because resolution is transitive and fails fast on missing or duplicate modules, the image is verified at link time, not first use.

Key properties and plugins:

- Only **explicit** modules qualify — modular JARs with a `module-info`. Automatic modules (a plain jar dropped on the module path) are rejected, which is the usual migration wall.
- `--compress`, `--strip-debug`, `--no-header-files`, `--no-man-pages` shrink the image; `--launcher name=module/mainclass` bakes in an executable entry point.
- `--bind-services` is needed to retain `ServiceLoader` providers, which are otherwise pruned as unreachable.

A hello-world image can drop from a full JDK of hundreds of megabytes to tens of megabytes — decisive for container layers (see [[08-quality-and-operations/devops-and-infrastructure/containers|Containers]]) where you ship the runtime alongside the app.

## jdeps: Computing The Closure

You rarely know your exact module set by hand. `jdeps` performs static dependency analysis on classes and jars; `jdeps --print-module-deps` emits the comma-separated module list to feed straight into `jlink --add-modules`. It also flags reliance on encapsulated JDK internals (`--jdk-internals`) — the same `sun.*` APIs that strong encapsulation now blocks — making it the standard first step when modularizing or slimming a legacy classpath app.

## AppCDS And Class-Data Sharing

Trimming the image shrinks footprint; **Class-Data Sharing (CDS)** attacks startup. CDS writes the parsed, verified representation of a set of classes into an archive that the JVM memory-maps at launch, so class loading and verification are largely skipped and the pages are shared copy-on-write across JVM processes. A default archive for `java.base` ships with the JDK. **AppCDS** (JEP 310) extends this to application classes; **Dynamic CDS** (JEP 350) records the archive automatically at exit with `-XX:ArchiveClassesAtExit`, and `-XX:+AutoCreateSharedArchive` regenerates it as needed. This is a startup and memory-density win, especially for short-lived or heavily-replicated services. Project Leyden is pushing this further toward an ahead-of-time cache of loaded-and-linked classes (and eventually compiled code), narrowing the gap with native images while keeping the JVM.

## jpackage: Shipping To Users

`jpackage` (JEP 392) wraps an application plus a runtime image (built via `jlink` under the hood) into a platform-native installer or app-image — `deb`/`rpm`, `msi`/`exe`, `dmg`/`pkg`. It targets *distribution*: the end user gets a double-clickable app with an embedded runtime and no separate JDK install. It is the modern replacement for the removed Java Web Start and applet delivery models.

## Contrast With GraalVM native-image

A jlink image still ships a JVM — the interpreter, the C1/C2 JIT, dynamic class loading, full reflection — just a smaller one. GraalVM `native-image` instead performs **closed-world AOT compilation**: it statically analyzes the whole program at build time and emits a standalone native executable with no JVM. The trade-offs are the crux of [[07-performance-engineering/jit-vs-aot|JIT vs AOT]]:

- **native-image** gives millisecond startup and low memory (no warmup, no JIT, no metaspace), ideal for CLIs and serverless — but pays with long builds, loss of peak JIT throughput, and a *closed world*: reflection, dynamic proxies, resources, and lambdas must be declared in configuration because nothing can be discovered at runtime.
- **jlink image** keeps full dynamic Java and eventual peak performance; it merely removes unused modules and, with CDS, warms startup — a smaller footprint without abandoning the JVM's adaptive optimization.

The senior mental model: `jlink` plus CDS is the *conservative* footprint-and-startup play that preserves Java's dynamism; native-image is the *aggressive* one that trades dynamism for a fixed, fast, small artifact. Choose per workload — long-running throughput services lean jlink; scale-to-zero functions lean native.

## See Also
- [[jpms-module-system]]
- [[class-loading]]
- [[jvm-architecture]]
- [[07-performance-engineering/jit-vs-aot|JIT vs AOT]]
- [[08-quality-and-operations/devops-and-infrastructure/containers|Containers]]
