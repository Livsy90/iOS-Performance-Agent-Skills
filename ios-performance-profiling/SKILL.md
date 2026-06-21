---
name: ios-performance-profiling
description: Use this skill when choosing, running, or interpreting iOS performance profiling workflows, including Instruments traces, signposts, XCTest metrics, MetricKit, Xcode Organizer, hangs, hitches, CPU, allocations, memory graphs, disk I/O, networking, power, or production performance signals. Do not use it as the deep domain skill for launch, SwiftUI, concurrency, perceived performance, or runtime issues unless the task is specifically about measurement, tool selection, trace interpretation, or verification.
---

# iOS Performance Profiling

## Purpose

Use this skill to choose the right profiling workflow, gather evidence, interpret performance signals, and recommend validation before claiming that an optimization worked.

This skill is a profiling router and evidence workflow. It should not replace more specific skills for launch performance, SwiftUI performance, Swift Concurrency performance, perceived performance, or Swift runtime costs.

## When to use this skill

Use this skill when the task involves:

- choosing an Instruments template or profiling workflow;
- interpreting traces, screenshots, XCTest metrics, MetricKit payloads, Organizer data, logs, or signposts;
- diagnosing hangs, animation hitches, CPU spikes, memory growth, leaks, disk I/O, network latency, power usage, or production regressions;
- designing a before/after measurement plan;
- adding signposts or performance tests;
- checking whether a proposed optimization is supported by evidence.

## When not to use this skill

Do not use this skill as the primary skill for:

- app startup architecture or launch-critical work unless the task asks how to profile, measure, or verify launch performance;
- SwiftUI invalidation, identity, layout, or scrolling fixes unless the task asks which profiling evidence to collect;
- Swift Concurrency design or actor isolation unless the task asks how to profile task behavior, actor hopping, or executor-related latency;
- perceived performance, loading states, skeletons, optimistic UI, or feedback design unless the task asks how to validate perceived latency;
- Swift runtime, ARC, allocation, existential, generic, dispatch, or linking costs unless the task asks how to measure them.

Prefer the more specific skill when the user already knows the domain and needs a fix rather than a measurement workflow.

## Core principle

Evidence before optimization.

Use this loop:

```text
Symptom -> reproducible scenario -> correct tool -> trace or metric -> hypothesis -> focused fix -> re-measure
```

Do not claim that a change improved performance unless there is a validation path. If evidence is missing, say what is not proven yet and what should be measured next.

## Capability check

Before recommending or running a real profiling workflow, check what is actually available.

Ask or infer:

- Is there a buildable Xcode project, workspace, scheme, and target?
- Is profiling possible on a real device, or only in Simulator?
- Is a Release or release-like configuration available?
- Is the scenario reproducible with stable data and stable app state?
- Are Instruments traces, screenshots, MetricKit payloads, Organizer screenshots, logs, or XCTest results available?
- Can traces or reports be shared as artifacts?
- Is the task asking for a profiling plan, a code review, or interpretation of existing evidence?

If tooling or artifacts are unavailable, provide a measurement plan instead of pretending to have profiled the app.

## Measurement baseline

Prefer profiling with:

- real device over Simulator for UI, launch, power, memory pressure, thermal behavior, and production-like responsiveness;
- Release or release-like builds over Debug builds;
- repeated runs over a single measurement;
- stable input data and deterministic scenarios;
- device and OS information included in the report;
- signposts around app-specific operations when system-level traces are too broad.

Use Simulator only when the question is about relative local investigation and the limitation is clearly stated.

## Tool selection

Choose the tool from the symptom, not from a favorite workflow.

| Symptom or question | Primary tool | Secondary signal |
|---|---|---|
| Cold launch, warm launch, first frame, first interaction | App Launch, Time Profiler, XCTest launch metrics | MetricKit, Organizer, signposts |
| Main-thread hang or freeze | Hangs, Time Profiler | Main Thread Checker, signposts |
| Animation hitch, scrolling hitch, dropped frames | Animation Hitches, Core Animation, Time Profiler | SwiftUI Instrument, signposts |
| SwiftUI repeated updates or broad invalidation | SwiftUI Instrument | Time Profiler, signposts |
| CPU spike or slow operation | Time Profiler | Counters, signposts, XCTest metrics |
| Memory growth, high allocations, churn | Allocations, VM Tracker | Memory Graph, Leaks, XCTest memory metric |
| Retain cycle or logical leak | Memory Graph Debugger | Allocations generation analysis |
| Disk reads/writes, persistence stalls, excessive writes | File Activity, System Trace | MetricKit disk write diagnostics |
| Slow networking or duplicated requests | Network instrument, URLSession metrics | Signposts, server timing |
| Battery drain, thermal pressure, wakeups | Energy Log, Power Profiler | MetricKit, Organizer |
| Production-only regression | MetricKit, Organizer | Local reproduction with Instruments |
| Regression protection | XCTest metrics | CI history, MetricKit release comparison |

When the signal is app-specific and not visible enough in system instruments, add `os_signpost` or `OSSignposter` around the operation.

## Cross-skill routing

Use this skill to select and validate the profiling path. Route deeper domain reasoning to narrower skills when needed:

- Use `ios-launch-performance` when the evidence points to pre-main work, dyld, static initializers, app initialization, root scene construction, first frame, first interaction, SDK startup, database warmup, or launch-critical dependency chains.
- Use `swiftui-performance` when the evidence points to broad state reads, unnecessary invalidation, unstable identity, expensive layout, row complexity, scrolling behavior, or repeated body work.
- Use `swift-concurrency-performance` when the evidence points to task explosions, actor hopping, MainActor bottlenecks, missing cancellation, AsyncSequence pressure, executor behavior, or async work causing UI latency.
- Use `ios-perceived-performance` when the evidence shows the app is technically doing work, but the user-visible problem is lack of feedback, poor loading states, late progressive rendering, or perceived latency.
- Use `swift-runtime-performance` when the evidence points to allocation churn, ARC traffic, existentials, generics, dynamic dispatch, copy-on-write, bridging, linking, or runtime-level costs.

Do not duplicate the deep guidance from those skills here.

## Profiling workflow

1. Identify the user-visible symptom.
2. Define the exact scenario that reproduces it.
3. Choose the primary profiling tool from the symptom.
4. Record the device, OS, build configuration, data set, and run count.
5. Capture the strongest signal: stack, frame hitch, allocation growth, retain path, network waterfall, disk write, wakeup, or production metric.
6. Separate what the data proves from what it only suggests.
7. Form one or two ranked hypotheses.
8. Recommend the smallest focused fix or the next inspection step.
9. Re-measure with the same scenario.
10. Suggest a regression guard when the issue is important enough.

## Trace interpretation rules

When reading traces, reports, or screenshots:

- Prefer the strongest signal over a broad list of possible causes.
- Distinguish local traces from production metrics.
- Distinguish CPU-bound, blocked, waiting, I/O-bound, memory-pressure, and network-bound symptoms.
- Check whether the cost is on the user-visible critical path.
- Treat averages carefully; p95 and p99 often matter more for hangs, launch, and production latency.
- Do not treat one clean run as proof that the issue is fixed.
- Do not infer a retain cycle from memory growth alone; inspect ownership paths.
- Do not infer CPU cost from wall-clock delay alone; the app may be blocked on I/O, locks, network, or the main actor.

## Fix selection rules

Recommend fixes only after connecting them to evidence.

Prefer:

- deferring or removing critical-path work;
- narrowing repeated work;
- reducing duplicate requests or duplicate computation;
- fixing ownership chains instead of adding weak references everywhere;
- batching disk writes or reducing write amplification;
- adding cancellation for invisible or obsolete work;
- using signposts and tests to keep the issue observable.

Avoid:

- broad rewrites without trace evidence;
- moving work to a background queue without checking whether the UI still awaits it;
- parallelizing work before understanding dependencies;
- optimizing code that is not on the critical path;
- claiming a tool proves something it does not measure.

## Gotchas

- Instruments explains local causes; MetricKit and Organizer identify production signals. Use both when possible.
- XCTest performance tests are better for regression protection than deep diagnosis.
- Debug builds can distort CPU, allocation, SwiftUI, and concurrency behavior.
- Simulator results can mislead for launch, scrolling, memory pressure, power, and thermal behavior.
- A hang can be a busy main thread, lock contention, synchronous I/O, actor waiting, or a dependency cycle. Do not assume CPU saturation.
- Memory Graph is usually better than Leaks for retain cycles where objects are still referenced.
- Allocation spikes are not automatically leaks. Look for growth across generations or retained object graphs.
- Network waterfalls can explain slow screens even when local CPU traces look clean.
- Power regressions often come from repeated small work: timers, polling, wakeups, background tasks, sensors, location, or offscreen animations.
- Do not call an optimization successful without a repeatable before/after measurement.

## References

Read these only when relevant:

- `references/tool-selection.md` — read when the task needs a deeper mapping from symptoms to Instruments templates, MetricKit, XCTest metrics, signposts, or production diagnostics.
- `references/time-profiler-and-hangs.md` — read when the task involves Time Profiler, Hangs, main-thread freezes, blocked threads, lock contention, synchronous I/O, CPU spikes, or stack interpretation.
- `references/animation-hitches-and-swiftui.md` — read when the task involves animation hitches, scrolling hitches, dropped frames, Core Animation, SwiftUI Instrument, repeated view updates, frame budget, or UI responsiveness traces.
- `references/memory-leaks-and-allocations.md` — read when the task involves Allocations, Leaks, Memory Graph Debugger, VM Tracker, memory growth, retain cycles, caches, decoded images, or allocation churn.
- `references/network-disk-power.md` — read when the task involves slow networking, duplicated requests, caching behavior, disk reads/writes, persistence stalls, excessive logging, background work, wakeups, battery drain, or thermal pressure.
- `references/xctest-metrickit-organizer.md` — read when the task involves XCTest performance tests, `XCTApplicationLaunchMetric`, CI regression guards, MetricKit payloads, Xcode Organizer, production regressions, device cohorts, p95, or p99.
- `references/signposts-and-scenarios.md` — read when the task needs a reproducible scenario, signpost instrumentation, signpost naming, custom trace regions, before/after comparison, or a profiling report template.

## Output expectations

For most profiling tasks, respond with:

```text
## Symptom

...

## Profiling path

Primary:
Secondary:
Why this tool:

## What to inspect

...

## Likely hypotheses

1. ...
2. ...

## Suggested fixes or next steps

...

## Verification

...
```

For code reviews, respond with:

```text
## Summary

...

## Findings

### 1. Finding title

Risk:
Why it matters:
Suggested change:
How to verify:

## Profiling checklist

...
```

For traces, reports, logs, MetricKit payloads, Organizer screenshots, or Instruments screenshots, respond with:

```text
## What the data shows

...

## Strongest signal

...

## Likely cause

...

## What is not proven yet

...

## Next step

...

## Verification

...
```

## Final review checklist

Before finalizing a performance answer, check:

- Did you identify the user-visible symptom?
- Did you choose the tool based on the symptom?
- Did you state what evidence is available and what is missing?
- Did you separate local traces from production metrics?
- Did you account for device, OS, build configuration, data set, and run count?
- Did you avoid claiming certainty without evidence?
- Did you recommend real-device Release profiling when relevant?
- Did you suggest signposts for app-specific operations when useful?
- Did you propose one focused fix or next inspection step at a time?
- Did you include a re-measurement step?
- Did you suggest XCTest, MetricKit, or Organizer for regression protection when appropriate?
- Did you avoid broad rewrites unless evidence supports them?
