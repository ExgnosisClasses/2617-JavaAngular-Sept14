# Shenandoah and Epsilon: A Conceptual Overview

Two collectors at opposite ends of the spectrum: one that does as much concurrent work as possible, and one that does no work at all.

## Shenandoah

### What it is

Shenandoah is a low-pause collector with the same goal as ZGC: keep pauses short and independent of heap size by compacting the heap while the application runs. It was developed by Red Hat, entered OpenJDK as experimental in JDK 12 (JEP 189), and became production ready in JDK 15. It ships in most OpenJDK distributions (Red Hat, Adoptium, Amazon Corretto, Azul, Microsoft) but is not included in Oracle's JDK builds.

### How it works

Like G1, Shenandoah divides the heap into regions and evacuates live objects from garbage-heavy regions into empty ones. The difference is that evacuation runs concurrently.

Its original mechanism was a **Brooks forwarding pointer**: every object carries an extra word that points either to itself or, once it has been copied, to its new location. Every access to an object first reads that word, so the application always finds the current copy. When the collector moves an object, it copies it and atomically swaps the old copy's forwarding pointer to the new address; any thread still holding the old reference follows the pointer and lands in the right place. Since JDK 17 the forwarding pointer is stored in the object header instead of a separate word, removing the per-object memory overhead.

Where ZGC puts its metadata in the *reference* (colored pointers) and checks it on *load*, Shenandoah puts its metadata in the *object* and checks it on *access*, particularly writes and comparisons. The two collectors reach similar results by different routes.

A cycle has the familiar shape: a brief pause to scan roots, concurrent marking, a brief pause to finish marking and pick the collection set, concurrent evacuation, and a concurrent pass to update stale references. Pauses are typically in the low milliseconds.

### Generational mode

Shenandoah was non-generational through JDK 23, scanning the whole heap each cycle. Generational Shenandoah arrived as experimental in JDK 24 (JEP 404) and follows the same reasoning as generational ZGC: collect the young objects often and cheaply, the old ones rarely.

### Costs and fit

- Barrier overhead of a few percent throughput, comparable to ZGC.
- Needs heap headroom; if allocation outpaces concurrent reclamation it falls back to a full stop-the-world collection, which is the pause it exists to avoid.
- Works on 32-bit and 64-bit platforms, unlike ZGC.
- Choose it for the same reasons as ZGC, especially on distributions where it is the better-supported option or when running on hardware ZGC does not cover.

## Epsilon

### What it is

Epsilon is a no-op collector, added as experimental in JDK 11 (JEP 318). It handles memory allocation and nothing else. When the heap fills, the JVM shuts down with an `OutOfMemoryError`. Enable it with `-XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC`.

### Why it exists

- **Performance testing.** Measuring an application with garbage collection removed shows how much of its behaviour is GC cost versus its own work, and gives a baseline to compare real collectors against.
- **Allocation-rate testing.** Run a workload with a fixed heap and see how long until it dies; that tells you how much garbage it produces.
- **Very short-lived jobs.** A process that runs for a few seconds and exits may never need a collection; Epsilon removes all GC setup and barrier overhead.
- **Latency-critical code that allocates almost nothing.** Some trading or embedded systems are written to avoid allocation entirely after startup; Epsilon guarantees a collection never happens.
- **JVM development.** A minimal collector is useful for testing the allocation and interface code paths in isolation.

### What it is not

Epsilon is not a production collector for ordinary applications. Any program that allocates steadily will exhaust the heap and die. It is a diagnostic and specialist tool.

## Side by side

| | Shenandoah | Epsilon |
|---|---|---|
| Goal | Low pauses, any heap size | Zero GC overhead |
| Reclaims memory | Yes, concurrently | Never |
| Key mechanism | Forwarding pointer per object, access barriers | None |
| Status in JDK 21 | Production, non-generational | Experimental |
| Typical use | Latency-sensitive services on OpenJDK builds | Benchmarks, tests, short-lived or allocation-free processes |

## Further reading

- JEP 189, Shenandoah: https://openjdk.org/jeps/189
- JEP 404, Generational Shenandoah: https://openjdk.org/jeps/404
- Shenandoah project wiki: https://wiki.openjdk.org/display/shenandoah
- JEP 318, Epsilon: https://openjdk.org/jeps/318
