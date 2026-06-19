# Network, Disk, and Power Profiling

Use this reference when the task involves slow networking, duplicated requests, caching behavior, disk reads/writes, persistence stalls, excessive logging, background work, wakeups, battery drain, or thermal pressure.

This file helps the agent separate network-bound, disk-bound, power-bound, and mixed performance problems. Do not explain every slow screen as CPU work.

## Scope Boundary

This reference covers:

* network waterfalls, duplicated requests, caching, retries, priorities, and URLSession timing;
* disk reads, writes, persistence stalls, migrations, logging, and write amplification;
* power, wakeups, polling, background work, sensors, location, and thermal pressure;
* background work that continues after the visible UI no longer needs it;
* evidence-driven routing to Network, File Activity, System Trace, Energy, MetricKit, Organizer, and signposts.

This reference does not cover:

* pure CPU hotspots unless they are triggered by network, disk, or power-related work;
* SwiftUI invalidation or animation hitches unless they overlap network, disk, or power evidence;
* memory leaks unless cache, image, or VM growth is the suspected cause;
* low-level networking protocol design;
* server-side performance except as observed through app-side timing evidence;
* generic architecture advice without measured network, disk, or power symptoms.

## Contents

* [Core Model](#core-model)
* [When to Use This Reference](#when-to-use-this-reference)
* [Tool Selection](#tool-selection)
* [Network Profiling](#network-profiling)
* [Disk I/O Profiling](#disk-io-profiling)
* [Power and Thermal Profiling](#power-and-thermal-profiling)
* [Background Work and Wakeups](#background-work-and-wakeups)
* [Signposts for Network, Disk, and Power Work](#signposts-for-network-disk-and-power-work)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [Decision Rules](#decision-rules)
* [Common Mistakes](#common-mistakes)
* [Validation](#validation)
* [Output Notes](#output-notes)

## Core Model

Network, disk, and power issues often look like generic slowness from the user's point of view.

Do not start with a CPU-only explanation. A screen can be slow because it waits for a request waterfall, blocks on persistence, rewrites too much data, wakes the device too often, or keeps work alive after the visible UI no longer needs it.

First classify the dominant cost:

* **network-bound** — waiting for request setup, server response, payload download, retries, cache validation, or dependency chains;
* **disk-bound** — reading, writing, serializing, fsyncing, logging, migrating, checkpointing, or compacting data;
* **power-bound** — repeated small work, wakeups, polling, sensors, location, networking, background tasks, or offscreen animation;
* **mixed** — network data triggers disk persistence, decoding, image work, UI updates, cache writes, logging, or retries.

Treat these as user-visible performance problems, not just infrastructure details.

A request can complete quickly while the screen still appears late because decoding, persistence, image decompression, model mapping, or UI updates happen after the response. A disk operation can run off the main thread but still block visible progress if the UI awaits it. Background work can still hurt foreground responsiveness, energy, and thermal behavior.

## When to Use This Reference

Use this reference when the task mentions:

* slow screen loading caused by API calls, images, remote config, feature flags, or request waterfalls;
* duplicated requests, missing cache hits, retry storms, or late visible content;
* disk reads during launch or first screen construction;
* persistence stalls, database transactions, migration cost, cache writes, or large serialization;
* excessive logging, repeated full-file rewrites, or write amplification;
* battery drain, thermal pressure, high energy usage, frequent wakeups, polling, timers, or background work;
* location, Bluetooth, motion, camera, microphone, sensors, streaming, or background sync staying active too long;
* MetricKit disk write, hang, CPU, or power-related production signals.

Do not use this reference as the primary guide for pure CPU hotspots, SwiftUI invalidation, memory leaks, or launch architecture unless network, disk, or power is the suspected measured cause.

## Tool Selection

Choose the tool from the symptom and the evidence available.

| Symptom                        | Primary tool or signal                                 | What it can show                                                               |
| ------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------------------------ |
| Slow API-driven screen         | Network instrument, `URLSessionTaskMetrics`, signposts | request waterfall, latency, retries, redirects, cache behavior, payload timing |
| Duplicate requests             | Network instrument, logs, signposts                    | repeated URL/session activity and missing coalescing                           |
| Image loading delay            | Network instrument, Time Profiler, signposts           | network wait, decode cost, cache behavior                                      |
| Disk reads/writes              | File Activity, System Trace                            | file operations, frequency, duration, call sites                               |
| Persistence stalls             | Time Profiler, File Activity, signposts                | serialization, transaction, query, locking, blocking, main-thread stalls       |
| Excessive writes in production | MetricKit disk write diagnostics, Organizer            | release-level disk write pressure by device/version                            |
| Battery drain                  | Energy Log, Power Profiler, MetricKit, Organizer       | wakeups, network, CPU, location, sensors, background activity                  |
| Thermal pressure               | MetricKit, Organizer, real-device sustained testing    | sustained CPU/GPU/sensor/network work                                          |
| Background work continues      | Energy Log, signposts, logs                            | tasks continuing after UI disappears or app backgrounds                        |

Use signposts when system tools can show cost but cannot name the app-specific operation that caused it.

Use production tools to identify affected versions, devices, and cohorts. Use local traces and signposts to explain root cause.

## Network Profiling

### What to Inspect

For slow networking, inspect:

* request waterfall and dependency chain;
* duplicated in-flight requests;
* DNS, TCP, TLS, redirects, and connection reuse;
* time to first byte;
* payload size and compression;
* cache headers and cache hit rate;
* cache validation behavior;
* retry behavior, jitter, and backoff;
* request priority;
* cancellation of obsolete requests;
* image requests competing with first visible content;
* sequential requests that could be independent;
* hidden requests from SDKs, analytics, feature flags, remote config, attribution, or logging.

Use `URLSessionTaskMetrics` when available to separate DNS, TCP, TLS, request, response, transfer, redirect, cache, and connection reuse behavior.

### Interpretation Rules

Separate these cases:

* The app is waiting for the backend.
* The app is making too many requests.
* The app is making the right requests in the wrong order.
* The app is downloading too much data.
* The app is not using cache effectively.
* The app has cache hits, but cached data still needs expensive decoding, disk reads, image decompression, or broad UI updates.
* The app receives data early but delays rendering because decode, persistence, image processing, or UI work follows.

Do not call a screen CPU-bound just because the visible symptom is a delay. If the main thread is mostly idle while the UI waits for network completion, the dominant issue is probably request structure, caching, server timing, or rendering dependency.

A cancelled UI request is not necessarily wasted work if the user changed intent. The problem is obsolete requests continuing to consume bandwidth, disk cache writes, decoding, or main-thread updates after the result is no longer needed.

### Common Network Causes

Look for:

* N+1 endpoint patterns;
* duplicated fetches caused by repeated lifecycle callbacks;
* request creation in SwiftUI `body` or unstable `.task(id:)` inputs;
* missing request coalescing for the same resource;
* cache bypass caused by headers, URL variation, auth variation, query parameter variation, or custom loaders;
* low-priority prefetches competing with visible content;
* image requests competing with text or first meaningful content;
* retries without jitter or backoff;
* sequential dependency chains where partial rendering would be possible;
* analytics, attribution, remote config, or SDK calls blocking visible work;
* images fetched at a larger size than displayed.

### Fix Directions

Prefer fixes that reduce waiting on the visible path:

* coalesce identical in-flight requests;
* cache stable data and images with explicit invalidation rules;
* avoid starting the same request from multiple view lifecycle paths;
* parallelize independent requests only when it does not overload the backend, radio, or device;
* render progressively when the first useful content does not require all data;
* reduce payload size or request only visible fields;
* prioritize visible content over prefetches and low-value background work;
* ensure prefetch and image requests have lower priority than first visible content;
* add cancellation for requests tied to disappearing screens, obsolete queries, or changed user intent;
* move non-critical remote config, analytics enrichment, attribution upload, or prefetching off the first interaction path.

## Disk I/O Profiling

### What to Inspect

For disk I/O, inspect:

* synchronous reads or writes on the main thread;
* file operations during launch, routing, or first screen construction;
* database open, migration, query, transaction, checkpoint, and save cost;
* repeated full-file rewrites;
* cache write amplification;
* excessive logging;
* serialization/deserialization cost;
* temporary file churn;
* image cache writes and reads;
* file coordination or locking;
* persistence work triggered by network completion;
* broad state updates triggered by persistence completion.

For SQLite, Core Data, or SwiftData, separate store opening, migration, query execution, faulting, relationship traversal, checkpointing, transaction cost, and save cost.

Memory-mapped files and databases may appear in VM Tracker rather than as obvious File Activity cost.

### Interpretation Rules

Separate wall-clock delay from CPU cost. Disk I/O can block the app even when CPU usage is not high.

Also separate:

* read cost from write cost;
* one-time migration from repeated steady-state cost;
* main-thread I/O from background I/O that still blocks UI through awaiting or locking;
* storage cost from serialization cost;
* query execution from object materialization or faulting;
* local development behavior from production device behavior.

Do not assume that moving disk work to a background queue fixes the user-visible issue. If the first screen awaits the result, or if the main actor waits for a persistence actor that is blocked on disk, the user still waits.

Background disk work can still hurt UI if it saturates storage, holds a database lock, blocks a persistence actor, triggers large notifications, or publishes broad state updates on the main actor.

### Common Disk Causes

Look for:

* reading large JSON or plist files during launch;
* opening or migrating a database before it is needed;
* saving after every small state change instead of batching;
* rewriting entire cache files for small mutations;
* logging large payloads or too frequently;
* writing analytics/event queues synchronously;
* storing decoded images or blobs inefficiently;
* repeated cache cleanup on app start;
* database queries without indexes;
* excessive faulting or relationship traversal;
* transactions that are too frequent or too broad;
* file locks shared by unrelated features.

Logging can become disk and power cost when it serializes large payloads, writes synchronously, flushes frequently, or logs in hot loops. Prefer bounded, sampled, asynchronous logging with privacy-safe payloads.

### Fix Directions

Prefer fixes that reduce blocking and write amplification:

* avoid disk access on the user-visible critical path unless required;
* lazily open stores that are not needed for the first screen;
* batch writes and use transactions intentionally;
* write deltas instead of rewriting full files;
* reduce logging volume, especially in production hot paths;
* move cleanup and compaction to safe idle/background moments;
* cache decoded or transformed data only when the write cost is worth it;
* add indexes or query limits when persistence is the bottleneck;
* reduce unnecessary faulting or relationship traversal;
* keep persistence APIs async, but verify that callers do not immediately block waiting for them;
* define durability requirements before delaying or batching writes.

## Power and Thermal Profiling

### What to Inspect

For battery drain or thermal pressure, inspect:

* timers and wakeup frequency;
* polling loops;
* background tasks and expiration behavior;
* location accuracy and update frequency;
* Bluetooth, sensors, camera, microphone, and motion updates;
* repeated small network requests;
* CPU work after the screen disappears;
* animations or display links running offscreen;
* unbounded tasks, retry loops, or streaming pipelines;
* excessive logging and disk writes;
* image/video processing;
* push, sync, and prefetch behavior;
* adaptation to Low Power Mode and thermal state.

### Interpretation Rules

Power problems often come from sustained moderate work, not one obvious spike.

Look for patterns:

* work continues after the user leaves the screen;
* periodic timers wake the app without user value;
* background work runs longer than necessary;
* sensors use higher precision than the feature needs;
* networking is chatty instead of batched;
* UI animations keep running when not visible;
* retry loops turn a temporary failure into continuous work;
* the app does the same work after every foreground transition;
* the app ignores Low Power Mode or thermal pressure.

Do not evaluate battery or thermal behavior only in Simulator. Use a real device whenever possible.

### Fix Directions

Prefer event-driven and bounded work:

* replace polling with push, notifications, callbacks, or state observation where practical;
* lower timer frequency or remove timers entirely;
* stop work when views disappear, tasks are cancelled, or the app backgrounds;
* use lower location accuracy and lower update frequency when acceptable;
* stop or reduce Bluetooth, motion, camera, microphone, and sensor updates when not visible or not needed;
* batch network and disk operations;
* add backoff and jitter to retries;
* stop offscreen animations and display links;
* keep background tasks short, cancellable, and purpose-specific;
* avoid prefetching that competes with visible content or drains battery without high hit rate;
* reduce polling, prefetching, animation, sensor accuracy, background sync, or processing quality under Low Power Mode or thermal pressure.

For location, motion, Bluetooth, camera, microphone, and sensors, verify start/stop ownership. The object that starts updates should have a clear condition for lowering accuracy, pausing, or stopping updates.

## Background Work and Wakeups

Background work is not free just because it is not on the main thread.

Inspect:

* whether the task still has user value;
* whether it survives screen dismissal;
* whether it is cancelled when inputs change;
* whether it wakes the app too often;
* whether it holds locks or actors needed by visible UI;
* whether retries continue after failure;
* whether background network/disk work competes with foreground work;
* whether background work resumes or duplicates when the app returns foreground.

For `BGTaskScheduler`, background `URLSession`, silent push, and background fetch, check expiration handling, idempotency, retry policy, cancellation, and whether work resumes or duplicates when the app returns foreground.

Prefer a lifecycle-aware model:

```text id="lnijhx"
visible need -> start work -> cancel or reduce work when invisible -> persist only useful results -> verify with trace or metric
```

If work is deferred from launch, verify that it does not become a post-first-frame burst that hurts first interaction, battery, disk, network, or thermal behavior.

## Signposts for Network, Disk, and Power Work

Use signposts when system tools show cost but cannot name the app-specific operation.

Useful signpost regions include:

* screen data load;
* API waterfall;
* image load/decode/display;
* cache read/write;
* database open;
* migration;
* query and mapping;
* save transaction;
* log flush;
* background sync;
* location or sensor session;
* retry loop;
* first useful content;
* first interaction readiness.

Example:

```swift id="jpws9s"
import os

private let signposter = OSSignposter(
    subsystem: "com.example.app",
    category: "Network"
)

func loadHome() async throws {
    let state = signposter.beginInterval("Home API waterfall")
    defer { signposter.endInterval("Home API waterfall", state) }

    try await homeService.load()
}
```

For async work, make sure the signpost interval covers the awaited operation you care about. A signpost around a function that only starts a detached task will not measure the detached work.

Do not signpost every helper. Instrument user-visible operations and suspected expensive regions.

## What the Agent Can Inspect

When repository access is available, inspect concrete network, disk, and power risks instead of giving generic advice.

Search for network entry points:

```sh id="i1mf8f"
rg "URLSession|URLRequest|dataTask|downloadTask|uploadTask|async.*throws|Alamofire|Moya|Apollo|GraphQL|WebSocket" .
```

Search for duplicate lifecycle-triggered requests:

```sh id="xjrej7"
rg "\.task\s*\{|\.onAppear\s*\{|viewDidAppear|viewWillAppear|scenePhase|onChange\(" .
```

Search for caching and request identity:

```sh id="v4yq37"
rg "URLCache|cachePolicy|Cache-Control|ETag|If-None-Match|NSCache|cacheKey|cachedResponse|reloadIgnoring" .
```

Search for disk and persistence:

```sh id="w98w4t"
rg "FileManager|Data\(|contentsOf:|write\(|fsync|sqlite|CoreData|SwiftData|ModelContainer|save\(|migrate|checkpoint|UserDefaults" .
```

Search for logging and write amplification:

```sh id="ex5m99"
rg "Logger|OSLog|print\(|debugPrint|log\(|flush|writeToFile|append" .
```

Search for power and background work:

```sh id="h2z1s4"
rg "Timer|CADisplayLink|poll|retry|BGTaskScheduler|beginBackgroundTask|CLLocationManager|CMMotionManager|CBCentralManager|AVCaptureSession|URLSessionConfiguration.background" .
```

Search for cancellation and lifecycle cleanup:

```sh id="yv6pe0"
rg "cancel\(|invalidate\(|stopUpdating|stopMonitoring|stop\(|deinit|task.cancel|withTaskCancellationHandler" .
```

Search for Low Power Mode and thermal adaptation:

```sh id="js1u08"
rg "isLowPowerModeEnabled|NSProcessInfoPowerStateDidChange|thermalState|ProcessInfo.processInfo" .
```

Use matches as leads, not proof. Confirm with traces, metrics, logs, signposts, or production evidence.

The agent can:

* classify a symptom as network-bound, disk-bound, power-bound, or mixed;
* identify likely request waterfalls, duplicate requests, cache bypasses, disk stalls, write amplification, polling, wakeups, and background work;
* recommend the right profiling tool or signal;
* propose signposts around app-owned phases;
* propose focused fixes tied to evidence;
* define before/after validation for the same scenario.

The agent cannot reliably:

* prove backend root cause without request timing evidence;
* prove production impact from one local trace;
* prove a power fix from Simulator behavior;
* treat a caching layer as effective without cache-hit and visible-content evidence;
* assume background work is harmless because it is off the main thread;
* promise battery or latency improvement without repeated measurement.

## Decision Rules

* If a screen waits on network, first inspect the request waterfall before optimizing local code.
* If network completes early but UI appears late, inspect decode, persistence, image processing, and main-thread rendering after response completion.
* If disk I/O appears during launch or first screen construction, ask whether that data is required before first value.
* If production shows excessive disk writes, look for write frequency and amplification before focusing on single write latency.
* If battery drain is reported, look for repeated small work and wakeups, not only CPU hotspots.
* If thermal pressure appears after prolonged usage, inspect sustained work across CPU, GPU, sensors, networking, and background tasks.
* If background work holds a lock, actor, database, or file coordination path needed by visible UI, treat it as a foreground responsiveness risk.
* If a fix defers work, verify that it does not create a later hitch at first interaction.
* If a fix batches work, verify that it does not increase data loss risk or delay high-priority persistence.
* If a fix lowers sensor accuracy, polling frequency, or refresh frequency, verify that product correctness still holds.

## Common Mistakes

* Treating a network-bound screen as a CPU problem because the user says it is “slow.”
* Ignoring request duplication caused by view lifecycle or repeated state changes.
* Assuming cache exists just because a caching layer is present.
* Treating a cache hit as free when decoding, disk access, decompression, or UI updates still block visibility.
* Moving disk work off the main thread while the UI still awaits the result.
* Measuring disk or power behavior only in Debug or Simulator.
* Optimizing a single file write while ignoring write frequency.
* Adding aggressive prefetching that improves one trace but hurts battery and network usage.
* Leaving timers, display links, streams, sensors, or tasks alive after the screen disappears.
* Treating background work as free because it is not on the main thread.
* Deferring work from launch into a burst that hurts first interaction, battery, disk, network, or thermal behavior.
* Treating MetricKit or Organizer as root-cause tools. They identify production signals; local traces explain causes.
* Claiming a power improvement without a repeated real-device measurement.

## Validation

For network changes, validate with:

* before/after request waterfall;
* request count;
* duplicate in-flight request count;
* time to first byte;
* redirect and connection reuse behavior;
* payload size;
* cache hit rate;
* first useful content time;
* cancellation of obsolete requests;
* signposts around user-visible loading phases.

For disk changes, validate with:

* File Activity or System Trace before/after;
* number of reads/writes;
* bytes read/written;
* main-thread blocking time;
* transaction count;
* query/faulting/mapping time when persistence was involved;
* write amplification;
* MetricKit disk diagnostics when available;
* launch or first-screen timing if disk was on the critical path.

For power changes, validate with:

* repeated real-device runs;
* wakeup frequency;
* timer activity;
* background task duration;
* network and disk operation frequency;
* location, sensor, camera, microphone, or Bluetooth activity duration;
* Energy Log or Power Profiler output;
* MetricKit or Organizer trends across releases;
* thermal behavior under sustained usage;
* Low Power Mode or thermal adaptation behavior when relevant.

Use the same scenario, device class, OS version, build configuration, and data set for before/after comparison whenever possible.

Also validate correctness:

* cache freshness;
* offline behavior;
* retry behavior;
* request cancellation semantics;
* data durability;
* event delivery;
* background task completion or expiration behavior;
* sensor/location accuracy required by the feature;
* user-visible loading, failure, and fallback states.

## Output Notes

When using this reference, include:

1. Whether the suspected cost is network-bound, disk-bound, power-bound, or mixed.
2. The primary tool or signal to inspect.
3. The strongest evidence needed.
4. The likely app-specific cause.
5. A focused fix direction.
6. Correctness risks or trade-offs.
7. A validation method.

Do not end with generic advice like “profile it.” Name the tool, the scenario, and the signal that would confirm or reject the hypothesis.

Use this output shape when helpful:

```markdown id="kw2o0z"
## Cost classification

Network-bound / disk-bound / power-bound / mixed / unknown.

## Primary evidence

Network instrument / URLSessionTaskMetrics / File Activity / System Trace / Energy Log / Power Profiler / MetricKit / Organizer / signposts / logs.

## Strongest signal to look for

The request waterfall, duplicated requests, cache misses, disk operation, write amplification, wakeup pattern, background task, sensor activity, or thermal trend that would prove the hypothesis.

## Likely cause

The app-specific operation that appears to own the cost.

## Fix direction

One focused change tied to the measured cause.

## Correctness risks

Cache freshness, data durability, routing, retry behavior, offline behavior, cancellation, background completion, event delivery, sensor accuracy, or loading/fallback behavior.

## Validation

How to compare before/after under the same scenario.
```
