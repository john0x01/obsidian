# I/O Streams And Readers

The `java.io` package is Java's original, blocking I/O API, built on two orthogonal abstractions: **byte streams** (`InputStream`/`OutputStream`) for raw 8-bit data, and **character streams** (`Reader`/`Writer`) for text that has been decoded through a charset. These are the *I/O* "streams" — unrelated to the functional [[stream-api]], a perennial source of confusion. The API's enduring lesson is composition: capability is added by *wrapping*, not subclassing.

## Bytes Versus Characters

`InputStream.read()` returns an `int` in 0–255, or `-1` at end of stream; it knows nothing of encoding. Text lives in the `Reader`/`Writer` hierarchy, where a `char` is a UTF-16 code unit. The bridge between the two worlds is `InputStreamReader`/`OutputStreamWriter`, which apply a `Charset` to decode bytes into characters. The senior pitfall is constructing those bridges (or `new String(bytes)`, `FileReader`) *without an explicit charset*: pre-Java 18 they used the platform default, so code that worked on a developer's UTF-8 laptop mangled text on a server with a different locale. JEP 400 made UTF-8 the default charset JVM-wide in Java 18, removing that footgun — but being explicit remains the correct habit.

## The Decorator Pattern

`java.io` is the textbook implementation of the Decorator pattern (see [[06-software-design/software-architecture/design-patterns-structural|Structural Design Patterns]]): each wrapper *is* a stream and *holds* a stream, adding one concern.

```java
DataInputStream in = new DataInputStream(
    new BufferedInputStream(
        new FileInputStream("data.bin")));
```

- `BufferedInputStream` adds an in-memory buffer so each `read()` is not a syscall — the single most important wrapper for performance.
- `DataInputStream`/`DataOutputStream` add typed primitives (`readInt`, `readUTF`).
- `ObjectInputStream`/`ObjectOutputStream` add object serialization.

The benefit is combinatorial: N concerns compose freely without an N-dimensional class explosion. The cost is that responsibility for *buffering* falls on the caller — forgetting `BufferedInputStream` around a `FileInputStream` turns one logical read into one syscall per byte.

## Blocking Semantics And try-with-resources

`java.io` is strictly **blocking**: `read()` parks the calling thread until data arrives, EOF is hit, or an exception fires. Historically this forced the thread-per-connection model and its scaling ceiling — the reason [[nio-and-channels]] exists. The modern twist: on a virtual thread (Project Loom), a blocking `read()` unmounts the virtual thread from its carrier instead of pinning an OS thread, so straightforward blocking `java.io` code once again scales to enormous concurrency (see [[threads-and-the-os]]). Loom deliberately rehabilitated this simple style.

Every stream implements `Closeable` (hence `AutoCloseable`), so **try-with-resources** is mandatory discipline: it closes resources in reverse order of declaration and attaches any close-time failure as a *suppressed* exception rather than masking the original. Closing the outermost decorator flushes and closes the whole chain.

## Serialization Caveats

Built-in serialization (`Serializable`, `ObjectOutputStream`) looks convenient and is a well-known mistake. The problems are structural:

- **Hidden protocol**: the real contract is `serialVersionUID` plus the magic `writeObject`/`readObject`/`readResolve` methods — an invisible, brittle API trivial to break across versions.
- **Deserialization is object *construction* that bypasses constructors**, so invariants a class enforces in its constructor are not enforced on the way in. Untrusted input then becomes a remote-code-execution vector via *gadget chains* — one of the most exploited classes of Java vulnerability.
- Mitigation is `ObjectInputFilter` (JEP 290) to allow-list deserializable types, but the standing senior advice is to avoid Java serialization for anything crossing a trust boundary and use an explicit format (JSON, protobuf). Records help: they deserialize through their canonical constructor, restoring invariant checking.

The philosophy running through `java.io` is Unix-like — small, single-purpose components composed into pipelines — realized in an object-oriented idiom. That composability is the part worth keeping; the implicit platform-default charset and the serialization framework are the parts the platform has spent two decades walking back.

## See Also
- [[nio-and-channels]]
- [[stream-api]]
- [[threads-and-the-os]]
- [[06-software-design/software-architecture/design-patterns-structural|Structural Design Patterns]]
- [[03-computer-systems/operating-systems/io-models|I/O Models]]
