# Network, Disk, and Power Profiling

Use this reference when the task involves slow networking, duplicated requests, caching behavior, disk reads/writes, persistence stalls, excessive logging, background work, wakeups, battery drain, or thermal pressure.

## Contents

- Core model
- When to use this reference
- Tool selection
- Network profiling
- Disk I/O profiling
- Power and thermal profiling
- Background work and wakeups
- Decision rules
- Common mistakes
- Validation
- Output notes

## Core model

Network, disk, and power issues often look like generic slowness from the user's point of view.

Do not start with a CPU-only explanation. A screen can be slow because it waits for a request waterfall, blocks on persistence, rewrites too much data, wakes the device too often, or keeps work alive after the visible UI no longer needs it.

First classify the dominant cost:

- network-bound: waiting for request setup, server response, payload download, retries, or dependency chains;
- disk-bound: reading, writing, serializing, fsyncing, logging, migrating, or compacting data;
- power-bound: repeated small work, wakeups, polling, sensors, location, networking, background tasks, or offscreen animation;
- mixed: network data triggers disk persistence, decoding, image work, UI updates, and cache writes.

Treat these as user-visible performance problems, not just infrastructure details.

## When to use this reference

Use this reference when the task mentions:

- slow screen loading caused by API calls, images, remote config, feature flags, or request waterfalls;
- duplicated requests, missing cache hits, retry storms, or late visible content;
- disk reads during launch or first screen construction;
- persistence stalls, database transactions, migration cost, cache writes, or large serialization;
- excessive logging, repeated full-file rewrites, or write amplification;
- battery drain, thermal pressure, high energy usage, frequent wakeups, polling, timers, or background work;
- MetricKit disk write, hang, CPU, or power-related production signals.

Do not use this reference as the primary guide for pure CPU hotspots, SwiftUI invalidation, memory leaks, or launch architecture unless network, disk, or power is the suspected measured cause.

## Tool selection

Choose the tool from the symptom and the evidence available.

| Symptom | Primary tool or signal | What it can show |
|---|---|---|
| Slow API-driven screen | Network instrument, URLSession metrics, signposts | request waterfall, latency, retries, payload timing |
| Duplicate requests | Network instrument, logs, signposts | repeated URL/session activity and missing coalescing |
| Image loading delay | Network instrument, Time Profiler, signposts | network wait, decode cost, cache behavior |
| Disk reads/writes | File Activity, System Trace | file operations, frequency, duration, call sites |
| Persistence stalls | Time Profiler, File Activity, signposts | serialization, transaction, blocking, main-thread stalls |
| Excessive writes in production | MetricKit disk write diagnostics, Organizer | release-level disk write pressure by device/version |
| Battery drain | Energy Log, Power Profiler, MetricKit, Organizer | wakeups, network, CPU, location, background activity |
| Thermal pressure | MetricKit, Organizer, device testing | sustained CPU/GPU/sensor/network work |
| Background work continues | Energy Log, signposts, logs | tasks continuing after UI disappears or app backgrounds |

Use signposts when system tools can show cost but cannot name the app-specific operation that caused it.

## Network profiling

### What to inspect

For slow networking, inspect:

- request waterfall and dependency chain;
- duplicated in-flight requests;
- DNS, TCP, TLS, and connection reuse;
- time to first byte;
- payload size and compression;
- cache headers and cache hit rate;
- retry behavior and backoff;
- request priority;
- image requests competing with first visible content;
- sequential requests that could be independent;
- hidden requests from SDKs, analytics, feature flags, or remote config.

### Interpretation rules

Separate these cases:

- The app is waiting for the backend.
- The app is making too many requests.
- The app is making the right requests in the wrong order.
- The app is downloading too much data.
- The app is not using cache effectively.
- The app receives data early but delays rendering because decode, persistence, or UI work follows.

Do not call a screen CPU-bound just because the visible symptom is a delay. If the main thread is mostly idle while the UI waits for network completion, the dominant issue is probably request structure, caching, server timing, or rendering dependency.

### Common network causes

Look for:

- N+1 endpoint patterns;
- duplicated fetches caused by repeated lifecycle callbacks;
- request creation in SwiftUI `body` or unstable `.task(id:)` inputs;
- missing request coalescing for the same resource;
- cache bypass caused by headers, URL variation, auth variation, or custom loaders;
- low-priority prefetches competing with visible content;
- retries without jitter or backoff;
- sequential dependency chains where partial rendering would be possible;
- analytics or SDK calls blocking visible work;
- images fetched at a larger size than displayed.

### Fix directions

Prefer fixes that reduce waiting on the visible path:

- coalesce identical in-flight requests;
- cache stable data and images with explicit invalidation rules;
- avoid starting the same request from multiple view lifecycle paths;
- parallelize independent requests only when it does not overload the backend or device;
- render progressively when the first useful content does not require all data;
- reduce payload size or request only visible fields;
- prioritize visible content over prefetches and low-value background work;
- add cancellation for requests tied to disappearing screens or obsolete queries;
- move non-critical remote config, analytics enrichment, or prefetching off the first interaction path.

## Disk I/O profiling

### What to inspect

For disk I/O, inspect:

- synchronous reads or writes on the main thread;
- file operations during launch, routing, or first screen construction;
- database open, migration, query, transaction, and checkpoint cost;
- repeated full-file rewrites;
- cache write amplification;
- excessive logging;
- serialization/deserialization cost;
- temporary file churn;
- image cache writes and reads;
- file coordination or locking;
- persistence work triggered by network completion.

### Interpretation rules

Separate wall-clock delay from CPU cost. Disk I/O can block the app even when CPU usage is not high.

Also separate:

- read cost from write cost;
- one-time migration from repeated steady-state cost;
- main-thread I/O from background I/O that still blocks UI through awaiting or locking;
- storage cost from serialization cost;
- local development behavior from production device behavior.

Do not assume that moving disk work to a background queue fixes the user-visible issue. If the first screen awaits the result, or if the main actor waits for a persistence actor that is blocked on disk, the user still waits.

### Common disk causes

Look for:

- reading large JSON or plist files during launch;
- opening or migrating a database before it is needed;
- saving after every small state change instead of batching;
- rewriting entire cache files for small mutations;
- logging large payloads or too frequently;
- writing analytics/event queues synchronously;
- storing decoded images or blobs inefficiently;
- repeated cache cleanup on app start;
- database queries without indexes;
- transactions that are too frequent or too broad;
- file locks shared by unrelated features.

### Fix directions

Prefer fixes that reduce blocking and write amplification:

- avoid disk access on the user-visible critical path unless required;
- lazily open stores that are not needed for the first screen;
- batch writes and use transactions intentionally;
- write deltas instead of rewriting full files;
- reduce logging volume, especially in production hot paths;
- move cleanup and compaction to safe idle/background moments;
- cache decoded or transformed data only when the write cost is worth it;
- add indexes or query limits when persistence is the bottleneck;
- keep persistence APIs async, but verify that callers do not immediately block waiting for them.

## Power and thermal profiling

### What to inspect

For battery drain or thermal pressure, inspect:

- timers and wakeup frequency;
- polling loops;
- background tasks and expiration behavior;
- location accuracy and update frequency;
- Bluetooth, sensors, camera, microphone, and motion updates;
- repeated small network requests;
- CPU work after the screen disappears;
- animations or display links running offscreen;
- unbounded tasks, retry loops, or streaming pipelines;
- excessive logging and disk writes;
- image/video processing;
- push, sync, and prefetch behavior.

### Interpretation rules

Power problems often come from sustained moderate work, not one obvious spike.

Look for patterns:

- work continues after the user leaves the screen;
- periodic timers wake the app without user value;
- background work runs longer than necessary;
- sensors use higher precision than the feature needs;
- networking is chatty instead of batched;
- UI animations keep running when not visible;
- retry loops turn a temporary failure into continuous work.

Do not evaluate battery or thermal behavior only in Simulator. Use a real device whenever possible.

### Fix directions

Prefer event-driven and bounded work:

- replace polling with push, notifications, callbacks, or state observation where practical;
- lower timer frequency or remove timers entirely;
- stop work when views disappear, tasks are cancelled, or the app backgrounds;
- use lower location accuracy and lower update frequency when acceptable;
- batch network and disk operations;
- add backoff and jitter to retries;
- stop offscreen animations and display links;
- keep background tasks short, cancellable, and purpose-specific;
- avoid prefetching that competes with visible content or drains battery without high hit rate.

## Background work and wakeups

Background work is not free just because it is not on the main thread.

Inspect:

- whether the task still has user value;
- whether it survives screen dismissal;
- whether it is cancelled when inputs change;
- whether it wakes the app too often;
- whether it holds locks or actors needed by visible UI;
- whether retries continue after failure;
- whether background network/disk work competes with foreground work.

Prefer a lifecycle-aware model:

```text
visible need -> start work -> cancel or reduce work when invisible -> persist only useful results -> verify with trace or metric
```

## Decision rules

- If a screen waits on network, first inspect the request waterfall before optimizing local code.
- If network completes early but UI appears late, inspect decode, persistence, image processing, and main-thread rendering after response completion.
- If disk I/O appears during launch or first screen construction, ask whether that data is required before first value.
- If production shows excessive disk writes, look for write frequency and amplification before focusing on single write latency.
- If battery drain is reported, look for repeated small work and wakeups, not only CPU hotspots.
- If thermal pressure appears after prolonged usage, inspect sustained work across CPU, GPU, sensors, networking, and background tasks.
- If a fix defers work, verify that it does not create a later hitch at first interaction.
- If a fix batches work, verify that it does not increase data loss risk or delay high-priority persistence.

## Common mistakes

- Treating a network-bound screen as a CPU problem because the user says it is “slow.”
- Ignoring request duplication caused by view lifecycle or repeated state changes.
- Assuming cache exists just because a caching layer is present.
- Moving disk work off the main thread while the UI still awaits the result.
- Measuring disk or power behavior only in Debug or Simulator.
- Optimizing a single file write while ignoring write frequency.
- Adding aggressive prefetching that improves one trace but hurts battery and network usage.
- Leaving timers, display links, streams, or tasks alive after the screen disappears.
- Treating MetricKit or Organizer as root-cause tools. They identify production signals; local traces explain causes.
- Claiming a power improvement without a repeated real-device measurement.

## Validation

For network changes, validate with:

- before/after request waterfall;
- request count;
- duplicate in-flight request count;
- time to first byte;
- payload size;
- cache hit rate;
- first useful content time;
- signposts around user-visible loading phases.

For disk changes, validate with:

- File Activity or System Trace before/after;
- number of reads/writes;
- bytes read/written;
- main-thread blocking time;
- transaction count;
- write amplification;
- MetricKit disk diagnostics when available;
- launch or first-screen timing if disk was on the critical path.

For power changes, validate with:

- repeated real-device runs;
- wakeup frequency;
- timer activity;
- background task duration;
- network and disk operation frequency;
- Energy Log or Power Profiler output;
- MetricKit or Organizer trends across releases;
- thermal behavior under sustained usage.

Use the same scenario, device class, OS version, build configuration, and data set for before/after comparison whenever possible.

## Output notes

When using this reference, include:

1. Whether the suspected cost is network-bound, disk-bound, power-bound, or mixed.
2. The primary tool or signal to inspect.
3. The strongest evidence needed.
4. The likely app-specific cause.
5. A focused fix direction.
6. A validation method.

Do not end with generic advice like “profile it.” Name the tool, the scenario, and the signal that would confirm or reject the hypothesis.
