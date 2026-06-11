# Diagnostics and Instruments for Swift Concurrency Performance

Use this reference when the user provides traces, logs, measurements, production signals, or asks how to validate concurrency-related performance changes.

This file helps the agent connect Swift Concurrency recommendations to evidence instead of guessing.

## Contents

- [Core model](#core-model)
- [Diagnostic workflow](#diagnostic-workflow)
- [Symptom mapping](#symptom-mapping)
- [Instruments selection](#instruments-selection)
- [Trace reading checklist](#trace-reading-checklist)
- [Signposts and logs](#signposts-and-logs)
- [Production signals](#production-signals)
- [Validation patterns](#validation-patterns)
- [Common mistakes](#common-mistakes)
- [Report template](#report-template)

## Core model

Swift Concurrency performance problems usually appear as one of these outcomes:

- UI work waits too long before it can run.
- Work runs on the wrong isolation domain.
- Too many tasks are created for the amount of useful work.
- A cooperative executor thread is blocked by synchronous work.
- An actor becomes a serialization bottleneck.
- Work continues after the user no longer needs it.
- Async bridges never finish, finish twice, or keep producers alive.
- Parallelism increases memory pressure instead of reducing latency.

Diagnostics should answer three questions:

1. What user-visible symptom happened?
2. Which async boundary or isolation boundary is involved?
3. What evidence proves the proposed fix affects that boundary?

Do not treat “uses async/await” as evidence. Async code can still block, serialize, leak, duplicate work, or delay UI state.

## Diagnostic workflow

Use this workflow when reviewing a trace, logs, metrics, or a proposed optimization.

1. Define the user-visible symptom: UI stall, delayed screen load, duplicate work, memory spike, stuck loading state, high tail latency, or work continuing after navigation.
2. Identify the async boundary: `Task`, `Task.detached`, task group, actor method, `MainActor`, continuation, stream, `for await` loop, callback bridge, or synchronous legacy API.
3. Identify the owner of the work: view, view model, request, service, actor, stream consumer, app session, or detached process.
4. Check whether the work should still be alive after dismissal, cancellation, timeout, parent cancellation, stream termination, or owner deallocation.
5. Connect the symptom to a measurable signal.
6. Recommend the smallest fix that changes the measured signal.
7. Define a before/after validation.

Useful evidence includes main-thread stalls, actor queue buildup, high task counts, blocked threads, repeated request ids, retained producers, allocation bursts, missing cancellation logs, or continuations without completion.

## Symptom mapping

### UI stall or slow interaction

Likely causes:

- heavy work running on `MainActor`;
- synchronous blocking call on the main path;
- too many actor hops before UI state updates;
- UI awaiting a long async dependency;
- cooperative pool starvation causing delayed resumption.

Look for long main-thread samples, CPU-heavy `@MainActor` methods, action-to-feedback intervals with several awaits before visible feedback, and repeated hops between UI code and actors.

Validate by comparing tap-to-feedback or action-to-render signpost intervals before and after. Move only CPU-heavy or blocking work out of UI isolation, and keep UI state mutation on `MainActor`.

### Actor queue buildup or low throughput

Likely causes:

- hot actor receiving too many small calls;
- long actor-isolated sections;
- blocking work inside actor methods;
- chatty API that forces repeated isolation hops;
- actor method suspending and resuming with stale assumptions.

Look for many calls to the same actor on a hot path, serialized execution where independent work could happen outside the actor, repeated await chains, or logs showing growing actor wait time.

Validate by batching actor operations, moving non-state work outside actor isolation, reducing per-item actor calls, and comparing request latency or actor wait time.

### High task count or task explosion

Likely causes:

- one child task per item in a large collection;
- repeated `.task` creation from view lifecycle churn;
- unstructured tasks created without ownership;
- task groups with unbounded fan-out;
- retry loops creating new tasks without cancelling old work.

Look for many short-lived tasks with similar call stacks, tasks surviving longer than their owner, or memory growth that tracks task count.

Validate by bounding concurrency, reusing a parent task where appropriate, storing and cancelling unstructured tasks, and comparing peak task count, peak memory, and total latency.

### Memory spike during async work

Likely causes:

- unbounded task group;
- collecting all results before yielding;
- `AsyncStream` buffering without a limit;
- tasks retaining large values;
- producers retained after stream termination;
- detached task retaining `self` or services longer than expected.

Look for allocation bursts aligned with fan-out, growing buffers, retained producers, retained callbacks, or task closures capturing large objects.

Validate by bounding parallelism, streaming partial results, configuring buffering policy, cleaning up producers in `onTermination`, and comparing peak memory.

### Duplicate network work or cache stampede

Likely causes:

- actor reentrancy after `await`;
- no in-flight request tracking;
- cache check happens before suspension but is not rechecked after resumption;
- multiple tasks independently request the same resource.

Look for repeated request ids or URLs, actor methods that check cache before awaiting network, and logs showing multiple callers entering the same miss path.

Validate by storing in-flight tasks, re-checking state after `await`, coalescing duplicate work, and comparing duplicate request count.

### Stuck loading state or never-ending task

Likely causes:

- continuation not resumed on every path;
- callback bridge misses failure, timeout, cancellation, or early return path;
- `AsyncStream` never finishes;
- `for await` loop waits forever;
- cancellation swallowed by broad `catch`.

Look for operation start without matching finish/error/cancel log, continuation bridges with ambiguous branching, streams that never finish, or tasks alive after owner cancellation.

Validate with exactly-once continuation guarantees, timeout and cancellation handling, deliberate stream finishing, and cancellation/error-path tests.

## Instruments selection

Choose the tool based on the question.

| Question | Useful tool | Look for |
|---|---|---|
| What consumed CPU? | Time Profiler | Expensive functions, CPU-heavy `@MainActor` work, synchronous parsing, decoding, sorting, formatting, compression, crypto, or image work. |
| Why did UI freeze? | Hangs, responsiveness tools, Time Profiler | Main-thread stalls, blocking calls, delayed UI state updates, long action-to-feedback intervals. |
| How many tasks exist? | Swift Concurrency instrument/template when available | Task explosions, long-lived tasks, actor executor buildup, tasks surviving cancellation. |
| Are threads blocked? | System Trace | Semaphore waits, lock contention, synchronous I/O, blocked cooperative threads. |
| Why did memory spike? | Allocations, memory graph | Allocation bursts, retained closures, retained producers, large buffers, task captures. |
| How do we compare before/after? | Points of Interest / signposts | Action-to-feedback, request-to-first-value, cancellation-to-stop, stream-subscribe-to-first-event intervals. |

When a tool is unavailable in the installed Xcode version, choose the closest observable signal: signposts, logs, Time Profiler, memory tools, or production metrics.

## Trace reading checklist

Before giving a recommendation from a trace, check:

- What exact symptom does the trace represent?
- Was it captured on a realistic device and build configuration?
- Is this a cold path, warm path, repeated interaction, or stress path?
- Which queue, thread, actor, or task owns the slow interval?
- Is the slow work CPU-bound, blocked, waiting, suspended, or serialized?
- Does the UI need the result before showing feedback?
- Are there repeated tasks or actor calls that could be batched?
- Is work still running after cancellation or owner deallocation?
- Is peak memory caused by parallelism, buffering, or retained producers?
- Is the proposed change expected to reduce a visible interval or only move work elsewhere?

If the trace does not support a conclusion, say what extra capture or signpost is needed.

## Signposts and logs

Use signposts when a trace needs semantic boundaries.

Good signpost intervals:

- user action to first feedback;
- screen appear to first data value;
- request start to cache hit or miss decision;
- cache miss to network completion;
- task group start to all children complete;
- stream subscription to first event;
- cancellation request to worker stopped;
- continuation created to continuation resumed.

Useful log fields include operation id, request id, task owner, parent operation id, screen or feature name, cancellation reason, start time, finish time, result type, and result source.

Avoid generic logs like “started task” or “finished task”. Logs should reveal duplicate work, missing completion, late cancellation, or owner mismatch.

## Production signals

Use production signals when local traces cannot reproduce the issue or when tail latency matters.

Useful signals include p50/p95/p99 latency, cancellation requested vs completed, duplicate request rate, active tasks per feature, memory peak during fan-out, stream subscriber count, producer lifetime, timeout rate, retry count, main-thread stall or hang reports, and MetricKit hang or responsiveness payloads when available.

Do not optimize only for mean latency. Concurrency problems often appear in p95 or p99 behavior.

## Validation patterns

### Before/after trace comparison

Use when the proposed change affects latency, actor hops, task count, or blocked work.

Compare the same device class, build configuration, data size, network condition when possible, repeated runs, and the same signpost interval.

Report the previous value, new value, run count or variance, what changed in the trace, and whether the user-visible symptom improved.

### Cancellation validation

Use when work should stop after navigation, timeout, parent cancellation, or owner deallocation.

Check that child work stops, stream cleanup runs, continuations finish correctly, no UI state update happens after cancellation, and retained objects are released.

### Bounded parallelism validation

Use when replacing unbounded fan-out.

Compare peak task count, peak memory, total latency, p95 latency, error behavior, cancellation behavior, and throughput under realistic input size.

A bounded implementation may slightly increase best-case latency while improving memory, stability, and tail latency. Explain that trade-off.

### Actor batching validation

Use when reducing actor hops.

Compare number of actor calls, high-level operation latency, actor wait time if measured, CPU cost of the batched work, and correctness under concurrent callers.

Do not batch so much work that the actor becomes blocked for longer.

### Stream cleanup validation

Use when fixing `AsyncStream` or long-running `AsyncSequence` code.

Check that the producer starts only when needed, stops on termination, does not buffer without bound, reacts to consumer cancellation, and releases after the stream ends.

## Common mistakes

- Treating Instruments as a way to confirm a preferred theory instead of testing alternatives.
- Reporting CPU cost without identifying whether the UI was waiting for it.
- Ignoring task lifetime and only looking at function duration.
- Measuring only one local run or comparing traces with different inputs, devices, or build configurations.
- Moving work off `MainActor` without checking UI state or API isolation requirements.
- Replacing sequential awaits with parallel tasks without measuring memory and cancellation behavior.
- Ignoring p95 and p99 latency.
- Missing cancellation paths in validation.
- Forgetting that a stream producer can outlive the consumer.
- Treating actor isolation as proof that duplicate work cannot happen.

## Report template

Use this structure when the user asks for a diagnosis, trace review, or validation plan.

```markdown
## Symptom

Describe the user-visible problem and when it happens.

## Evidence

List the trace, log, measurement, or production signal that supports the finding.

## Likely concurrency boundary

Name the relevant `Task`, task group, actor, `MainActor` path, continuation, stream, or legacy blocking API.

## Finding

Explain the likely performance, lifetime, isolation, cancellation, or responsiveness issue.

## Recommendation

Give the smallest change that should affect the measured signal.

## Trade-off

Explain what the change improves and what it may complicate.

## Validation

Describe the before/after trace, signpost, cancellation test, memory check, or production metric that would confirm the fix.
```
