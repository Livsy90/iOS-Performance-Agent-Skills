---
name: swift-concurrency-performance
description: Use this skill when reviewing Swift Concurrency performance and responsiveness, including task explosions, actor hopping, MainActor bottlenecks, cancellation, AsyncSequence cleanup, continuations, reentrancy, executor behavior, blocking async work, or async work that affects UI latency. Do not use it for general async/await syntax questions unless performance, responsiveness, cancellation, or lifetime is part of the task.
---

# Swift Concurrency Performance

## Purpose

Use this skill to review, diagnose, and improve Swift Concurrency code when async work affects UI responsiveness, throughput, memory, cancellation, task lifetime, actor contention, or correctness under load.

This skill is not a general Swift Concurrency tutorial. It is a performance and responsiveness review workflow.

## When to use this skill

Use this skill when the task involves:

* UI stalls, slow interactions, hangs, or frame drops related to async work;
* excessive `Task` creation, task groups, detached tasks, or unstructured concurrency;
* `MainActor` bottlenecks, actor hopping, actor queue buildup, or actor contention;
* cancellation that does not stop work, navigation leaks, or tasks outliving their owner;
* `AsyncSequence`, `AsyncStream`, long-running streams, buffering, or producer cleanup;
* continuation bridges, delegate/callback wrappers, or async wrappers around legacy APIs;
* blocking calls inside async contexts, semaphores, synchronous I/O, locks, or cooperative pool starvation;
* actor reentrancy, duplicate in-flight work, cache stampedes, or inconsistent actor state after `await`;
* Swift 6 isolation behavior, explicit isolation, `@concurrent`, `nonisolated`, or Sendable boundaries;
* Instruments traces, logs, or production signals that point to concurrency-related latency, memory growth, or throughput loss.

## When not to use this skill

Do not use this skill for:

* basic `async`/`await` syntax questions with no performance, lifetime, or responsiveness concern;
* general architecture discussions where concurrency is not part of the critical path;
* purely SwiftUI rendering issues unless async lifecycle work contributes to the symptom;
* launch performance unless async startup work, task lifetime, or actor isolation is part of the launch path;
* runtime-level allocation, ARC, generics, or existential costs unless they interact with concurrency behavior;
* server-side concurrency questions unless the task is specifically about Swift Concurrency performance patterns.

Prefer another skill when a more specific domain dominates the task:

* use `ios-launch-performance` for app startup, first frame, first interaction, pre-main, dyld, or SDK launch work;
* use `swiftui-performance` for SwiftUI invalidation, identity, layout, scrolling, or body cost;
* use `ios-performance-profiling` when the main task is choosing or interpreting profiling tools;
* use `swift-runtime-performance` for allocations, ARC traffic, dispatch, existentials, generics, or copy-on-write costs.

## Core workflow

1. Identify the user-visible symptom: UI stall, slow interaction, low throughput, memory growth, duplicate work, leaked task, missed cancellation, actor contention, or blocked cooperative threads.
2. Locate the async boundary: `Task`, task group, actor method, `MainActor`, continuation, stream, lifecycle callback, delegate bridge, or legacy blocking API.
3. Determine the lifetime owner: view, view model, service, actor, request, app session, stream consumer, or detached background process.
4. Separate required work from optional or deferrable work.
5. Check whether concurrency is being used to express structure, isolation, and cancellation rather than as a vague performance fix.
6. Look for blocking work inside async contexts.
7. Check whether work that affects UI state is isolated narrowly and whether CPU-heavy work is kept off the main actor.
8. Check cancellation propagation, especially across task groups, streams, continuations, loops, and navigation lifetimes.
9. Check for actor reentrancy after every `await` inside actor-isolated methods.
10. Propose the smallest safe change that improves lifetime, cancellation, isolation, or throughput.
11. Include a validation path before calling the change a performance improvement.

## Decision rules

* Treat `async` as suspension, not as automatic background execution.
* Do not assume concurrency improves performance. More tasks can increase scheduling overhead, memory pressure, actor contention, and cancellation complexity.
* Prefer structured concurrency when the parent owns the lifetime of the work.
* Use unstructured tasks only when the lifetime is deliberately independent and cancellation ownership is explicit.
* Use `Task.detached` only as an explicit escape hatch from inherited context, priority, task-local values, and actor isolation.
* Bound parallel work when the input size can grow.
* Keep `MainActor` work short and focused on UI state, presentation coordination, and main-thread-only APIs.
* Move CPU-heavy work outside main-actor isolation, but do not cross isolation boundaries casually.
* Batch actor calls on hot paths when repeated hops dominate latency.
* After an `await` inside an actor, assume actor state may have changed.
* Use checked continuations by default and verify every path resumes exactly once.
* Treat stream termination and producer cleanup as part of the API contract.
* Prefer cancellation-aware loops and pipelines for long-running or high-volume work.
* Connect every performance claim to evidence or a validation plan.

## Gotchas

* Do not recommend adding `async` or `Task` simply because code is slow.
* Do not move work off the `MainActor` if the API or UI state must remain main-actor isolated.
* Do not leave CPU-heavy computation in a `@MainActor` type just because the type also owns UI state.
* Do not use `Task.detached` to silence isolation errors without explaining the lifetime, cancellation, priority, and data-safety consequences.
* Do not create one child task per item for large or unbounded collections without limiting concurrency.
* Do not swallow cancellation with broad `catch` blocks.
* Do not assume cancelling a parent automatically stops legacy callbacks, streams, delegates, or manually retained producers.
* Do not wrap a blocking API in `async` if the underlying work still blocks a cooperative executor thread.
* Do not treat actor isolation as a duplicate-work prevention mechanism when the actor method suspends during a cache miss.
* Do not use unsafe continuations unless profiling shows checked continuation overhead matters and the resume contract is proven.
* Do not call an optimization successful without before/after validation.

## Reference routing

Read these only when relevant:

* `references/concurrency-runtime.md` — read when the task needs the mental model for tasks, suspension, cooperative executors, actor executors, priorities, structured concurrency, or why blocking async code is harmful.
* `references/mainactor-responsiveness.md` — read when the task involves `MainActor`, `@MainActor` types, UI state, view models, main-thread stalls, `@concurrent`, or moving CPU-heavy work away from UI isolation.
* `references/task-lifetime-and-structure.md` — read when the task involves structured concurrency, unstructured tasks, task ownership, `Task {}`, `Task.detached`, view/view-model lifetimes, or tasks that outlive their owner.
* `references/cancellation-and-task-lifetime.md` — read when the task involves navigation cancellation, long-running work, cancellation propagation, cancellation swallowed by `catch`, task groups, streams, or cancellation tests.
* `references/bounded-task-groups.md` — read when the task involves `withTaskGroup`, `withThrowingTaskGroup`, parallel mapping, fan-out work, memory spikes, or limiting concurrency.
* `references/actor-reentrancy.md` — read when the task involves actor-isolated state, duplicate network requests, cache stampedes, state checks before and after `await`, or actor queue buildup.
* `references/swift-6-isolation.md` — read when the task involves Swift 6 isolation behavior, default actor isolation, `@concurrent`, `nonisolated`, Sendable boundaries, or migration-related performance regressions.
* `references/blocking-legacy-apis.md` — read when the task involves semaphores, synchronous file I/O, blocking networking, locks, callback APIs, old SDKs, or async wrappers around blocking work.
* `references/continuation-safety.md` — read when the task involves `withCheckedContinuation`, `withCheckedThrowingContinuation`, delegate bridges, callback wrappers, timeout paths, cancellation paths, or exactly-once resume guarantees.
* `references/asyncsequence-and-stream-cleanup.md` — read when the task involves `AsyncSequence`, `AsyncStream`, `AsyncThrowingStream`, long-running streams, buffering, producer lifetime, `onTermination`, or `for await` loops.
* `references/diagnostics-and-instruments.md` — read when the user provides traces, logs, measurements, production signals, or asks how to validate concurrency-related performance changes.

## Validation expectations

Recommend validation that matches the suspected issue:

* use Instruments when the symptom involves UI stalls, actor contention, task lifetime, blocked threads, or high task counts;
* use signposts when comparing before/after latency across async boundaries;
* use cancellation tests when work should stop after navigation, deallocation, timeout, or parent cancellation;
* use memory graphs or allocation instruments when streams, task groups, or long-lived tasks may retain producers or large values;
* use logs with task identifiers or request identifiers when checking duplicate in-flight work;
* use XCTest performance tests only when the workload is repeatable enough to produce meaningful comparisons;
* use production metrics when local traces cannot reproduce tail latency or rare stuck tasks.

Do not present a concurrency refactor as a performance win unless there is a clear validation path.

## Output expectations

When reviewing code, respond with:

1. **Finding** — the likely concurrency performance, lifetime, isolation, or responsiveness issue.
2. **Why it matters** — the impact on UI latency, throughput, memory, cancellation, actor contention, or correctness.
3. **Evidence** — the code pattern, trace symptom, lifecycle mismatch, missing cancellation path, blocking call, actor hop pattern, or continuation/stream contract issue.
4. **Recommended change** — the smallest safe change first; avoid broad rewrites unless the design itself causes the issue.
5. **Trade-offs** — what the change improves and what it may complicate.
6. **Validation** — how to verify the result with Instruments, signposts, cancellation tests, logs, memory tools, UI behavior, or production metrics.

When the task asks for an investigation plan, respond with:

1. the symptom to reproduce;
2. the suspected async boundary;
3. the likely lifetime or isolation owner;
4. the first trace or log to collect;
5. the signal that would confirm or reject the hypothesis;
6. the smallest next code area to inspect.

When the task asks for an explanation, keep it practical:

1. explain the model briefly;
2. show one concrete iOS or Swift example only if needed;
3. name the common misconception;
4. include a validation or debugging technique.
   ::: 
