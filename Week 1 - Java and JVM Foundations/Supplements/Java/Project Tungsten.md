# Project Tungsten: A Conceptual Overview

## What it is

Project Tungsten was the initiative inside Apache Spark, starting with Spark 1.4 in 2015 and largely complete by Spark 2.0 in 2016, to push Spark's performance closer to the limits of the hardware. Its premise was that Spark's bottleneck had shifted from disk and network I/O to CPU and memory efficiency, and that the biggest gains would come from bypassing the JVM's default object model rather than tuning it.

Tungsten is not a separate library or API. It is the name for a set of internal changes to Spark SQL and the DataFrame/Dataset engine that are on by default and invisible to application code.

## The problem it solved

A Spark job on the JVM paid three hidden costs:

- **Object overhead.** A Java `String` of four characters takes roughly 48 bytes: object header, hash field, array reference, and a separate `char[]` with its own header. A row stored as a `Row` of boxed `Integer` and `String` objects carried several times its raw data size in headers and pointers.
- **Garbage collection.** Billions of small, short-lived row objects put constant pressure on the collector. GC pauses on large executors could dominate job time.
- **Cache misses.** Objects scattered across the heap defeat CPU caches; a sort that follows pointers to compare keys spends most of its time waiting for memory.

## The three pillars

### 1. Explicit memory management and binary processing

Tungsten replaced boxed row objects with **UnsafeRow**, a compact binary format that stores a row as a contiguous block of bytes: a null bitmap, fixed-width fields inline, and variable-width fields (strings, arrays) as offset-and-length pairs into the same block. Fields are read directly from raw memory with no object allocation.

These rows live in memory that Spark manages itself, either off-heap (allocated natively, outside the GC's view, enabled with `spark.memory.offHeap.enabled`) or in large on-heap `long[]` pages that the GC sees as a few big arrays rather than millions of small objects. Either way, the collector has almost nothing to trace. Spark's own **MemoryManager** tracks execution and storage memory in pages, spilling to disk when it runs out, instead of relying on the GC to free space.

Reading and writing that raw memory used `sun.misc.Unsafe`, the same unsupported API that JEP 471 is now deprecating. This is the classic example of why Unsafe existed: high-performance frameworks needed off-heap access before the platform offered a safe way to do it. The Foreign Function and Memory API's `MemorySegment` is the supported replacement for exactly this kind of work.

### 2. Cache-aware computation

Sorting and hashing were redesigned around how CPUs actually fetch memory. Tungsten's sort keeps a compact array of pointers each paired with a **key prefix** (the first few bytes of the sort key) so most comparisons resolve from the prefix in cache without dereferencing the pointer to the full row. Aggregation hash maps store keys and values in the same binary pages. The goal is sequential memory access and fewer cache misses rather than fewer instructions.

### 3. Code generation

Instead of interpreting a query plan through a tree of generic operators with virtual calls at every step, Spark generates Java source for the specific query at runtime, compiles it with the Janino compiler, and runs it. **Whole-stage code generation** (Spark 2.0) fuses an entire chain of operators into a single tight loop, the way a hand-written program would look, eliminating per-row function calls and intermediate row objects between stages. The JIT then compiles that loop to machine code. Expression evaluation, serialization, and even hash computation are all generated per query.

## Where it connects to the JVM memory model

| JVM concept | Tungsten's use of it |
|---|---|
| Object headers and boxing | Avoided entirely by the UnsafeRow binary format |
| Garbage collection pressure | Reduced by storing data in a few large pages instead of many small objects |
| Off-heap memory | Optional home for execution and storage memory, outside the GC |
| `sun.misc.Unsafe` | The tool used for raw memory access, now being replaced by the FFM API |
| CPU caches | Targeted directly by prefix-sorting and contiguous layouts |
| JIT compilation | Fed fused, generated loops that optimize far better than interpreted operator trees |

## Why it matters to a Java developer

Tungsten is a case study in the trade-off between the JVM's safety and convenience and raw performance. Spark chose to take over memory layout, allocation, and lifetime from the JVM because the default object model cost too much at scale. Every technique it used (binary rows, arena-style pages, explicit off-heap memory, generated code) is one the JVM has since moved toward supporting properly, with the FFM API and Project Valhalla's value objects being the platform-level answers to the same problems.

## Further reading

- Databricks announcement, "Project Tungsten: Bringing Apache Spark Closer to Bare Metal" (2015): https://www.databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html
- Databricks, "Apache Spark as a Compiler: Joining a Billion Rows per Second on a Laptop" (whole-stage code generation, 2016): https://www.databricks.com/blog/2016/05/23/apache-spark-as-a-compiler-joining-a-billion-rows-per-second-on-a-laptop.html
- Spark configuration reference (memory settings): https://spark.apache.org/docs/latest/configuration.html#memory-management
- Spark tuning guide: https://spark.apache.org/docs/latest/tuning.html
