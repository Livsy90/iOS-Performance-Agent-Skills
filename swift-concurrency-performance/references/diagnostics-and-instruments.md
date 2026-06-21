# Diagnostics and Instruments for Swift Concurrency Performance

Use this reference when the user provides traces, logs, measurements, production signals, or asks how to validate concurrency-related performance changes.

This reference helps the agent connect Swift Concurrency recommendations to evidence instead of guessing. Use it when the answer needs diagnosis, validation, trace interpretation, production metric interpretation, or a before/after measurement plan.

This is not a general Instruments guide. It is a concurrency-performance diagnostic workflow.

## Contents

* [Core model](#core-model)
* [Diagnostic workflow](#diagnostic-workflow)
* [Decision rules](#decision-rules)
* [Important distinctions](#important-distinctions)
* [Symptom mapping](#symptom-mapping)
* [Instruments selection](#instruments-selection)
* [Trace reading checklist](#trace-reading-checklist)
* [Signposts and logs](#signposts-and-logs)
* [Production signals](#production-signals)
* [Validation patterns](#validation-patterns)
* [Common mistakes](#common-mistakes)
* [Report template](#report-template)

## Core model

Swift Concurrency performance problems usually appear as one of these outcomes:

* UI work waits too long before it can run.
* Work runs on the wrong isolation domain.
* Too many tasks are created for the amount of useful work.
* A cooperative executor thread is blocked by synchronous work.
* An actor becomes a serialization bottleneck.
* Work continues after the user no longer needs it.
* Async bridges never finish, finish twice, or keep producers alive.
* Parallelism increases memory pressure instead of reducing latency.
* Cancellation suppresses UI updates but does not stop the underlying work.
* Production tail latency gets worse even when average latency looks acceptable.

Diagnostics should answer three questions:

1. What user-visible symptom happened?
2. Which async boundary, isolation boundary, or lifetime boundary is involved?
3. What evidence proves the proposed fix affects that boundary?

Do not treat “uses async/await” as evidence. Async code can still block, serialize, leak, duplicate work, ignore cancellation, retain producers, or delay UI state.

Do not treat a preferred optimization as proven until a trace, test, log, or production signal changes in the expected direction.

## Diagnostic workflow

Use this workflow when reviewing a trace, logs, metrics, or a proposed optimization.

1. Define the user-visible symptom: UI stall, delayed screen load, duplicate work, memory spike, stuck loading state, high tail latency, or work continuing after navigation.
2. Identify the async boundary: `Task`, `Task.detached`, task group, actor method, `MainActor`, continuation, stream, `for await` loop, callback bridge, or synchronous legacy API.
3. Identify the owner of the work: view, view model, request, service, actor, stream consumer, app session, or detached process.
4. Check whether the work should still be alive after dismissal, cancellation, timeout, parent cancellation, stream termination, or owner deallocation.
5. Classify the slow interval: CPU-bound, blocked, suspended, waiting, serialized, retained, or over-parallelized.
6. Connect the symptom to a measurable signal.
7. Recommend the smallest fix that changes the measured signal.
8. Define a before/after validation.

Useful evidence includes:

* main-thread stalls;
* long `MainActor` intervals;
* actor queue buildup;
* high task counts;
* blocked threads;
* repeated request ids;
* retained producers;
* allocation bursts;
* missing cancellation logs;
* continuations without completion;
* duplicate callback or duplicate request events;
* p95/p99 latency regression;
* memory that does not return to baseline.

If the trace does not support a conclusion, say what extra capture, signpost, log field, test, or production metric is needed.

## Decision rules

* Start from the user-visible interval, not from an API preference.
* Do not call a change an optimization until the measured signal changes.
* Separate CPU-bound, blocked, suspended, waiting, serialized, retained, and over-parallelized work.
* Use signposts when traces lack semantic boundaries.
* Prefer controlled tests for cancellation, continuation, stream, duplicate-work, and owner-lifetime bugs.
* Prefer Release or release-like builds for performance comparisons.
* Do not move work off `MainActor` unless UI isolation and API requirements are understood.
* Do not replace sequential awaits with parallelism without checking memory, cancellation, and downstream limits.
* Do not claim cooperative executor starvation without supporting evidence.
* Do not diagnose actor contention only because an actor exists; show repeated hops, long isolated work, queueing, or serialized latency.
* Do not diagnose task explosion only because task groups exist; connect it to input size, task count, memory, scheduling overhead, or downstream contention.
* Do not use mean latency alone when tail behavior matters.
* Report uncertainty when the trace does not prove the boundary.

## Important distinctions

Do not treat waiting, suspension, and blocking as the same thing.

### Suspended task

A suspended task is not currently occupying a thread. This can be healthy. A task waiting for network, a timer, or another async operation may simply be suspended.

A suspended task becomes suspicious when:

* it never resumes;
* it outlives its owner;
* it waits for a continuation that is never resumed;
* it depends on a producer that was not cleaned up;
* it blocks user-visible progress for too long.

### Blocked thread

A blocked thread is occupied while not making useful progress. This is usually more serious in async paths because cooperative executor threads are shared runtime resources.

Look for:

* semaphore waits;
* synchronous I/O;
* `Thread.sleep`;
* blocking SDK calls;
* long lock contention;
* `queue.sync` waits;
* CPU work hidden behind an async wrapper.

### Waiting on actor or serialized resource

A task waiting for an actor, lock, database, SDK queue, or server limit may not be blocked at the thread level, but user-visible latency can still increase.

Look for:

* repeated actor calls;
* hot actor APIs;
* long actor-isolated sections;
* one database queue handling many requests;
* server-side rate limits;
* many callers waiting on the same resource.

### CPU-bound work

CPU-bound work consumes processor time. It may be correct but still too expensive for a user-visible path.

Look for:

* decoding;
* sorting;
* formatting;
* compression;
* crypto;
* image processing;
* text layout;
* JSON parsing;
* model mapping;
* large diff computation.

### Retained work

Retained work continues after the result is no longer needed. It may not show as one obvious slow function, but it can drain memory, CPU, network, or battery.

Look for:

* tasks alive after navigation;
* streams whose producers continue after termination;
* callbacks retained by SDKs;
* detached tasks without owners;
* continuations that never resume;
* requests that continue after cancellation.

## Symptom mapping

### UI stall or slow interaction

Likely causes:

* heavy work running on `MainActor`;
* synchronous blocking call on the main path;
* too many actor hops before UI state updates;
* UI awaiting a long async dependency before showing feedback;
* cooperative executor delays caused by blocking work in async paths;
* expensive result aggregation returning to `MainActor`.

Look for:

* long main-thread samples;
* CPU-heavy `@MainActor` methods;
* action-to-feedback intervals with several awaits before visible feedback;
* repeated hops between UI code and actors;
* blocking calls in a user-visible path;
* a slow awaited dependency that gates first feedback.

Validate by comparing tap-to-feedback or action-to-render signpost intervals before and after. Move only CPU-heavy or blocking work out of UI isolation, and keep UI state mutation on `MainActor`.

Do not claim cooperative executor starvation unless the trace shows blocked cooperative threads, many runnable tasks delayed by blocking work, or strong indirect evidence such as widespread delayed resumptions with blocking calls in async paths.

### Actor queue buildup or low throughput

Likely causes:

* hot actor receiving too many small calls;
* long actor-isolated sections;
* blocking work inside actor methods;
* chatty API that forces repeated isolation hops;
* actor method suspending and resuming with stale assumptions;
* task group repeatedly entering the same actor.

Look for:

* many calls to the same actor on a hot path;
* serialized execution where independent work could happen outside the actor;
* repeated await chains;
* logs showing growing actor wait time;
* signposts showing many small actor operations;
* one actor method dominating operation latency.

Validate by batching actor operations, moving non-state work outside actor isolation, reducing per-item actor calls, and comparing request latency or actor wait time.

Good batching often means entering the actor once to copy or compute a small state snapshot, then doing CPU-heavy transformation outside actor isolation. Do not batch so much work that the actor becomes blocked for longer.

### High task count or task explosion

Likely causes:

* one child task per item in a large collection;
* repeated `.task` creation from view lifecycle churn;
* unstructured tasks created without ownership;
* task groups with unbounded fan-out;
* retry loops creating new tasks without cancelling old work;
* detached tasks used as a generic background mechanism.

Look for:

* many short-lived tasks with similar call stacks;
* tasks surviving longer than their owner;
* memory growth that tracks task count;
* input size that directly controls child task count;
* repeated task creation after navigation, refresh, or search changes.

Validate by bounding concurrency, reusing a parent task where appropriate, storing and cancelling unstructured tasks, and comparing peak task count, peak memory, cancellation behavior, and total latency.

### Memory spike during async work

Likely causes:

* unbounded task group;
* collecting all results before yielding;
* `AsyncStream` buffering without a limit;
* tasks retaining large values;
* producers retained after stream termination;
* detached task retaining `self` or services longer than expected;
* continuation bridges retaining owners or operation handles after completion.

Look for:

* allocation bursts aligned with fan-out;
* growing buffers;
* retained producers;
* retained callbacks;
* retained bridge objects;
* task closures capturing large objects;
* memory that does not return to baseline after cancellation or completion.

Validate by bounding parallelism, streaming partial results, configuring buffering policy, cleaning up producers in `onTermination`, clearing continuation bridges, and comparing peak memory and post-cancellation baseline.

### Duplicate network work or cache stampede

Likely causes:

* actor reentrancy after `await`;
* no in-flight request tracking;
* cache check happens before suspension but is not rechecked after resumption;
* multiple tasks independently request the same resource;
* replacement operations not cancelling or coalescing previous work.

Look for:

* repeated request ids or URLs;
* actor methods that check cache before awaiting network;
* logs showing multiple callers entering the same miss path;
* concurrent calls for the same key;
* repeated work after cancellation or retry.

Validate by storing in-flight tasks, re-checking state after `await`, coalescing duplicate work, and comparing duplicate request count.

### Stuck loading state or never-ending task

Likely causes:

* continuation not resumed on every path;
* callback bridge misses failure, timeout, cancellation, or early-return path;
* `AsyncStream` never finishes;
* `for await` loop waits forever;
* cancellation swallowed by broad `catch`;
* producer outlives consumer and keeps the operation alive.

Look for:

* operation start without matching finish/error/cancel log;
* continuation bridges with ambiguous branching;
* streams that never finish;
* tasks alive after owner cancellation;
* missing terminal state in logs;
* UI state that remains loading after cancellation or error.

Validate with exactly-once continuation guarantees, timeout and cancellation handling, deliberate stream finishing, and cancellation/error-path tests.

## Instruments selection

Choose the tool based on the question.

| Question                                  | Useful tool                                                                | Look for                                                                                                                               |
| ----------------------------------------- | -------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| What consumed CPU?                        | Time Profiler                                                              | Expensive functions, CPU-heavy `@MainActor` work, synchronous parsing, decoding, sorting, formatting, compression, crypto, image work. |
| Why did UI freeze?                        | Hangs, responsiveness tools, Time Profiler                                 | Main-thread stalls, blocking calls, delayed UI state updates, long action-to-feedback intervals.                                       |
| Are threads blocked?                      | System Trace                                                               | Semaphore waits, lock contention, synchronous I/O, blocked cooperative threads, `queue.sync`, sleeping threads.                        |
| Why did memory spike?                     | Allocations, Memory Graph                                                  | Allocation bursts, retained closures, retained producers, retained bridge objects, large buffers, task captures.                       |
| What happened across a semantic interval? | Points of Interest / signposts                                             | Action-to-feedback, request-to-first-value, cancellation-to-stop, stream-subscribe-to-first-event intervals.                           |
| Is work still alive after cancellation?   | Logs, signposts, memory graph, concurrency/task instruments when available | Long-lived tasks, retained owners, producers not cleaned up, missing cancellation completion.                                          |
| Is actor or task behavior suspicious?     | Swift Concurrency / task / actor diagnostics when available                | Task explosions, long-lived tasks, actor executor buildup, tasks surviving cancellation.                                               |

Tool names and available Swift Concurrency diagnostics vary by Xcode version. Prefer the most specific concurrency, task, or actor instrument available, but fall back to Time Profiler, System Trace, signposts, logs, memory tools, and production metrics when the template is missing.

Do not rely on a tool name alone. Pick the signal that answers the question.

## Trace reading checklist

Before giving a recommendation from a trace, check:

* What exact symptom does the trace represent?
* Was it captured on a realistic device?
* Was it captured in Release or a release-like configuration when timing matters?
* Is this a cold path, warm path, repeated interaction, or stress path?
* Are inputs, network conditions, device class, and build settings comparable?
* Which queue, thread, actor, or task owns the slow interval?
* Is the slow work CPU-bound, blocked, waiting, suspended, serialized, retained, or over-parallelized?
* Does the UI need the result before showing feedback?
* Are there repeated tasks or actor calls that could be batched?
* Is work still running after cancellation or owner deallocation?
* Is peak memory caused by parallelism, buffering, captured values, or retained producers?
* Is the proposed change expected to reduce a visible interval or only move work elsewhere?
* Does the trace prove the concurrency boundary, or only show a symptom?

Prefer Release or release-like builds for performance comparisons. Debug builds are useful for logic checks and signpost wiring, but can distort timing, allocation behavior, optimizer-sensitive work, and SwiftUI/concurrency overhead.

If the trace does not support a conclusion, say what extra capture or signpost is needed.

## Signposts and logs

Use signposts when a trace needs semantic boundaries.

Good signpost intervals:

* user action to first feedback;
* screen appear to first data value;
* request start to cache hit or miss decision;
* cache miss to network completion;
* task group start to all children complete;
* task group start to first result;
* stream subscription to first event;
* stream cancellation to producer stopped;
* cancellation request to worker stopped;
* continuation created to continuation resumed;
* actor API entry to exit;
* actor batch start to batch complete.

Useful log fields include:

* operation id;
* request id;
* task owner;
* parent operation id;
* screen or feature name;
* actor or service name;
* input size;
* concurrency limit;
* cancellation reason;
* start time;
* finish time;
* result type;
* result source;
* timeout reason;
* retry count.

Avoid generic logs like “started task” or “finished task”. Logs should reveal duplicate work, missing completion, late cancellation, owner mismatch, or unexpected retention.

Keep logs and signposts low-overhead. Do not add high-volume logging inside hot loops unless it is sampled, gated, or temporary.

## Production signals

Use production signals when local traces cannot reproduce the issue or when tail latency matters.

Useful signals include:

* p50/p95/p99 latency;
* cancellation requested vs cancellation completed;
* duplicate request rate;
* active tasks per feature;
* memory peak during fan-out;
* stream subscriber count;
* producer lifetime;
* timeout rate;
* retry count;
* main-thread stall or hang reports;
* crash or watchdog trends;
* MetricKit hang, responsiveness, CPU, memory, disk, launch, or energy payloads when available.

Do not optimize only for mean latency. Concurrency problems often appear in p95 or p99 behavior.

MetricKit rarely proves the exact concurrency boundary by itself. Use it to identify affected cohorts, time windows, hang/memory/CPU/energy trends, and then reproduce locally or add signposts and logs to connect the production signal to a specific task, actor, stream, continuation, or blocking bridge.

When production signals show a regression, segment by:

* device class;
* OS version;
* app version;
* Low Power Mode when available;
* network quality;
* input size;
* feature path;
* account or data shape;
* cold vs warm path.

## Validation patterns

### Before/after trace comparison

Use when the proposed change affects latency, actor hops, task count, memory, or blocked work.

Compare:

* same device class;
* same build configuration;
* same data size;
* same network condition when possible;
* repeated runs;
* same signpost interval;
* same user path;
* same cold/warm state.

Report:

* previous value;
* new value;
* run count or variance;
* what changed in the trace;
* what did not change;
* whether the user-visible symptom improved.

A change that only moves work elsewhere is not enough. The user-visible interval, resource peak, cancellation behavior, or tail latency should improve.

### Cancellation validation

Use when work should stop after navigation, timeout, parent cancellation, replacement operation, or owner deallocation.

Check that:

* child work stops;
* stream cleanup runs;
* continuations finish correctly;
* no UI state update happens after cancellation;
* underlying requests or producers stop when appropriate;
* retained objects are released;
* cancellation is not converted into a user-visible error unless intended.

### Bounded parallelism validation

Use when replacing unbounded fan-out.

Compare:

* peak task count;
* peak memory;
* total latency;
* p95 latency;
* error behavior;
* cancellation behavior;
* throughput under realistic input size;
* backend rate-limit or retry behavior;
* old-device behavior when relevant.

A bounded implementation may slightly increase best-case latency while improving memory, stability, energy, and tail latency. Explain that trade-off.

### Actor batching validation

Use when reducing actor hops.

Compare:

* number of actor calls;
* high-level operation latency;
* actor wait time if measured;
* CPU cost of the batched work;
* correctness under concurrent callers;
* time spent inside actor isolation.

Do not batch so much work that the actor becomes blocked for longer. Prefer snapshotting required state inside the actor and performing heavy work outside actor isolation when possible.

### Stream cleanup validation

Use when fixing `AsyncStream` or long-running `AsyncSequence` code.

Check that:

* the producer starts only when needed;
* the producer stops on termination;
* buffering is bounded when needed;
* the stream reacts to consumer cancellation;
* the producer, observer, delegate, timer, or callback source is released after the stream ends;
* buffered values do not keep growing.

### Continuation validation

Use when fixing callback or delegate bridges.

Check that:

* success resumes exactly once;
* failure resumes exactly once;
* cancellation path is explicit;
* timeout path is explicit when needed;
* malformed callback result is handled;
* duplicate callback cannot resume twice;
* callback-after-cancel is safe;
* owner deallocation does not leave the task suspended forever;
* bridge objects are released after terminal completion.

### Controlled concurrency tests

Use when the bug involves cancellation, continuations, streams, duplicate work, or owner lifetime.

Use fake services that can:

* delay completion;
* complete successfully;
* fail;
* never complete;
* call back twice;
* call back after cancellation;
* record cancellation;
* record active subscriber count;
* record duplicate request count;
* expose when producers are started and stopped.

Verify:

* exactly-once completion;
* cancellation propagation;
* producer cleanup;
* duplicate request count;
* stale result suppression;
* no UI update after cancellation;
* retained objects are released after completion or cancellation.

Controlled tests do not replace Instruments for performance, but they are often better than Instruments for proving lifetime and cancellation correctness.

## Common mistakes

* Treating Instruments as a way to confirm a preferred theory instead of testing alternatives.
* Reporting CPU cost without identifying whether the UI was waiting for it.
* Treating suspension as blocking.
* Treating waiting on an actor as thread blocking.
* Ignoring task lifetime and only looking at function duration.
* Measuring only one local run.
* Comparing traces with different inputs, devices, network conditions, or build configurations.
* Using Debug traces as final performance evidence.
* Moving work off `MainActor` without checking UI state or API isolation requirements.
* Replacing sequential awaits with parallel tasks without measuring memory and cancellation behavior.
* Claiming cooperative executor starvation without blocked-thread or delayed-resumption evidence.
* Ignoring p95 and p99 latency.
* Missing cancellation paths in validation.
* Forgetting that a stream producer can outlive the consumer.
* Treating actor isolation as proof that duplicate work cannot happen.
* Optimizing continuation overhead before checking missing completion, blocking work, callback queue, or duplicate operations.
* Reporting “fixed” without a before/after signal.

## Report template

Use this structure when the user asks for a diagnosis, trace review, or validation plan.

```markdown
## Symptom

Describe the user-visible problem and when it happens.

## Evidence

List the trace, log, measurement, controlled test, or production signal that supports the finding.

## Likely concurrency boundary

Name the relevant `Task`, task group, actor, `MainActor` path, continuation, stream, async sequence, cancellation path, or legacy blocking API.

## Classification

Say whether the issue appears CPU-bound, blocked, suspended, waiting, serialized, retained, over-parallelized, or not proven yet.

## Finding

Explain the likely performance, lifetime, isolation, cancellation, or responsiveness issue.

## Recommendation

Give the smallest change that should affect the measured signal.

## Trade-off

Explain what the change improves and what it may complicate.

## Validation

Describe the before/after trace, signpost, controlled test, cancellation test, memory check, or production metric that would confirm the fix.

## Confidence

State whether the evidence proves the finding or whether more instrumentation is needed.
```
