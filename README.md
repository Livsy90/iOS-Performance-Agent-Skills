# iOS Performance Agent Skills

<p align="center">
  <img width="280" alt="Image" src="https://github.com/user-attachments/assets/36210daa-63b7-4619-8084-5410ff63bfdd" />
</p>

<p align="center">
<img width="500" alt="Image" src="https://github.com/user-attachments/assets/b35e663a-7eea-4cce-a080-bb388406c962" />
</p>
A collection of AI-agent skills for reviewing, diagnosing, and improving performance in iOS applications.

Each skill lives under the `skills/` directory and can be copied manually into another agent setup if needed.

## What this is

This repository contains practical performance-focused instructions for AI coding agents.

The skills are designed to help an agent reason about iOS performance problems with a more structured mental model: what to inspect, what to avoid assuming, which trade-offs matter, and when measurement is required.

It is a collection of reusable guidance files for AI-assisted code review, refactoring, and investigation.

## Included skills

### `ios-launch-performance`

For analyzing app startup and launch-time regressions.

Covers cold, warm, and prewarmed launch, dyld and pre-main work, static initializers, AppDelegate, SceneDelegate, SwiftUI app entry points, first frame, SDK initialization, launch metrics, and validation with tools such as Instruments, XCTest, and MetricKit.

### `ios-performance-profiling`

For choosing and interpreting performance profiling workflows.

Covers Instruments, traces, XCTest metrics, MetricKit payloads, signposts, memory graphs, animation hitches, hangs, CPU work, allocations, disk I/O, networking, and production performance signals.

### `ios-perceived-performance`

For improving how fast and responsive an app feels to users.

Covers loading states, first feedback, progressive rendering, skeletons, optimistic updates, high-stakes actions, perceived latency, and validation through recordings, traces, and user-visible behavior.

### `swiftui-performance`

For reviewing SwiftUI code composition and update behavior.

Covers identity, state ownership, dependency scope, body cost, list performance, row complexity, bindings, closures, layout, drawing, animations, async lifecycle work, and profiling validation.

### `swift-concurrency-performance`

For reviewing performance and correctness trade-offs in Swift Concurrency.

Covers actor isolation, MainActor usage, task lifecycle, task groups, cancellation, AsyncSequence, continuations, reentrancy, executor behavior, and concurrency-related responsiveness issues.

### `swift-runtime-performance`

For reviewing Swift runtime-level performance costs.

Covers allocations, ARC traffic, stack vs heap behavior, dispatch, existentials, generics, opaque types, copy-on-write, SIL optimization, unsafe Swift, modularization, linking, and launch-time trade-offs.

## Repository structure

```text
skills/
  ios-launch-performance/
    SKILL.md
    references/

  ios-performance-profiling/
    SKILL.md

  ios-perceived-performance/
    SKILL.md
    references/

  swiftui-performance/
    SKILL.md
    references/

  swift-concurrency-performance/
    SKILL.md
    references/

  swift-runtime-performance/
    SKILL.md
    references/
