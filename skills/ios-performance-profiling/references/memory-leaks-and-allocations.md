# Memory Leaks and Allocations

Use this reference when the task involves Allocations, Leaks, Memory Graph Debugger, VM Tracker, memory growth, retain cycles, caches, decoded images, or allocation churn.

This file helps the agent separate leak diagnosis from allocation-cost diagnosis. Do not treat every memory increase as a leak.

## Contents

- [Core model](#core-model)
- [Tool choice](#tool-choice)
- [Capability check](#capability-check)
- [Investigation workflow](#investigation-workflow)
- [Allocations](#allocations)
- [Leaks](#leaks)
- [Memory Graph Debugger](#memory-graph-debugger)
- [VM Tracker](#vm-tracker)
- [Caches and decoded images](#caches-and-decoded-images)
- [Allocation churn](#allocation-churn)
- [Retain cycles](#retain-cycles)
- [Fix direction](#fix-direction)
- [Validation](#validation)
- [Output notes](#output-notes)

## Core model

Memory problems are not all leaks.

Use these buckets:

- **Leak** — memory is no longer reachable by app logic, but remains allocated.
- **Retain cycle** — objects are still strongly referenced through an ownership cycle after the flow should be gone.
- **Unbounded growth** — data, caches, images, tasks, observers, or model graphs accumulate without a useful limit.
- **High steady-state footprint** — the app keeps a lot of memory alive, but the memory may be intentional and bounded.
- **High peak footprint** — memory spikes during a flow and may trigger pressure even if it later drops.
- **Allocation churn** — many short-lived allocations create CPU, ARC, and energy cost even if live memory stays flat.
- **VM growth** — mapped files, graphics memory, decoded image buffers, WebKit, video, maps, or ML memory grow outside the obvious object graph.

Say “memory growth” until the trace proves a leak or retain cycle.

## Tool choice

| Symptom or question | Primary tool | What it helps prove |
|---|---|---|
| Memory grows across repeated runs | Allocations | retained growth by type, generation, allocation site |
| A dismissed object stays alive | Memory Graph Debugger | ownership paths and retain cycles |
| Instruments reports unreachable leaked blocks | Leaks | unreachable leaked allocations |
| Resident memory grows but objects do not explain it | VM Tracker | VM regions, dirty memory, graphics/image/WebKit/video memory |
| High memory during image-heavy flows | Allocations + VM Tracker | decoded buffers, image caches, peak footprint |
| CPU cost from object creation | Allocations + Time Profiler | allocation churn and hot allocation sites |
| Production memory exits | MetricKit / Organizer | affected devices, releases, cohorts, and frequency |

Use local tools to explain cause. Use production metrics to confirm impact.

## Capability check

Before diagnosing memory, check:

- Is the scenario reproducible with the same navigation path and data set?
- Is profiling on a real device, especially for pressure, graphics, images, or production-like behavior?
- Is the build Release or release-like?
- Does memory grow after repeated runs, or only peak during one operation?
- Are Instruments traces, memory graph snapshots, MetricKit payloads, Organizer screenshots, or logs available?
- Does the app use large images, video, maps, web views, ML models, caches, databases, or offline storage?

If evidence is missing, provide a measurement plan instead of claiming the cause.

## Investigation workflow

1. Define the user-visible symptom: growth, crash, memory exit, slow screen, scrolling hitch, or retained screen.
2. Define the exact scenario that reproduces it.
3. Record baseline memory before the scenario.
4. Run the scenario once and record peak memory.
5. Return to the expected baseline state, such as dismissing the screen.
6. Wait briefly for normal cleanup.
7. Repeat the same scenario several times.
8. Check whether memory returns near baseline or grows by generation.
9. Identify retained object types, allocation sites, or VM regions.
10. Inspect ownership paths when objects remain alive.
11. Form one focused hypothesis.
12. Fix one ownership, cache, image, or allocation pattern.
13. Re-run the same scenario and compare.

Useful loop:

```text
baseline -> open feature -> exercise feature -> close feature -> wait -> repeat
```

If memory rises during the feature and returns after close, it may be peak footprint rather than a leak.

## Allocations

Use Allocations for object creation, retained growth, allocation sites, generations, and churn.

Inspect live bytes, persistent bytes, allocation count by type, allocation backtraces, generation growth, surviving objects after the flow ends, and short-lived allocations in hot paths. Ask whether the issue is retained memory, peak memory, or allocation rate.

Common findings:

- decoded images retained by image views, caches, or view models;
- arrays and dictionaries rebuilt repeatedly;
- attributed strings created during scrolling;
- JSON models retained after dismissal;
- tasks, observers, subscriptions, or callbacks kept alive;
- temporary buffers created per frame, per row, or per request.

Allocations can show growth and allocation sites. It does not by itself prove a retain cycle.

## Leaks

Use Leaks when looking for unreachable leaked allocations.

Inspect leaked allocation type, allocation backtrace, number and size of leaked blocks, and whether the leak repeats after each scenario iteration.

Important distinction:

Leaks can miss logical leaks and retain cycles because those objects are still reachable. A clean Leaks run does not prove the app has no memory problem.

If the screen, coordinator, view model, or service stays alive after dismissal, use Memory Graph Debugger.

## Memory Graph Debugger

Use Memory Graph Debugger when objects should have deallocated but remain alive.

Typical cases:

- a view controller remains after dismissal;
- a SwiftUI view model or observable model remains after leaving a screen;
- a coordinator, router, task, timer, subscription, callback, observer, or service retains a feature;
- a cache, singleton, static container, dependency container, or registry keeps feature data alive.

Workflow:

1. Navigate to the feature.
2. Leave the feature in the normal way.
3. Take a memory graph snapshot.
4. Search for the type that should be gone.
5. Inspect the strong reference path back to a root.
6. Fix the first incorrect ownership edge.
7. Repeat the same scenario and snapshot.

Inspect especially closure captures, delegates, timers, display links, NotificationCenter, Combine, async tasks, callbacks, coordinator ownership, singleton services, and global registries.

Prefer fixing the ownership model over adding weak references everywhere.

## VM Tracker

Use VM Tracker when resident memory or virtual memory growth is not explained by Swift or Objective-C objects.

Inspect dirty memory, resident memory, mapped files, image and graphics memory, Core Animation surfaces, WebKit, maps, video, database, ML regions, malloc regions, and growth by VM region.

VM Tracker is useful when images, graphics, WebKit, maps, video, ML, or mapped databases are involved.

Do not expect VM Tracker to show retain cycles. Use it to identify the growing region, then inspect the responsible feature.

## Caches and decoded images

Caches are not leaks by default. They become a memory problem when they are unbounded, duplicated, poorly evicted, or hold decoded data longer than useful.

Inspect cache key cardinality, cost and count limits, eviction policy, duplicate cache layers, full-size images retained for thumbnail UI, decoded buffers, aggressive prefetching, and memory warning handling.

Fix by setting cost/count limits, downsampling before display, avoiding full-resolution storage for thumbnail UI, canceling obsolete requests, avoiding duplicate decoded copies, shrinking caches on memory warnings, and prioritizing visible content over eager preloading.

For image-heavy scrolling, combine this reference with `animation-hitches-and-swiftui.md`.

## Allocation churn

Allocation churn can hurt CPU, scrolling, animation, ARC traffic, and energy even when live memory stays flat.

Look for churn in:

- SwiftUI `body` paths;
- row creation;
- text and date formatting;
- attributed string construction;
- JSON mapping;
- diff generation;
- image resizing or decoding;
- bridging between Swift and Objective-C;
- existential boxing or type erasure in hot paths;
- temporary arrays, dictionaries, closures, buffers, and tasks.

Fix by moving repeated work out of hot paths, caching stable derived values, reusing formatters or buffers where safe, reducing intermediate collections, avoiding per-frame/per-row construction, and removing unnecessary type erasure when evidence points to it.

Route to `swift-runtime-performance` when evidence points to ARC traffic, existentials, generics, dispatch, bridging, copy-on-write, or runtime-level costs.

## Retain cycles

Common retain-cycle sources:

- `self` captured strongly by escaping closures;
- timers or display links retaining targets or closures;
- Combine subscriptions retained by `self` while a closure captures `self`;
- async tasks captured and stored in a way that outlives the feature;
- strong delegates;
- parent and child coordinators retaining each other;
- services retaining callbacks that retain view models;
- singleton registries retaining feature objects.

Use weak captures where the closure must not prolong the owner lifetime. Strong captures are fine when the work must keep the object alive and the lifetime is bounded. Cancel tasks/subscriptions, invalidate timers/display links, remove child coordinators, and prefer explicit lifetime boundaries over relying only on weak captures.

Do not blindly add `[weak self]` everywhere. First understand the intended ownership.

## Fix direction

| Evidence | Likely fix direction |
|---|---|
| Retain path keeps dismissed feature alive | Break the incorrect strong ownership edge |
| Objects grow by generation | Release, evict, or bound the owner |
| Peak memory spikes then falls | Downsample, stream, batch, or reduce temporary copies |
| Allocation rate is high but live memory is flat | Remove repeated allocations from the hot path |
| VM region grows | Identify subsystem: images, graphics, WebKit, maps, video, database, ML |
| Cache dominates memory | Add limits, eviction, smaller representations, or memory-warning handling |
| Production memory exits | Reproduce on affected device class and reduce peak or steady footprint |

Prefer one focused fix at a time. Re-measure after each fix.

## Validation

A memory fix is not validated until the same scenario is re-run with comparable conditions.

Good validation includes:

- same device class or affected device class;
- same OS version when possible;
- same build configuration;
- same user flow and data set;
- repeated scenario iterations;
- baseline, peak, and post-scenario memory;
- retained instance count for the suspected type;
- before/after screenshots or exported trace summaries;
- MetricKit or Organizer confirmation for production regressions.

Use this structure:

```text
Scenario:
Device / OS / build:
Baseline memory:
Peak memory:
Post-scenario memory:
Repeated iterations:
Retained instances:
Strongest signal:
Conclusion:
```

Avoid claiming success from one run, Debug-only results, Simulator-only pressure testing, a clean Leaks report without checking retain paths, or deinit logs without memory evidence.

## Output notes

When responding to a memory profiling task, include:

1. The memory problem type: leak, retain cycle, unbounded growth, high peak, high steady-state footprint, allocation churn, or VM growth.
2. The best primary tool and why.
3. The exact scenario to reproduce.
4. What to inspect in the trace or memory graph.
5. The strongest current evidence.
6. What is not proven yet.
7. One focused fix or next inspection step.
8. A before/after validation plan.

Use cautious language when evidence is incomplete:

- “This suggests retained growth, but does not yet prove a retain cycle.”
- “A clean Leaks run would not rule this out; inspect the Memory Graph.”
- “This looks like allocation churn rather than a leak if live memory returns to baseline.”
- “VM Tracker is needed if object allocations do not explain resident memory growth.”
