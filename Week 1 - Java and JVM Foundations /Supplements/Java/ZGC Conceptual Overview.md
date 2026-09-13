# ZGC: A Conceptual Overview

## The problem ZGC solves

Every earlier collector had the same weak point: moving objects. Marking can run alongside the application (CMS proved that), but compacting means changing an object's address, and if the application is running it might read the old address a nanosecond after the object moved. So G1 and its predecessors stop the world to relocate, and the pause grows with the amount of live data being moved. On a 100 GB heap that can be seconds.

ZGC's goal is pauses that stay under a millisecond regardless of heap size, from a few hundred megabytes to many terabytes. To get there, it had to make relocation concurrent, which no production HotSpot collector had done.

## The core idea: fix references on the way in, not all at once

Instead of stopping everything to update every reference to a moved object, ZGC lets the application keep running and intercepts each reference *as it is loaded*. If the reference points to a stale location, ZGC repairs it right then, transparently, before the application code sees it. The application never observes a bad pointer because it never gets to use one.

Two mechanisms make this work.

### Colored pointers

A 64-bit reference has far more bits than needed to address memory, so ZGC uses a few of the spare high bits as metadata stamped directly into the pointer. These "colors" record the reference's state:

- Has this object been marked in the current cycle?
- Has it been relocated?
- Has this pointer been updated since the last cycle?

The object itself is untouched; the information travels with the reference. Think of it as a postmark on an envelope rather than a note in the file it points to.

### Load barriers

The JIT inserts a small check into every place the application loads a reference from the heap.

- **Fast path:** a single test of the color bits. If the color says "current and correct," proceed with no further cost.
- **Slow path:** the barrier may mark the object, look up its new address in a forwarding table, patch the reference in place so the next load takes the fast path, and hand back the correct pointer.

The barrier is on *loads* specifically because that is the one moment the collector can guarantee it sees a reference before the application uses it.

## A collection cycle

1. **Pause: mark start** (microseconds). Flip the "good" color for this cycle and scan thread stacks for roots.
2. **Concurrent mark.** Trace the object graph. Application threads that load an unmarked reference mark it via the barrier, so the two cooperate.
3. **Pause: mark end** (microseconds). Confirm marking is complete.
4. **Concurrent prepare.** Choose the relocation set: the regions with the most garbage, where moving a few live objects frees the most space.
5. **Pause: relocate start** (microseconds). Flip colors again and relocate the root-referenced objects.
6. **Concurrent relocate.** Copy live objects out of the chosen regions into fresh ones, recording each move in a per-region forwarding table. Application threads that hit a moved object through the barrier relocate or remap it themselves. Once a region is empty it is freed immediately, even though stale references to it may still exist elsewhere on the heap.
7. **Remap.** Stale references left over are fixed lazily by the barrier as they are loaded, and any that are never loaded are caught by the marking phase of the *next* cycle, which walks everything anyway. Remapping is folded into the following mark and costs no separate pass.

The pauses exist only to flip state and scan roots. Everything proportional to heap size runs concurrently, which is why pause time is flat as the heap grows.

## Generational ZGC (JDK 21 onward)

The original ZGC treated the heap as one flat space and re-marked everything every cycle. That wastes effort on the long-lived majority and forces cycles to run often enough to keep up with allocation.

Generational ZGC (opt-in in JDK 21 with `-XX:+ZGenerational`, default in JDK 23, the only mode from JDK 24) adds young and old generations so most cycles only scan recently allocated objects. This cuts CPU overhead and lets ZGC keep up with much higher allocation rates. The pointer coloring scheme was reworked to carry the extra generational state and remembered-set information.

## What it costs

- **Throughput.** The barrier adds a small per-load cost, typically a few percent compared with Parallel GC.
- **Headroom.** Because ZGC frees memory concurrently rather than in a pause, allocation can outrun reclamation if the heap is sized too tightly. The result is an "allocation stall" while a thread waits for a cycle to finish.
- **Platform.** 64-bit only, since the colored-pointer trick needs the spare bits.

## Where it fits

| Collector | Pause goal | Trade-off |
|---|---|---|
| G1 (default) | Around 200 ms | Good throughput, general purpose |
| ZGC | Under 1 ms, independent of heap size | A few percent throughput, needs heap headroom |

Choose ZGC when tail latency matters: trading systems, request-serving applications with strict SLAs, or any workload with a heap large enough that G1's pauses become visible.

## One-sentence summary

ZGC never stops the world to move objects because it tags every reference with its state and checks the tag on every load, fixing stale references on demand instead of all at once.

## Further reading

- JEP 333, ZGC experimental (JDK 11): https://openjdk.org/jeps/333
- JEP 377, ZGC production ready (JDK 15): https://openjdk.org/jeps/377
- JEP 439, Generational ZGC (JDK 21): https://openjdk.org/jeps/439
- ZGC project wiki: https://wiki.openjdk.org/display/zgc
