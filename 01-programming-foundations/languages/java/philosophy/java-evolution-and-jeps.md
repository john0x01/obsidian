# Java Evolution and JEPs

Java's release model changed more dramatically than the language itself around 2017: it went from slow, feature-driven mega-releases to a strict six-month train, with a subset of releases marked long-term-support. Layered on top is a governance process (JEPs) and a set of mechanisms — preview features, incubator modules, staged deprecation — that let the language evolve fast *without* betraying its backward-compatibility promise. Knowing how this machine works is what lets a senior engineer reason about what "modern Java" actually means for a given codebase.

## From Mega-Releases to the Six-Month Cadence

The old world shipped a major version every few years (Java 6 in 2006, 7 in 2011, 8 in 2014), so a delayed feature slipped the entire release. Since Java 9 (2017) the JDK ships on a **time-based cadence**: a new feature release every six months (March and September), and anything not ready simply rides the next train. This decouples *when we ship* from *what is ready*, trading a few large, risky releases for many small, predictable ones.

## LTS and What "Modern Java" Means

Because upgrading every six months is impractical for most production shops, certain releases are designated **Long-Term Support**: they receive years of updates and security patches from vendors (Oracle, and OpenJDK distributions like Eclipse Temurin, Amazon Corretto, Azul). The LTS line has been **17 (2021), 21 (2023), and 25 (2025)** — a roughly two-year LTS interval in recent years, after the earlier gap from 11 to 17. Non-LTS releases (18–20, 22–24) are production-quality but supported only until the next one lands.

The practical consequence: most teams live on LTS releases and skip the interim ones, so a feature that graduated during a non-LTS window only *reaches* them at the next LTS. "Modern Java" is therefore best read as "the current LTS" — 21 or 25 — which looks radically different from Java 8 despite the compatibility contract, because a decade of six-month features accumulated between them.

## The JEP Process

A **JEP** (JDK Enhancement Proposal) is the design document and unit of change for anything nontrivial in the JDK — a language feature, an API, a GC, a build change. Each JEP has an owner and moves through review roughly as *Draft → Candidate → Proposed to Target → Targeted → Integrated → Completed* (or is withdrawn/closed). JEPs are the engineering artifact; the formal *specification* still flows through the JCP (Java Community Process) as JSRs, which the language/VM spec changes feed into. The value of the process is transparency: the rationale, alternatives, and risks of a change are public before it ships.

## Preview and Incubator: Shipping Unfinished Work Safely

The hard problem is: how do you get real feedback on a big feature before committing to it *forever*? Two escape hatches, both deliberately gated so nothing unstable can leak into a production dependency by accident:

- **Preview features** are fully specified and fully implemented *language/VM* features that are explicitly **not permanent** — they can change or be removed in a later release. They compile and run only with `--enable-preview` (alongside a matching `--release`), and the flag is contagious: code built with it is marked so it won't silently run on a different version. Big features iterate through several preview rounds (pattern matching and virtual threads both spent multiple releases in preview) before finalizing.
- **Incubator modules** do the same for *APIs* rather than syntax: a non-final API ships in a separate `jdk.incubator.*` module you must explicitly require, signalling instability. The Vector API, for instance, has incubated across many releases; the Foreign Function & Memory API spent years in incubation/preview before finalizing. (Exact graduating versions vary — verify per feature rather than trusting memory.)

This is the compatibility promise operating in reverse: a feature isn't covered by the "never break it" contract *until* it exits preview/incubation, which is precisely what lets the language experiment.

## Deprecation for Removal

Even Java removes things, but slowly and loudly. Ordinary `@Deprecated` merely discourages use; `@Deprecated(forRemoval = true, since = "…")` signals a genuine plan to delete, upgrading compiler warnings and giving a migration window. The `jdeprscan` tool audits a codebase against deprecated APIs. Long-standing fixtures have gone this route — object finalization and the `SecurityManager`, for example, were deprecated for removal before disappearing — always with years of warning rather than a surprise break.

## The Compatibility Contract

Underpinning all of this are three distinct compatibility guarantees: **source** (old source still compiles), **binary** (old `.class` files still link), and **behavioral** (they do the same thing). The `--release N` flag lets a newer compiler target an older API level so you don't accidentally use too-new methods. This contract is *why* the fast cadence is safe: velocity on new features, near-absolute stability for existing code.

## See Also

- [[design-philosophy]]
- [[the-jvm-as-a-platform]]
- [[jpms-module-system]]
- [[records-and-sealed-types]]
- [[pattern-matching-and-switch]]
- [[virtual-threads]]
