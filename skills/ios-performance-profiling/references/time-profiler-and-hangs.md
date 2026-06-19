# Time Profiler and Hangs

Use this reference when the task involves Time Profiler, Hangs, main-thread freezes, blocked threads, lock contention, synchronous I/O, CPU spikes, or stack interpretation.

This file helps the agent distinguish busy CPU from blocked execution. Do not treat every freeze as a CPU problem.

## Scope Boundary

This reference covers:

* Time Profiler interpretation;
* Hangs instrument interpretation;
* main-thread freezes;
* blocked threads;
* CPU spikes;
* synchronous I/O;
* lock contention;
* stack interpretation;
* signpost handoff for broad traces;
* before/after validation for CPU, blocking, and hang fixes.

This reference does not cover:

* animation hitch diagnosis unless the issue is CPU or blocking inside the hitch window;
* memory leaks except allocation churn visible in CPU traces;
* network, disk, or power profiling except when they appear as CPU or blocking stacks;
* SwiftUI invalidation, identity, or state-scope fixes beyond stack-level evidence;
* launch architecture except launch-time CPU, blocking, or hangs;
* production monitoring strategy except as validation evidence for hangs or responsiveness regressions.

## Contents

* [Core Model](#core-model)
* [Tool Choice](#tool-choice)
* [Before Profiling](#before-profiling)
* [Time Profiler Workflow](#time-profiler-workflow)
* [Hangs Workflow](#hangs-workflow)
* [Stack Interpretation](#stack-interpretation)
* [Main-Thread Freezes](#main-thread-freezes)
* [Blocked Threads](#blocked-threads)
* [CPU Spikes](#cpu-spikes)
* [Synchronous I/O](#synchronous-io)
* [Lock Contention](#lock-contention)
* [Signposts](#signposts)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [Fix Selection](#fix-selection)
* [Common Mistakes](#common-mistakes)
* [Validation](#validation)
* [Output Notes](#output-notes)

## Core Model

Time Profiler answers:

**Where did sampled CPU time go while the process was running?**

The Hangs instrument answers:

**Where did the app stop responding long enough to create a user-visible freeze or hang?**

These are related, but not equivalent.

A freeze can happen because the main thread is busy burning CPU. It can also happen because the main thread is blocked on a lock, synchronous I/O, IPC, a semaphore, `DispatchQueue.sync`, a database transaction, a callback, a result from another task, or an actor dependency.

Do not assume every hang is a CPU problem.

Time Profiler is sampling-based. It shows where execution was observed during samples, not an exact count of every call. Short functions can matter if repeated often, and very short spikes can be missed if the sampling window is poor.

Treat Hangs as evidence of user-visible unresponsiveness, not as a complete explanation. A hang interval still needs stack interpretation to distinguish busy CPU, wait, lock, I/O, actor dependency, queue inversion, or run loop starvation.

## Tool Choice

Use **Time Profiler** when the task involves:

* CPU spikes;
* slow synchronous operations;
* expensive parsing, mapping, formatting, diffing, sorting, layout, or rendering preparation;
* repeated work during scrolling, interaction, screen setup, or launch;
* understanding which functions dominate execution time;
* stack-level explanation of a selected slow interval.

Use **Hangs** when the task involves:

* visible freezes;
* long stalls during interaction;
* main-thread unresponsiveness;
* intermittent hangs;
* blocking stacks;
* deciding whether the app is busy, waiting, locked, or blocked on I/O.

If the user reports a freeze and no trace is available, recommend recording both Hangs and Time Profiler for the same scenario.

Use `animation-hitches-and-swiftui.md` when the primary symptom is dropped frames or animation hitches and CPU/blocking stacks are only part of the explanation.

Use `network-disk-and-power-profiling.md` when the dominant issue is request waterfalls, disk write pressure, wakeups, sensors, or battery drain rather than a selected CPU or hang stack.

## Before Profiling

Prefer:

* real device;
* Release or release-like build;
* stable data set;
* repeatable scenario;
* clean app state when relevant;
* device model and OS version recorded;
* one focused interaction per recording;
* signposts around the app-specific operation if the trace is too broad.

Be careful with:

* one Debug run;
* Simulator-only evidence for responsiveness;
* traces that include many unrelated interactions;
* screenshots that do not show the relevant stack or time range;
* whole-recording totals when the symptom happened in a narrow interval.

Useful Time Profiler views often include:

* separate by thread;
* invert call tree;
* hide system libraries when looking for app-heavy stacks;
* show system libraries again when framework work, blocking APIs, runtime behavior, or I/O matter;
* select the exact time range around the symptom before interpreting totals.

Do not make a production claim from one local trace. Use the trace to form or validate a focused hypothesis.

## Time Profiler Workflow

1. Define the scenario in user terms: launch, tap, scroll, search, screen open, sync, import, export, or background refresh.
2. Record the smallest trace that includes the slow interval.
3. Select the exact time range where the symptom happens.
4. Separate main-thread work from background work.
5. Find the hottest stack on the user-visible critical path.
6. Expand stacks until app frames appear.
7. Separate self time from total time.
8. Classify the cost: computation, decoding, formatting, layout, rendering preparation, bridging, allocation-heavy code, synchronization, or I/O.
9. Form one focused hypothesis.
10. Propose the smallest fix that removes, defers, caches, batches, or moves the measured work.
11. Re-measure the same scenario and selected interval.

Prefer the selected slow interval over whole-recording totals. Whole traces often hide the actual user-visible problem.

Do not optimize a globally hot function if it is not active during the symptom interval.

## Hangs Workflow

1. Identify the user-visible freeze: what action triggered it and what stopped responding.
2. Record with Hangs. Add Time Profiler when CPU cost may also matter.
3. Select the hang interval.
4. Inspect the main-thread stack first.
5. Decide whether the main thread is busy or blocked.
6. If blocked, identify the wait: lock, semaphore, dispatch sync, I/O, database, IPC, task dependency, actor dependency, callback, run loop wait, or another thread.
7. If another thread is involved, inspect the owning or contending thread.
8. Check whether the dependency is required for the user-visible operation.
9. Propose a fix that removes the wait from the main thread or shortens the critical section.
10. Re-measure with the same interaction.

A hang report can be more useful than a broad CPU profile when the app is waiting rather than computing.

If the main thread is waiting, the next question is not “what is hot?” but “what owns the thing the main thread is waiting for?”

## Stack Interpretation

Use stack frames as evidence, not as a list of names to mention.

Ask:

* Is this stack inside the selected slow interval?
* Is it on the main thread?
* Is it app code, framework code called by app code, or unrelated system work?
* Is it CPU work or a wait state?
* Is the function slow once, or cheap but repeated many times?
* Is this frame the cause, or only a caller of the expensive operation?
* Does the frame have high self time or high total time?
* Does the cost grow with data size, number of views, number of requests, or number of iterations?

Read stacks from the symptom toward the app-level cause. Top frames often show what is executing or waiting. Lower app frames often reveal the feature path or call site.

Separate self time from total time. A function with high total time may mostly be a caller. A function with high self time is more likely doing direct work.

Repeated small costs can dominate even when no single call looks expensive. Inspect repeated stacks inside the selected interval.

Do not blame the first app frame automatically. It may only be the caller of expensive framework work.

## Main-Thread Freezes

Common causes:

* synchronous disk reads or writes;
* database queries, migrations, or transactions;
* JSON parsing and model mapping;
* image decoding or resizing;
* date, number, text, or attributed string formatting in loops;
* large diff generation;
* layout or text measurement for many items;
* blocking waits for async work;
* lock contention;
* synchronous queue hops;
* actor or task dependency cycles;
* callbacks that re-enter the main thread at the wrong time.

Fix direction:

* remove work from the main-thread critical path;
* cache stable results;
* batch repeated work;
* defer non-critical work until after the interaction;
* avoid awaiting background work before first feedback;
* make long-running async work cancellable;
* avoid synchronous bridges from async APIs to main-thread call sites.

Do not assume `async` means non-blocking for the UI. Heavy work can still run on the `MainActor`, and awaiting a dependency can still delay first feedback or interaction.

## Blocked Threads

A blocked main thread may show low CPU usage. That does not make the hang harmless.

Look for:

* mutexes and unfair locks;
* semaphores and condition waits;
* `DispatchQueue.sync`;
* database locks;
* file coordination;
* synchronous APIs hidden behind wrappers;
* actor or task dependencies that the main actor awaits;
* callbacks, delegates, or completion handlers that are required to unblock progress;
* queue inversion.

Swift concurrency can create hangs when the `MainActor` awaits a task that is waiting on another actor, lock, database, callback, or queue that eventually needs the main actor again. Look for circular waits, unstructured tasks, `MainActor.run`, semaphore bridges, and async APIs wrapped in synchronous interfaces.

Watch for main queue sync calls and queue inversion: a background queue waits synchronously for the main queue while the main queue waits for that background queue, lock, actor, or completion.

Fix direction:

* avoid main-thread waits for background results;
* avoid semaphore bridges from async code;
* avoid background queues that synchronously call main while main waits for them;
* make dependencies explicit;
* shorten critical sections;
* break circular waits between queues, locks, actors, callbacks, and the main thread.

## CPU Spikes

A CPU spike matters when it affects the user-visible path, battery, thermal state, or responsiveness.

Common causes:

* repeated parsing, mapping, sorting, or filtering;
* expensive equality or hashing;
* string processing;
* image processing;
* compression or encryption;
* diffing large data sets;
* repeated formatting;
* excessive logging;
* polling or retry loops;
* repeated allocation-heavy transformations.

Fix direction:

* reduce frequency;
* cache stable work;
* improve algorithmic complexity;
* batch operations;
* cancel obsolete work;
* keep heavy work out of repeated UI update paths;
* add regression tests for important hot paths.

Check whether cost grows linearly, superlinearly, or with repeated invalidations as data size increases. A fix that works for 20 items may not work for 2,000.

## Synchronous I/O

Synchronous I/O can appear as hangs, launch delays, or slow interactions.

Look for:

* file reads during screen construction;
* cache reads before first render;
* full-file rewrites;
* database work on the main thread;
* logging inside hot paths;
* image loading from disk followed by decode on main;
* repeated preferences, keychain, or configuration reads.

Keychain, UserDefaults, file reads, database fetches, and configuration loads can be cheap in one case and expensive in another. Treat them as suspicious only when they appear inside the selected slow interval or hang stack.

Fix direction:

* defer non-critical reads;
* batch writes;
* use transactions;
* avoid full-file rewrites for small changes;
* keep database work off the main thread;
* cache values that are safe to cache;
* add signposts around file and database operations.

Do not simply “move I/O to background” if the UI immediately waits for the result. That changes the thread, not necessarily the latency.

## Lock Contention

Lock contention matters when the main thread waits for a lock held by another thread, or when many workers serialize on a shared resource.

Look for:

* main thread blocked on a lock;
* background thread holding a lock while doing I/O, logging, parsing, or callbacks;
* nested locks;
* lock acquisition inside frequently called paths;
* shared caches protected by coarse locks;
* synchronization around code that calls back into UI or user-provided closures;
* ordinary locks held across work that may re-enter app code or cross actor boundaries.

Fix direction:

* reduce lock scope;
* avoid I/O while holding locks;
* avoid callbacks while holding locks;
* avoid holding ordinary locks across calls that may re-enter app code, call delegates, post notifications, or cross actor boundaries;
* never suspend with an ordinary lock held;
* split shared state;
* use immutable snapshots;
* measure after changing synchronization because fixes can move contention elsewhere.

Do not add locks as a performance fix without validating contention afterward.

## Signposts

Use signposts when system stacks are too broad to identify the app-specific operation.

Good signpost regions:

* screen setup;
* search query processing;
* model mapping;
* diff generation;
* database fetch;
* cache read/write;
* image decode or resize;
* import/export;
* sync step;
* interaction handler.

Use operation names, not implementation placeholders.

Prefer:

```text id="vupbfc"
CatalogScreen.loadInitialData
Search.applyQuery
Feed.diffSnapshot
ImagePipeline.decodeThumbnail
```

Avoid:

```text id="ohhgpv"
doWork
managerCall
step1
performanceTest
```

Use `signposts-and-scenarios.md` when the scenario, naming, metadata, or before/after report needs to be designed. Use this section only to decide where signposts would clarify Time Profiler or Hangs.

## What the Agent Can Inspect

When repository access is available, inspect concrete CPU, blocking, and hang risks instead of giving generic advice.

Search for synchronous waits and queue hops:

```sh id="ggtoir"
rg "DispatchQueue\.main\.sync|DispatchQueue\..*\.sync|semaphore|wait\(|NSCondition|pthread_mutex|os_unfair_lock|NSLock" .
```

Search for main-actor and async bridge risks:

```sh id="3c0zrw"
rg "@MainActor|MainActor\.run|Task\s*\{|Task\.detached|withCheckedContinuation|withUnsafeContinuation|runBlocking|blocking|await" .
```

Search for synchronous I/O:

```sh id="km8mej"
rg "Data\(|contentsOf:|FileManager|UserDefaults|Keychain|SecItem|write\(|read|sqlite|CoreData|SwiftData|save\(|fetch" .
```

Search for CPU-heavy operations:

```sh id="e19qny"
rg "sorted\(|sort\(|map\(|compactMap\(|filter\(|reduce\(|JSONDecoder|DateFormatter|NumberFormatter|NSAttributedString|UIImage|CGImage" .
```

Search for locks and callbacks:

```sh id="p8oz2b"
rg "lock\(|unlock\(|withLock|synchronized|delegate|callback|completion|NotificationCenter|post\(" .
```

Search for signposts and timing boundaries:

```sh id="w4k5kq"
rg "OSSignposter|os_signpost|beginInterval|endInterval|CFAbsoluteTimeGetCurrent|CACurrentMediaTime|Date\(" .
```

Use matches as leads, not proof. Confirm the matched code appears in the selected slow interval or hang stack.

The agent can:

* classify a trace as busy CPU, blocking, synchronous I/O, lock contention, or inconclusive;
* identify likely app call sites from stack traces;
* recommend the next tool or narrower time range;
* suggest signposts around unclear app operations;
* propose focused fixes tied to the measured cause;
* define before/after validation for the same scenario.

The agent cannot reliably:

* prove root cause from a whole-trace total alone;
* prove a hang is CPU-bound without checking wait/blocking state;
* blame framework frames without an app call site;
* prove production impact from a single local trace;
* make a safe synchronization change without understanding ownership and dependencies.

## Fix Selection

Choose fixes that match the measured cause.

If the trace shows busy CPU:

* reduce work;
* cache stable work;
* improve algorithmic complexity;
* avoid repeated computation;
* move non-critical work away from the main thread;
* reduce allocation-heavy transformations when they appear in the selected interval.

If the trace shows blocking:

* remove the wait from the main thread;
* shorten the critical section;
* avoid synchronous bridges;
* avoid circular queue or actor waits;
* make dependencies explicit;
* add timeout or cancellation only when product correctness allows it.

If the trace shows synchronous I/O:

* defer non-critical I/O;
* batch writes;
* reduce write amplification;
* avoid disk access during interaction;
* use signposts to confirm where I/O happens.

If the trace is inconclusive:

* narrow the scenario;
* add signposts;
* collect another trace;
* compare a good run and a bad run;
* avoid broad rewrites.

## Common Mistakes

* Treating wall-clock delay as CPU cost without checking blocked states.
* Treating Time Profiler samples as exact call counts.
* Blaming framework frames without finding the app call site.
* Looking at the whole trace instead of the selected slow interval.
* Treating high total time in a caller as proof that the caller itself is expensive.
* Treating Debug-build hot spots as production facts.
* Optimizing background work that is not on the user-visible critical path.
* Moving work to a background queue while the main thread still waits for it.
* Adding locks to fix races without measuring contention afterward.
* Holding locks across I/O, callbacks, notifications, or actor-crossing work.
* Ignoring repeated small costs because no single function looks huge.
* Assuming `async` work cannot block or delay the UI.
* Calling the issue fixed after one clean run.

## Validation

Validate with the same scenario before and after the change.

Include:

* device model;
* OS version;
* build configuration;
* scenario steps;
* data set size;
* number of runs;
* primary metric;
* strongest trace evidence;
* selected slow or hang interval;
* whether the main thread is still busy or blocked;
* whether p95/p99 or worst-case behavior improved when relevant.

Validate the same selected interval, not only total recording time. Confirm that the main thread is no longer busy or blocked at the original symptom point.

For important regressions, add a guard:

* XCTest performance test for deterministic CPU-heavy code;
* signpost-based measurement for app-specific operations;
* MetricKit or Organizer monitoring for production hangs and responsiveness regressions.

Also validate correctness:

* cancellation behavior;
* loading or failure states;
* data consistency;
* ordering guarantees;
* thread safety;
* UI responsiveness;
* battery or thermal side effects when work is moved or retried.

Do not claim success without a before/after comparison or a clear plan to obtain one.

## Output Notes

When using this reference in an answer, include:

1. The symptom.
2. Whether the evidence suggests busy CPU, blocking, I/O, lock contention, or inconclusive data.
3. The strongest stack or trace signal.
4. The most likely app-level cause.
5. One focused fix or next inspection step.
6. What is not proven yet.
7. How to re-measure.
8. The next trace or screenshot needed if evidence is incomplete.

Use this output shape when helpful:

```markdown id="4pgjlu"
## Symptom

## Evidence classification

Busy CPU / blocking / synchronous I/O / lock contention / inconclusive.

## Strongest stack or trace signal

## Likely app-level cause

## What is not proven yet

## Fix or next inspection step

## Re-measurement plan
```
