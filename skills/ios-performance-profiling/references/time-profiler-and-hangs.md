# Time Profiler and Hangs

Use this reference when the task involves Time Profiler, Hangs, main-thread freezes, blocked threads, lock contention, synchronous I/O, CPU spikes, or stack interpretation.

## Contents

- [Core model](#core-model)
- [Tool choice](#tool-choice)
- [Before profiling](#before-profiling)
- [Time Profiler workflow](#time-profiler-workflow)
- [Hangs workflow](#hangs-workflow)
- [Stack interpretation](#stack-interpretation)
- [Main-thread freezes](#main-thread-freezes)
- [Blocked threads](#blocked-threads)
- [CPU spikes](#cpu-spikes)
- [Synchronous I/O](#synchronous-io)
- [Lock contention](#lock-contention)
- [Signposts](#signposts)
- [Fix selection](#fix-selection)
- [Common mistakes](#common-mistakes)
- [Validation](#validation)
- [Output notes](#output-notes)

## Core model

Time Profiler answers: **where did sampled CPU time go while the process was running?**

The Hangs instrument answers: **where did the app stop responding long enough to create a user-visible freeze or hang?**

These are related, but not equivalent. A freeze can happen because the main thread is busy burning CPU. It can also happen because the main thread is blocked on a lock, synchronous I/O, IPC, a semaphore, `DispatchQueue.sync`, a database transaction, or a result from another task or actor.

Do not assume every hang is a CPU problem.

## Tool choice

Use **Time Profiler** when the task involves:

- CPU spikes;
- slow synchronous operations;
- expensive parsing, mapping, formatting, diffing, sorting, layout, or rendering preparation;
- repeated work during scrolling, interaction, screen setup, or launch;
- understanding which functions dominate execution time.

Use **Hangs** when the task involves:

- visible freezes;
- long stalls during interaction;
- main-thread unresponsiveness;
- intermittent hangs;
- blocking stacks;
- deciding whether the app is busy, waiting, locked, or blocked on I/O.

If the user reports a freeze and no trace is available, recommend recording both Hangs and Time Profiler for the same scenario.

## Before profiling

Prefer:

- real device;
- Release or release-like build;
- stable data set;
- repeatable scenario;
- clean app state when relevant;
- device model and OS version recorded;
- one focused interaction per recording;
- signposts around the app-specific operation if the trace is too broad.

Be careful with:

- one Debug run;
- Simulator-only evidence for responsiveness;
- traces that include many unrelated interactions;
- screenshots that do not show the relevant stack or time range.

## Time Profiler workflow

1. Define the scenario in user terms: launch, tap, scroll, search, screen open, sync, import, export, or background refresh.
2. Record the smallest trace that includes the slow interval.
3. Select the exact time range where the symptom happens.
4. Separate main-thread work from background work.
5. Find the hottest stack on the user-visible critical path.
6. Expand stacks until app frames appear.
7. Classify the cost: computation, decoding, formatting, layout, rendering preparation, bridging, allocation-heavy code, synchronization, or I/O.
8. Form one focused hypothesis.
9. Propose the smallest fix that removes, defers, caches, batches, or moves the measured work.
10. Re-measure the same scenario.

Prefer the selected slow interval over whole-recording totals. Whole traces often hide the actual user-visible problem.

## Hangs workflow

1. Identify the user-visible freeze: what action triggered it and what stopped responding.
2. Record with Hangs. Add Time Profiler when CPU cost may also matter.
3. Select the hang interval.
4. Inspect the main-thread stack first.
5. Decide whether the main thread is busy or blocked.
6. If blocked, identify the wait: lock, semaphore, dispatch sync, I/O, database, IPC, task dependency, actor dependency, or another thread.
7. If another thread is involved, inspect the owning or contending thread.
8. Check whether the dependency is required for the user-visible operation.
9. Propose a fix that removes the wait from the main thread or shortens the critical section.
10. Re-measure with the same interaction.

A hang report can be more useful than a broad CPU profile when the app is waiting rather than computing.

## Stack interpretation

Use stack frames as evidence, not as a list of names to mention.

Ask:

- Is this stack inside the selected slow interval?
- Is it on the main thread?
- Is it app code, framework code called by app code, or unrelated system work?
- Is it CPU work or a wait state?
- Is the function slow once, or cheap but repeated many times?
- Is this frame the cause, or only a caller of the expensive operation?

Read stacks from the symptom toward the app-level cause. Top frames often show what is executing or waiting. Lower app frames often reveal the feature path or call site.

Do not blame the first app frame automatically. It may only be the caller of expensive framework work.

## Main-thread freezes

Common causes:

- synchronous disk reads or writes;
- database queries, migrations, or transactions;
- JSON parsing and model mapping;
- image decoding or resizing;
- date, number, text, or attributed string formatting in loops;
- large diff generation;
- layout or text measurement for many items;
- blocking waits for async work;
- lock contention;
- synchronous queue hops.

Fix direction:

- remove work from the main-thread critical path;
- cache stable results;
- batch repeated work;
- defer non-critical work until after the interaction;
- avoid awaiting background work before first feedback;
- make long-running async work cancellable.

## Blocked threads

A blocked main thread may show low CPU usage. That does not make the hang harmless.

Look for:

- mutexes and unfair locks;
- semaphores and condition waits;
- `DispatchQueue.sync`;
- database locks;
- file coordination;
- synchronous APIs hidden behind wrappers;
- actor or task dependencies that the main actor awaits.

Fix direction:

- avoid main-thread waits for background results;
- avoid semaphore bridges from async code;
- avoid background queues that synchronously call main while main waits for them;
- make dependencies explicit;
- shorten critical sections.

## CPU spikes

A CPU spike matters when it affects the user-visible path, battery, thermal state, or responsiveness.

Common causes:

- repeated parsing, mapping, sorting, or filtering;
- expensive equality or hashing;
- string processing;
- image processing;
- compression or encryption;
- diffing large data sets;
- repeated formatting;
- excessive logging;
- polling or retry loops.

Fix direction:

- reduce frequency;
- cache stable work;
- improve algorithmic complexity;
- batch operations;
- cancel obsolete work;
- keep heavy work out of repeated UI update paths;
- add regression tests for important hot paths.

## Synchronous I/O

Synchronous I/O can appear as hangs, launch delays, or slow interactions.

Look for:

- file reads during screen construction;
- cache reads before first render;
- full-file rewrites;
- database work on the main thread;
- logging inside hot paths;
- image loading from disk followed by decode on main;
- repeated preferences, keychain, or configuration reads.

Fix direction:

- defer non-critical reads;
- batch writes;
- use transactions;
- avoid full-file rewrites for small changes;
- keep database work off the main thread;
- cache values that are safe to cache;
- add signposts around file and database operations.

Do not simply “move I/O to background” if the UI immediately waits for the result. That changes the thread, not necessarily the latency.

## Lock contention

Lock contention matters when the main thread waits for a lock held by another thread, or when many workers serialize on a shared resource.

Look for:

- main thread blocked on a lock;
- background thread holding a lock while doing I/O, logging, parsing, or callbacks;
- nested locks;
- lock acquisition inside frequently called paths;
- shared caches protected by coarse locks;
- synchronization around code that calls back into UI or user-provided closures.

Fix direction:

- reduce lock scope;
- avoid I/O while holding locks;
- avoid callbacks while holding locks;
- split shared state;
- use immutable snapshots;
- measure after changing synchronization because fixes can move contention elsewhere.

## Signposts

Use signposts when system stacks are too broad to identify the app-specific operation.

Good signpost regions:

- screen setup;
- search query processing;
- model mapping;
- diff generation;
- database fetch;
- cache read/write;
- image decode or resize;
- import/export;
- sync step;
- interaction handler.

Use operation names, not implementation placeholders.

Prefer:

```text
CatalogScreen.loadInitialData
Search.applyQuery
Feed.diffSnapshot
ImagePipeline.decodeThumbnail
```

Avoid:

```text
doWork
managerCall
step1
performanceTest
```

## Fix selection

Choose fixes that match the measured cause.

If the trace shows busy CPU:

- reduce work;
- cache stable work;
- improve algorithmic complexity;
- avoid repeated computation;
- move non-critical work away from the main thread.

If the trace shows blocking:

- remove the wait from the main thread;
- shorten the critical section;
- avoid synchronous bridges;
- avoid circular queue or actor waits.

If the trace shows synchronous I/O:

- defer non-critical I/O;
- batch writes;
- reduce write amplification;
- avoid disk access during interaction;
- use signposts to confirm where I/O happens.

If the trace is inconclusive:

- narrow the scenario;
- add signposts;
- collect another trace;
- compare a good run and a bad run;
- avoid broad rewrites.

## Common mistakes

- Treating wall-clock delay as CPU cost without checking blocked states.
- Blaming framework frames without finding the app call site.
- Looking at the whole trace instead of the selected slow interval.
- Treating Debug-build hot spots as production facts.
- Optimizing background work that is not on the user-visible critical path.
- Moving work to a background queue while the main thread still waits for it.
- Adding locks to fix races without measuring contention afterward.
- Ignoring repeated small costs because no single function looks huge.
- Calling the issue fixed after one clean run.

## Validation

Validate with the same scenario before and after the change.

Include:

- device model;
- OS version;
- build configuration;
- scenario steps;
- data set size;
- number of runs;
- primary metric;
- strongest trace evidence;
- whether the main thread is still busy or blocked;
- whether p95/p99 or worst-case behavior improved when relevant.

For important regressions, add a guard:

- XCTest performance test for deterministic CPU-heavy code;
- signpost-based measurement for app-specific operations;
- MetricKit or Organizer monitoring for production hangs and responsiveness regressions.

Do not claim success without a before/after comparison or a clear plan to obtain one.

## Output notes

When using this reference in an answer, include:

1. The symptom.
2. Whether the evidence suggests busy CPU, blocking, I/O, or inconclusive data.
3. The strongest stack or trace signal.
4. The most likely app-level cause.
5. One focused fix or next inspection step.
6. How to re-measure.

If evidence is missing, say exactly what trace or screenshot is needed next.
