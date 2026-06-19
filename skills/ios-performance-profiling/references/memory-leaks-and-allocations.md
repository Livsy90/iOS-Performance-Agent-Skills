# Memory Leaks and Allocations

Use this reference when the task involves Allocations, Leaks, Memory Graph Debugger, VM Tracker, memory growth, retain cycles, caches, decoded images, or allocation churn.

This file helps the agent separate leak diagnosis from allocation-cost diagnosis. Do not treat every memory increase as a leak.

## Scope Boundary

This reference covers:

* leak vs retained-growth vs peak-memory diagnosis;
* Allocations, Leaks, Memory Graph Debugger, VM Tracker, MetricKit, and Organizer routing;
* retain cycles, logical leaks, caches, decoded images, allocation churn, and VM growth;
* ownership and lifetime investigation;
* scenario-based before/after validation.

This reference does not cover:

* general ARC/runtime optimization without memory evidence;
* SwiftUI invalidation or scrolling hitches unless allocation churn or memory pressure overlaps the UI symptom;
* low-level malloc allocator internals;
* detailed image pipeline architecture beyond memory diagnosis;
* production crash triage except memory exits and memory-pressure signals;
* generic cache architecture unless it affects measured memory behavior.

## Contents

* [Core Model](#core-model)
* [Tool Choice](#tool-choice)
* [Capability Check](#capability-check)
* [Investigation Workflow](#investigation-workflow)
* [Allocations](#allocations)
* [Leaks](#leaks)
* [Memory Graph Debugger](#memory-graph-debugger)
* [VM Tracker](#vm-tracker)
* [Caches and Decoded Images](#caches-and-decoded-images)
* [Allocation Churn](#allocation-churn)
* [Retain Cycles](#retain-cycles)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [Fix Direction](#fix-direction)
* [Validation](#validation)
* [Output Notes](#output-notes)

## Core Model

Memory problems are not all leaks.

Use these buckets:

* **Leak** — allocated memory is unreachable and cannot be freed normally.
* **Logical leak / retained growth** — objects are still reachable, but should no longer be alive for the completed scenario.
* **Retain cycle** — objects are still strongly referenced through an ownership cycle after the flow should be gone.
* **Unbounded growth** — data, caches, images, tasks, observers, or model graphs accumulate without a useful limit.
* **High steady-state footprint** — the app keeps a lot of memory alive, but the memory may be intentional and bounded.
* **High peak footprint** — memory spikes during a flow and may trigger pressure even if it later drops.
* **Allocation churn** — many short-lived allocations create CPU, ARC, and energy cost even if live memory stays flat.
* **VM growth** — mapped files, graphics memory, decoded image buffers, WebKit, video, maps, or ML memory grow outside the obvious object graph.

Say “memory growth” until the trace proves a leak, retain cycle, or specific ownership problem.

A leak detector looks for unreachable allocated memory. A retain cycle or logical leak can remain reachable from a root object, singleton, task, cache, notification center, callback, or closure chain. In that case, the issue is incorrect ownership or lifetime, not necessarily unreachable memory.

## Tool Choice

| Symptom or question                                 | Primary tool                | What it helps prove                                          |
| --------------------------------------------------- | --------------------------- | ------------------------------------------------------------ |
| Memory grows across repeated runs                   | Allocations                 | Retained growth by type, generation, allocation site         |
| A dismissed object stays alive                      | Memory Graph Debugger       | Ownership paths and retain cycles                            |
| Instruments reports unreachable leaked blocks       | Leaks                       | Unreachable leaked allocations                               |
| Resident memory grows but objects do not explain it | VM Tracker                  | VM regions, dirty memory, graphics/image/WebKit/video memory |
| High memory during image-heavy flows                | Allocations + VM Tracker    | Decoded buffers, image caches, peak footprint                |
| CPU cost from object creation                       | Allocations + Time Profiler | Allocation churn and hot allocation sites                    |
| Production memory exits                             | MetricKit / Organizer       | Affected devices, releases, cohorts, and frequency           |

Use local tools to explain cause. Use production metrics to confirm impact.

## Capability Check

Before diagnosing memory, check:

* Is the scenario reproducible with the same navigation path and data set?
* Is profiling on a real device, especially for pressure, graphics, images, or production-like behavior?
* Is the build Release or release-like?
* Does memory grow after repeated runs, or only peak during one operation?
* Are Instruments traces, memory graph snapshots, MetricKit payloads, Organizer screenshots, or logs available?
* Does the app use large images, video, maps, web views, ML models, caches, databases, or offline storage?
* Is the issue local, CI-detected, production-only, or user-reported?
* Is the symptom memory growth, memory exit, UI hitch, slow flow, or retained screen?

If evidence is missing, provide a measurement plan instead of claiming the cause.

## Investigation Workflow

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

```text id="xz4a9k"
baseline -> open feature -> exercise feature -> close feature -> wait -> repeat
```

If memory rises during the feature and returns after close, it may be peak footprint rather than a leak.

If memory rises slightly and then stabilizes across repeated runs, it may be cache warmup or steady-state growth rather than a leak. Validate whether the plateau is bounded and acceptable for affected devices.

## Allocations

Use Allocations for object creation, retained growth, allocation sites, generations, and churn.

Inspect:

* live bytes;
* persistent bytes;
* allocation count by type;
* allocation backtraces;
* generation growth;
* surviving objects after the flow ends;
* short-lived allocations in hot paths;
* peak memory during the flow;
* allocation rate during scrolling, animation, parsing, or mapping.

Ask whether the issue is retained memory, peak memory, or allocation rate.

Common findings:

* decoded images retained by image views, caches, or view models;
* arrays and dictionaries rebuilt repeatedly;
* attributed strings created during scrolling;
* JSON models retained after dismissal;
* tasks, observers, subscriptions, or callbacks kept alive;
* temporary buffers created per frame, per row, or per request;
* repeated formatter, decoder, or layout-related object creation.

Allocations can show growth and allocation sites. It does not by itself prove a retain cycle.

## Leaks

Use Leaks when looking for unreachable leaked allocations.

Inspect:

* leaked allocation type;
* allocation backtrace;
* number and size of leaked blocks;
* whether the leak repeats after each scenario iteration;
* whether leaked blocks map to app code, framework code, or a vendor SDK.

Important distinction:

Leaks can miss logical leaks and retain cycles because those objects are still reachable. A clean Leaks run does not prove the app has no memory problem.

If the screen, coordinator, view model, observable model, service, or task stays alive after dismissal, use Memory Graph Debugger.

## Memory Graph Debugger

Use Memory Graph Debugger when objects should have deallocated but remain alive.

Typical cases:

* a view controller remains after dismissal;
* a SwiftUI view model or observable model remains after leaving a screen;
* a coordinator, router, task, timer, subscription, callback, observer, or service retains a feature;
* a cache, singleton, static container, dependency container, registry, or global store keeps feature data alive.

Workflow:

1. Navigate to the feature.
2. Leave the feature in the normal way.
3. Wait until expected cleanup should have happened.
4. Take a memory graph snapshot.
5. Search for the type that should be gone.
6. Inspect the strong reference path back to a root.
7. Fix the first incorrect ownership edge.
8. Repeat the same scenario and snapshot.

Take the snapshot after the expected release point, not while the feature is still legitimately visible or an operation is intentionally in progress.

For SwiftUI, remember that `View` values are transient. Investigate retained reference types: `ObservableObject`, `@Observable` models, coordinators, services, tasks, Combine subscriptions, timers, delegates, caches, and app-owned reference state. Do not expect a SwiftUI `View` struct itself to behave like a retained view controller.

Inspect especially:

* closure captures;
* delegates;
* timers;
* display links;
* NotificationCenter observers;
* KVO;
* Combine subscriptions;
* async tasks;
* async sequences;
* callbacks;
* coordinator ownership;
* singleton services;
* global registries;
* dependency containers;
* caches.

Prefer fixing the ownership model over adding weak references everywhere.

Use deinit logs only as supporting evidence. Missing `deinit` suggests retention, but Memory Graph or retained instance counts should identify why.

## VM Tracker

Use VM Tracker when resident memory or virtual memory growth is not explained by Swift or Objective-C objects.

Inspect:

* dirty memory;
* resident memory;
* mapped files;
* image and graphics memory;
* Core Animation surfaces;
* WebKit;
* maps;
* video;
* database regions;
* ML regions;
* malloc regions;
* growth by VM region.

VM Tracker is useful when images, graphics, WebKit, maps, video, ML, or mapped databases are involved.

Do not treat virtual size alone as user-impacting memory. Prioritize resident and dirty memory, memory pressure, and growth by region.

Do not expect VM Tracker to show retain cycles. Use it to identify the growing region, then inspect the responsible feature.

## Caches and Decoded Images

Caches are not leaks by default. They become a memory problem when they are unbounded, duplicated, poorly evicted, or hold decoded data longer than useful.

Inspect:

* cache key cardinality;
* cost and count limits;
* eviction policy;
* duplicate cache layers;
* full-size images retained for thumbnail UI;
* decoded buffers;
* aggressive prefetching;
* memory warning handling;
* whether cached entries are tied to feature lifetime or app lifetime;
* whether cache growth plateaus or keeps increasing.

`NSCache` is not a precise memory budget. It can evict under pressure, but cost and count limits are hints, not strict guarantees. Validate actual resident memory, dirty memory, and decoded buffer behavior.

A compressed image file size is not the decoded memory cost. Decoded memory depends on pixel dimensions, scale, pixel format, and number of retained decoded copies.

Fix by:

* setting cost/count limits;
* downsampling before display;
* avoiding full-resolution storage for thumbnail UI;
* canceling obsolete requests;
* avoiding duplicate decoded copies;
* shrinking caches on memory pressure;
* prioritizing visible content over eager preloading.

Check whether the app responds to memory pressure by clearing noncritical caches, canceling prefetches, releasing decoded images, and reducing background work. Do not use memory warnings as the only eviction mechanism; prefer bounded caches first.

For image-heavy scrolling, combine this reference with `animation-hitches-and-swiftui.md`.

## Allocation Churn

Allocation churn can hurt CPU, scrolling, animation, ARC traffic, and energy even when live memory stays flat.

Look for churn in:

* SwiftUI `body` paths;
* row creation;
* text and date formatting;
* attributed string construction;
* JSON mapping;
* diff generation;
* image resizing or decoding;
* bridging between Swift and Objective-C;
* existential boxing or type erasure in hot paths;
* temporary arrays, dictionaries, closures, buffers, and tasks;
* repeated creation of formatters, decoders, layout helpers, or view models.

Fix by:

* moving repeated work out of hot paths;
* caching stable derived values;
* reusing formatters or buffers where safe;
* reducing intermediate collections;
* avoiding per-frame or per-row construction;
* removing unnecessary type erasure when evidence points to it;
* batching work to reduce repeated allocation waves.

In mixed Swift/Objective-C code, autoreleased objects and bridging can create short-lived memory spikes. Consider autorelease pool boundaries around batch processing when evidence points to autorelease buildup.

Route to `swift-runtime-performance` when evidence points to ARC traffic, existentials, generics, dispatch, bridging, copy-on-write, or runtime-level costs.

## Retain Cycles

Common retain-cycle sources:

* `self` captured strongly by escaping closures;
* timers or display links retaining targets or closures;
* Combine subscriptions retained by `self` while a closure captures `self`;
* async tasks captured and stored in a way that outlives the feature;
* strong delegates;
* parent and child coordinators retaining each other;
* services retaining callbacks that retain view models;
* singleton registries retaining feature objects;
* notification, KVO, or async sequence observers that are never removed or cancelled.

Use weak captures where the closure must not prolong the owner lifetime. Strong captures are fine when the work must keep the object alive and the lifetime is bounded.

A `Task` can keep captured objects alive until it completes or is cancelled. A task stored by an owner can also create an `owner -> task -> closure -> owner` cycle. Check whether tasks are cancelled on dismissal and whether long-lived tasks intentionally own their captured state.

`Task { [weak self] in ... }` is not enough if the task later strongly unwraps `self` for a long-running operation. Validate lifetime with Memory Graph or retained instance counts.

For Combine, inspect who owns the `AnyCancellable`. A common cycle is:

```text id="22ghh6"
self -> cancellables -> subscription -> closure -> self
```

Break it with weak capture, cancellation, or moving ownership to a shorter-lived object.

Check observers, KVO, async sequences, and notification streams. Token-based observers, `for await` loops, and `AsyncStream` continuations can keep owners alive if they are not cancelled, removed, or finished.

Cancel tasks/subscriptions, invalidate timers/display links, remove observers, remove child coordinators, and prefer explicit lifetime boundaries over relying only on weak captures.

Do not blindly add `[weak self]` everywhere. First understand the intended ownership.

## What the Agent Can Inspect

When repository access is available, inspect concrete memory ownership and allocation risks instead of giving generic advice.

Search for closure captures:

```sh id="555nsj"
rg "\[weak self\]|\[unowned self\]|self\." .
```

Search for long-lived callbacks and subscriptions:

```sh id="s79cbf"
rg "sink\(|assign\(|AnyCancellable|cancellables|NotificationCenter|addObserver|Timer|CADisplayLink|KVO|observe\(" .
```

Search for tasks and async streams:

```sh id="lusvn3"
rg "Task\s*\{|Task\(|Task\.detached|AsyncStream|AsyncThrowingStream|for await|continuation|withTaskGroup" .
```

Search for caches, singletons, and long-lived stores:

```sh id="pdi0ft"
rg "NSCache|cache|Cache|shared|static let|singleton|registry|store|pool|buffer|history" .
```

Search for image memory risks:

```sh id="6suhqy"
rg "UIImage|CGImage|Data\(|contentsOf:|jpegData|pngData|UIGraphicsImageRenderer|resize|thumbnail|downsample|decode" .
```

Search for large collections and retained models:

```sh id="zxeqnh"
rg "append\(|removeAll|Dictionary|Array|Set|models|items|history|buffer|queue" .
```

Search for cleanup and lifetime boundaries:

```sh id="ifgujv"
rg "deinit|cancel\(|invalidate\(|removeObserver|finish\(|close\(|stop\(|tearDown|cleanup|dispose" .
```

Search for SwiftUI retained reference state:

```sh id="xamca2"
rg "@StateObject|@Observable|ObservableObject|@EnvironmentObject|\.environmentObject\(|\.environment\(" .
```

Use matches as leads, not proof. Confirm with Allocations, Memory Graph, VM Tracker, or production memory evidence.

The agent can:

* classify memory growth as leak, logical leak, retain cycle, unbounded growth, high peak, high steady-state footprint, allocation churn, or VM growth;
* propose the right profiling tool for the symptom;
* inspect ownership paths and long-lived references;
* identify likely cache, image, task, subscription, observer, or singleton retention risks;
* propose a repeatable scenario and before/after validation plan.

The agent cannot reliably:

* prove a leak from source search alone;
* prove production impact from one local memory graph;
* treat every cache as a bug;
* treat every strong capture as a leak;
* promise a fixed memory reduction without measurement;
* replace memory profiling with deinit logs alone.

## Fix Direction

Connect the fix to the evidence.

| Evidence                                        | Likely fix direction                                                                   |
| ----------------------------------------------- | -------------------------------------------------------------------------------------- |
| Retain path keeps dismissed feature alive       | Break the incorrect strong ownership edge                                              |
| Logical leak without a cycle                    | Move ownership to the correct lifetime, clear registry/cache/store, or remove callback |
| Objects grow by generation                      | Release, evict, or bound the owner                                                     |
| Peak memory spikes then falls                   | Downsample, stream, batch, or reduce temporary copies                                  |
| Allocation rate is high but live memory is flat | Remove repeated allocations from the hot path                                          |
| VM region grows                                 | Identify subsystem: images, graphics, WebKit, maps, video, database, ML                |
| Cache dominates memory                          | Add limits, eviction, smaller representations, or memory-pressure handling             |
| Production memory exits                         | Reproduce on affected device class and reduce peak or steady footprint                 |

Prefer one focused fix at a time. Re-measure after each fix.

Do not use weak references to hide missing cancellation. If the operation should stop when the feature closes, cancel it. If it should continue, move ownership to an object with the correct longer lifetime.

## Validation

A memory fix is not validated until the same scenario is re-run with comparable conditions.

Good validation includes:

* same device class or affected device class;
* same OS version when possible;
* same build configuration;
* same user flow and data set;
* repeated scenario iterations;
* baseline, peak, and post-scenario memory;
* retained instance count for the suspected type;
* ownership path no longer retaining the dismissed feature;
* VM region growth reduced when VM Tracker was the evidence;
* allocation rate reduced when churn was the evidence;
* before/after screenshots or exported trace summaries;
* MetricKit or Organizer confirmation for production regressions.

Use this structure:

```text id="1ic69k"
Scenario:
Device / OS / build:
Baseline memory:
Peak memory:
Post-scenario memory:
Repeated iterations:
Retained instances:
VM region / allocation type:
Strongest signal:
Conclusion:
```

Also validate that the fix did not replace retained growth with repeated reallocation churn, excessive cache misses, image flicker, network re-fetching, worse scrolling performance, or worse first-use latency.

Avoid claiming success from one run, Debug-only results, Simulator-only pressure testing, a clean Leaks report without checking retain paths, or deinit logs without memory evidence.

## Output Notes

When responding to a memory profiling task, include:

1. The memory problem type: leak, logical leak, retain cycle, unbounded growth, high peak, high steady-state footprint, allocation churn, or VM growth.
2. The best primary tool and why.
3. The exact scenario to reproduce.
4. What to inspect in the trace or memory graph.
5. The strongest current evidence.
6. What is not proven yet.
7. One focused fix or next inspection step.
8. A before/after validation plan.

Use cautious language when evidence is incomplete:

* “This suggests retained growth, but does not yet prove a retain cycle.”
* “A clean Leaks run would not rule this out; inspect the Memory Graph.”
* “This looks like allocation churn rather than a leak if live memory returns to baseline.”
* “VM Tracker is needed if object allocations do not explain resident memory growth.”
* “Missing `deinit` suggests retention, but the ownership path still needs to be identified.”
