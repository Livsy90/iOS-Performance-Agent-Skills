# Third-Party SDKs at Launch

Use this reference when a launch investigation involves vendor SDKs or app-owned wrappers around vendor SDKs: analytics, crash reporting, ads, attribution, remote config, feature flags, experimentation, push, consent, security, fraud prevention, logging, monitoring, diagnostics, or other services started during app startup.

Keep this file focused on **SDK startup strategy**: which parts must run on the launch path, which parts can be reduced or deferred, and what correctness risks must be preserved when changing startup order.

## Scope Boundary

This file covers:

* whether a vendor SDK must start before first frame, before first interaction, or later;
* whether SDK startup can be split into lightweight setup and deferred work;
* SDK startup from `AppDelegate`, `SceneDelegate`, SwiftUI app entry points, dependency containers, bootstrap coordinators, and app-owned facades;
* analytics, crash reporting, ads, attribution, remote config, feature flags, experimentation, push, consent, security, fraud, logging, monitoring, diagnostics, and observability SDKs;
* launch-specific correctness risks caused by delaying or reordering a vendor SDK;
* vendor-supported lazy, deferred, offline, cached, minimal, manual-start, or background modes;
* hidden SDK startup caused by app-owned wrappers, dependency graph construction, root model creation, static access, SwiftUI environment ownership, or automatic vendor behavior.

This file does not cover:

* dyld, `+load`, constructor functions, or static initializer mechanics;
* static, dynamic, or mergeable linking strategy;
* UIKit root window, root view controller, or first-screen routing implementation;
* SwiftUI `@main App`, root view, `.task`, or `.onAppear` behavior except when it starts SDKs;
* launch taxonomy and measurement comparability;
* Instruments, XCTest, MetricKit, Organizer, signpost, or CI setup;
* general privacy, legal, compliance, or security architecture unless it directly affects launch-time SDK startup.

Use this file for SDK startup policy. If the issue becomes binary packaging, dynamic image count, pre-main hooks, app lifecycle structure, SwiftUI lifecycle, launch orchestration, or measurement tooling, route to the focused reference.

## Contents

* [Core Model](#core-model)
* [Review Procedure](#review-procedure)
* [Vendor Documentation Rule](#vendor-documentation-rule)
* [SDK Startup Phases](#sdk-startup-phases)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [Hidden Startup Triggers](#hidden-startup-triggers)
* [SDK Startup Classification](#sdk-startup-classification)
* [Category Guidance](#category-guidance)
* [Analytics and Event Pipelines](#analytics-and-event-pipelines)
* [Crash Reporting and Diagnostics](#crash-reporting-and-diagnostics)
* [Ads and Monetization SDKs](#ads-and-monetization-sdks)
* [Attribution, Install Tracking, and Deep Linking](#attribution-install-tracking-and-deep-linking)
* [Remote Config](#remote-config)
* [Feature Flags and Experimentation](#feature-flags-and-experimentation)
* [Push Notifications](#push-notifications)
* [Consent and Privacy-Gated SDKs](#consent-and-privacy-gated-sdks)
* [Security, Fraud, and Integrity SDKs](#security-fraud-and-integrity-sdks)
* [Logging, Monitoring, and Diagnostics](#logging-monitoring-and-diagnostics)
* [Safe Patch Heuristics](#safe-patch-heuristics)
* [Review Checklist](#review-checklist)
* [Agent Guidance](#agent-guidance)
* [Boundary With Other References](#boundary-with-other-references)

## Core Model

The central question is:

**What user-visible, diagnostic, security, routing, compliance, or correctness problem occurs if this SDK starts after first frame or after first interaction?**

Avoid both extremes:

* Do not assume every SDK can be delayed safely.
* Do not assume a vendor instruction to "initialize at launch" means every part of the SDK must block first frame.

Many SDKs can be split into phases:

```text
minimal local configuration
→ critical handler/delegate installation
→ cached/default state exposure
→ first visible UI
→ network/session/upload/preload work
→ feature-specific module startup on first use
```

The goal is not always to remove the SDK from launch. The goal is to keep only the truly launch-critical portion on the startup path and move the rest behind a clear readiness, feature, or background boundary.

Deferring SDK startup does not mean starting every SDK in an unbounded `Task {}` immediately after first frame. Post-first-frame SDK work can still hurt first interaction through CPU, I/O, locks, networking, memory pressure, or main-actor updates.

## Review Procedure

When using this reference:

1. Find all SDK startup entry points.
2. Inspect app-owned wrappers before vendor calls.
3. Identify hidden startup triggers.
4. Classify each SDK by startup necessity.
5. Split each SDK into minimal critical setup and deferrable work.
6. Check correctness risks: crash coverage, routing, consent, security, fraud, compliance, push, feature flags, diagnostics, and product requirements.
7. Check whether vendor-supported lazy, deferred, manual-start, cached, offline, or minimal modes exist.
8. Check whether post-first-frame SDK work is bounded and does not compete with first interaction.
9. Recommend the smallest safe change.
10. Define validation for first frame, first interaction, SDK correctness, and production diagnostics.

Do not recommend deferral until the SDK's launch responsibility is understood.

## Vendor Documentation Rule

Vendor documentation is a constraint, not the final launch design.

Preserve:

* required initialization order;
* supported startup modes;
* required delegates or handlers;
* privacy and consent requirements;
* crash or security coverage requirements;
* documented restrictions around lazy or manual startup.

Still separate:

* minimal required setup;
* handler/delegate installation;
* cached/default state exposure;
* network refreshes;
* uploads;
* attribution sync;
* device scans;
* preloads;
* session enrichment;
* optional feature modules;
* maintenance work.

If vendor docs do not describe a safe deferred mode, do not invent one. Recommend measuring, isolating app-owned wrapper work, or consulting vendor documentation/support.

## SDK Startup Phases

Separate SDK operations by behavior before recommending changes.

Common phases:

* local configuration;
* API key, environment, release, or build metadata setup;
* delegate or handler registration;
* event buffering;
* enabling or disabling collection;
* reading cached state;
* exposing default or last-known-good values;
* network refresh;
* upload or flush;
* session enrichment;
* device metadata collection;
* feature module preload;
* topic, subscription, or identity sync;
* maintenance cleanup.

Different phases may have different launch deadlines.

Examples:

* a crash SDK may need handler installation early, but previous crash upload can often move later;
* an analytics facade may need to accept early events locally, but vendor network upload can move later;
* a remote config system may need cached/default values early, but refresh can move later;
* a push system may need notification payload preservation early, while token upload or topic sync can move later;
* a security SDK may need a minimal gate early, while diagnostics upload or enrichment can move later.

Do not treat `configure`, `start`, `enable collection`, `identify`, `flush`, `sync`, and `upload` as one indivisible launch task unless vendor constraints require it.

## What the Agent Can Inspect

When repository access is available, inspect app-owned startup surfaces and wrappers before blaming the vendor SDK itself.

Search startup and bootstrap code:

```sh id="tf0byc"
rg "didFinishLaunching|willFinishLaunching|configurationForConnecting|willConnectTo|@main|AppDelegate|SceneDelegate|Bootstrap|Startup|AppInitializer|Launch|DependencyContainer|ServiceLocator|Assembler|CompositionRoot" .
```

Search SDK categories and common wrappers:

```sh id="0et2ep"
rg "Analytics|Crash|Crashlytics|Sentry|Bugsnag|Firebase|Amplitude|Mixpanel|AppsFlyer|Adjust|Branch|AdMob|GoogleMobileAds|RemoteConfig|FeatureFlag|Experiment|ABTest|Push|Notification|OneSignal|Security|Fraud|Jailbreak|Attestation|Monitoring|Logger|Telemetry|Consent|Tracking|SDKManager|ThirdParty" .
```

Search concrete vendor startup calls when relevant:

```sh id="e4p394"
rg "FirebaseApp\.configure|SentrySDK\.start|Bugsnag\.start|GADMobileAds|AppsFlyerLib|Adjust\.appDidLaunch|Branch\.getInstance|OneSignal|Mixpanel\.initialize|Amplitude|RemoteConfig\.remoteConfig" .
```

Search wrapper-style startup methods:

```sh id="sis7n1"
rg "configure\(|start\(|initialize\(|setup\(|register\(|activate\(|enable\(|track\(|identify\(|setUser|setDevice|setDelegate|setDataCollection" .
```

Search hidden eager access:

```sh id="frsne1"
rg "shared|singleton|default|static let|static var|lazy var|@StateObject|@Observable|ObservableObject|\.environment\(|\.task\s*\{|\.onAppear" .
```

Search async deferral and post-launch fan-out:

```sh id="1phfz1"
rg "Task\s*\{|Task\(|Task\.detached|async let|withTaskGroup|await .*configure|await .*start|await .*initialize|await .*sync|await .*upload|await .*flush" .
```

Search expensive work near SDK startup:

```sh id="52y52s"
rg "Data\(|contentsOf:|FileManager|Keychain|SecItem|JSONDecoder|PropertyListDecoder|URLSession|upload|flush|sync|wait\(|semaphore|migrate|scan|preload|warm" .
```

Search manifests and binary SDK ownership:

```sh id="byx5fu"
rg "Firebase|Crashlytics|Sentry|Bugsnag|AppsFlyer|Adjust|Branch|AdMob|GoogleMobileAds|OneSignal|Amplitude|Mixpanel|RemoteConfig|xcframework|vendored_frameworks|binaryTarget|use_frameworks!" .
```

Inspect manifests only to identify SDK ownership and startup surfaces. If the question becomes binary packaging, dynamic image count, static/dynamic linkage, or pre-main hooks, route to `linking-strategy.md` or `pre-main-dyld-and-static-initializers.md`.

Use search results as leads, not proof. Confirm whether the SDK starts before first frame, before first interaction, on resume, after first frame, or only when a feature opens.

The agent can:

* identify SDK startup entry points;
* identify app-owned wrappers that eagerly start vendors;
* classify SDK startup necessity;
* separate minimal required setup from deferrable work;
* detect duplicate SDK startup across lifecycle surfaces;
* recommend idempotency, phase splitting, lazy feature startup, cached/default state, buffering, timeout, or fallback behavior;
* add signposts around app-owned SDK phases;
* ask for vendor documentation or production correctness requirements when needed.

The agent cannot reliably:

* change vendor SDK internals;
* invent unsupported lazy startup modes;
* delay crash/security/fraud/compliance startup without product review;
* decide privacy/legal requirements without product context;
* prove SDK launch cost without measurement;
* assume a vendor sample app represents the app's required startup policy.

## Hidden Startup Triggers

SDK work may begin before an obvious `start()` or `configure()` call.

Check for:

* eager singletons;
* static/global access;
* dependency container registration that creates instances eagerly;
* app-owned facades whose initializer starts vendors;
* SwiftUI root view or root model initialization touching SDK services;
* SwiftUI `@StateObject`, `@Observable`, or environment value construction or first access;
* UIKit lifecycle callbacks that call wrapper setup indirectly;
* automatic session tracking;
* automatic event collection;
* Info.plist or build-setting-driven vendor behavior;
* Objective-C categories, `+load`, constructors, or load-time hooks;
* notification, deep-link, or push delegates installed during launch;
* remote config or feature flag reads that trigger network refreshes;
* consent state reads that initialize tracking vendors.

A SwiftUI `@StateObject`, `@Observable`, or environment value can start SDK work during construction or first access. Preserve ownership and lifetime when moving SDK wrappers out of root state.

If hidden startup is pre-main or load-time behavior, route to `pre-main-dyld-and-static-initializers.md`.

If hidden startup is caused by eager dependency graph construction, route orchestration decisions to `launch-orchestration-and-dependency-graph.md`.

## SDK Startup Classification

Classify each SDK or wrapper before recommending changes.

### Launch-critical and synchronous

Use this classification when delaying the SDK would break a required launch-time property.

Examples may include:

* crash handler installation when startup crash coverage is a product requirement;
* security, fraud, jailbreak, attestation, or integrity checks required before showing sensitive content;
* consent enforcement required before any tracking-capable SDK starts;
* deep-link routing required to show the correct first screen;
* privacy lock, authentication, or compliance gates that must run before visible content.

Even then, look for a smaller launch-critical subset.

### Launch-critical but reducible

Use this classification when the SDK must install a handler, expose cached state, or register a delegate early, but heavier work can move later.

Prefer:

* install handler/delegate early;
* configure keys and local options early;
* read only small cached state synchronously;
* expose safe defaults;
* postpone uploads;
* postpone network refresh;
* postpone session enrichment;
* postpone device scans;
* postpone preloading;
* postpone cleanup and maintenance.

This is the most common target for launch optimization.

### First-interaction required

Use this classification when the SDK does not need to block first frame but must be ready before the user performs the first meaningful action.

Examples:

* feature flag needed before enabling a button;
* fraud check needed before money movement;
* remote config needed before showing a sensitive entry point;
* push registration needed before notification permission flow;
* logging/diagnostics required before a critical action.

The first frame may render a placeholder, disabled control, cached state, or loading state while the SDK becomes ready.

### Post-first-frame acceptable

Use this classification when startup can happen shortly after the first visible UI without breaking correctness.

Examples:

* analytics upload;
* session enrichment;
* user property sync;
* noncritical monitoring setup;
* remote config refresh when cached defaults are available;
* event queue flush;
* diagnostics upload;
* attribution sync that does not affect initial routing.

Make sure post-first-frame work does not immediately block first interaction. Bound concurrency, avoid large synchronous main-actor updates, and avoid launching every deferred SDK at once.

### Feature-specific and lazy

Use this classification when the SDK is only needed by a later feature.

Examples:

* ad mediation used only on ad-supported screens;
* chat/support SDK opened from a support screen;
* payment risk SDK used only during checkout;
* map/search/location SDK used only in a map flow;
* social login SDK used only when user chooses that provider;
* video, audio, scanner, document, or ML SDK used only inside a feature.

Move startup to the feature boundary and provide loading, failure, retry, and cancellation behavior there.

Lazy startup is only a win if the feature can absorb the cost without creating a worse first-use hitch. Consider prefetching at a safe idle or readiness point only when it does not compete with launch or first interaction.

### Background-only or maintenance

Use this classification when the SDK work is useful but not required for first frame or first interaction.

Examples:

* uploading cached logs;
* cleanup;
* diagnostics sync;
* session refresh;
* campaign sync;
* cache pruning;
* periodic health checks;
* optional monitoring enrichment.

Schedule after launch or in an appropriate background path.

## Category Guidance

Use the following category rules as defaults. Product requirements, vendor constraints, legal requirements, and measured evidence can override them.

## Analytics and Event Pipelines

Default stance:

Analytics usually does not need full synchronous startup before first frame.

Prefer:

* cheap local event buffering;
* cached identity;
* nonblocking session start;
* deferred upload;
* deferred user property sync;
* consent-aware startup;
* event queue that can accept early events before network setup is complete;
* small facade that records launch events without starting the full vendor stack.

Red flags:

* synchronous network request during launch;
* flushing old events before first frame;
* resolving user identity through keychain or network synchronously;
* loading large configuration before first screen;
* starting multiple analytics vendors through one wrapper without phase separation;
* broad device metadata collection before first frame.

Do not break:

* consent rules;
* required internal audit events;
* launch attribution events if the product depends on them;
* event ordering when the app requires it.

## Crash Reporting and Diagnostics

Default stance:

Crash handler installation may need to be early. Crash upload, enrichment, and large diagnostics work usually do not.

Prefer:

* install the minimal crash handler early;
* configure release/environment keys early;
* defer upload of previous crash reports;
* defer previous crash processing when vendor and product requirements allow it;
* defer log attachment collection;
* defer session enrichment;
* avoid synchronous disk scans;
* avoid heavy symbol or metadata work on launch;
* add a clear phase boundary between handler installation and upload/enrichment.

Installing the handler for new crashes and uploading or enriching old reports are different responsibilities.

Red flags:

* reading large logs before first frame;
* uploading crash reports synchronously;
* collecting device/app state synchronously;
* initializing multiple diagnostics SDKs through one blocking wrapper;
* blocking launch on previous crash processing.

Do not break:

* startup crash coverage when it is a product requirement;
* privacy and consent constraints for diagnostics collection;
* required crash grouping metadata;
* release/environment configuration correctness.

## Ads and Monetization SDKs

Default stance:

Ads are usually poor candidates for launch-critical synchronous initialization unless the first visible screen contains an ad and the product explicitly accepts that trade-off.

Prefer:

* start ad SDK at the first ad surface;
* preload only when the screen requires it;
* use feature-local loading state;
* keep mediation startup out of the global launch path;
* defer consent-dependent ad startup until consent state is known;
* separate ad SDK configuration from ad loading.

Red flags:

* initializing ad mediation in `didFinishLaunching` for screens that do not show ads;
* loading ads before first frame;
* network preloads at launch;
* blocking first screen on ad readiness;
* starting multiple ad networks through global bootstrap;
* starting tracking-capable ad SDKs before consent state is valid.

Do not break:

* consent and tracking requirements;
* first ad screen loading/fallback behavior;
* monetization reporting required by product decisions;
* vendor-supported initialization order.

## Attribution, Install Tracking, and Deep Linking

Default stance:

Deep-link routing and attribution upload are different responsibilities. Do not treat them as one indivisible launch task.

Prefer:

* parse launch URL, universal link, notification payload, or install payload early enough for routing;
* preserve payload for later attribution sync;
* use cached attribution state when possible;
* use a pending-route model when routing depends on delayed SDK state;
* show a safe root UI while attribution enrichment or deep-link resolution completes when product behavior allows it;
* time-bound routing decisions;
* provide fallback routing when the attribution SDK is slow or unavailable;
* defer campaign upload or enrichment when it does not affect initial route.

If routing waits for SDK resolution, make the wait time-bounded and define fallback routing.

Red flags:

* blocking first frame on attribution network calls;
* delaying initial routing while campaign metadata refreshes;
* losing payload when SDK startup is deferred;
* requiring attribution SDK readiness for every app-icon launch;
* mixing install tracking, deep-link routing, and analytics upload in one blocking setup call.

Do not break:

* initial deep-link routing;
* notification routing;
* install attribution data capture when product requires it;
* payload preservation across deferred startup;
* fraud or campaign validation rules if they gate initial content.

## Remote Config

Default stance:

Remote config should not make launch network-dependent unless the config controls launch safety, compliance, or initial surface correctness.

Prefer:

* local defaults;
* last-known-good cached config;
* minimal launch-critical config subset;
* async refresh after first frame;
* timeout and fallback behavior;
* explicit loading/disabled state for config-gated UI;
* separation between config read and config refresh.

For safety, compliance, kill switches, app-version compatibility, and security-sensitive flags, define whether missing config is fail-open or fail-closed. Do not let timeout behavior accidentally choose a permissive state.

Red flags:

* blocking first frame on remote config fetch;
* decoding large config synchronously at launch;
* refreshing all experiments before first screen;
* failing launch when config fetch fails without a defined fallback;
* using remote config as a global dependency for root UI when only one feature needs it.

Do not break:

* kill switches;
* safety gates;
* compliance gates;
* app-version compatibility checks;
* required initial-surface correctness;
* last-known-good fallback semantics.

## Feature Flags and Experimentation

Default stance:

Feature flags may be required for initial routing or first-screen layout, but full experiment sync usually should not block first frame.

Prefer:

* local default values;
* cached assignment;
* stable bucketing;
* minimal initial flag set;
* async refresh for noncritical flags;
* feature-owned flag loading where possible;
* explicit fallback for unknown flag state.

For safety, compliance, kill switches, and security-sensitive flags, define whether missing state is fail-open or fail-closed. Do not let timeout behavior accidentally choose a permissive state.

Red flags:

* fetching all flags synchronously at launch;
* recomputing all experiments before first frame;
* blocking first screen on noncritical experiment data;
* changing root navigation based on a value that may arrive late without fallback;
* forcing entire root environment to depend on a flag provider refresh.

Do not break:

* assignment stability;
* experiment exposure logging rules;
* kill switches;
* compliance or safety flags;
* routing correctness for initial screen.

## Push Notifications

Default stance:

Push setup is often needed early enough for registration, permission flow, or notification routing, but not all push-related work must block first frame.

Separate:

* notification delegate installation;
* launch notification payload preservation;
* notification routing;
* permission prompt flow;
* APNs registration;
* push SDK registration;
* token upload;
* topic/subscription sync;
* notification analytics.

These responsibilities do not necessarily share the same launch deadline.

Prefer:

* preserve launch notification payload early;
* install notification delegates when needed;
* defer token upload if it is not required for first frame;
* defer subscription/topic sync;
* avoid blocking UI on permission or token refresh unless flow requires it;
* separate notification routing from push service maintenance.

Red flags:

* uploading token synchronously before first frame;
* syncing topics at launch;
* blocking root UI on push registration;
* losing tapped-notification payload while deferring SDK startup;
* registering push SDK globally when push is only used after onboarding.

Do not break:

* tapped notification routing;
* notification payload preservation;
* permission flow correctness;
* required delegate installation;
* token registration if the first flow depends on it.

## Consent and Privacy-Gated SDKs

Default stance:

Consent and privacy state can be launch-critical. Tracking-capable SDK startup must respect consent state.

Model consent as explicit state:

* granted;
* denied;
* unknown.

Do not let `unknown` accidentally start tracking-capable SDKs as if consent were granted.

Prefer:

* cheap cached consent read;
* local consent defaults;
* minimal consent gate before tracking SDKs start;
* deferred vendor startup until consent allows it;
* disabled or buffering behavior documented as compliant by the SDK;
* explicit disabled state for tracking vendors;
* clear separation between consent check and vendor network work.

If consent is unknown at launch, prefer disabled or documented buffering behavior until the app can resolve consent safely.

Red flags:

* starting analytics, ads, attribution, or tracking SDKs before consent state is known;
* treating unknown consent as granted;
* blocking first frame on consent network refresh when cached state is valid;
* performing broad SDK setup before privacy gating;
* hiding vendor startup inside wrappers that bypass consent checks.

Do not break:

* legal/privacy requirements;
* consent persistence semantics;
* user opt-out behavior;
* SDK disabled mode when consent is denied;
* auditability of tracking startup.

## Security, Fraud, and Integrity SDKs

Default stance:

Security, fraud, jailbreak, attestation, and integrity SDKs may be launch-critical when the app must not show sensitive content until checks pass.

Prefer:

* minimal early gate;
* cached risk state only if product/security policy allows it;
* freshness, expiry, and invalidation rules for cached risk state;
* explicit behavior when risk cache is missing or stale;
* timeout and degraded mode policy;
* explicit blocked/loading state;
* defer noncritical scans and uploads;
* feature-specific checks for feature-specific risk;
* clear separation between gate decision and telemetry upload.

If cached risk state is used, define freshness, expiry, invalidation, and what happens when the cache is missing or stale.

Red flags:

* blocking all launches on long network attestation without fallback;
* showing sensitive UI before required checks pass;
* doing broad device scans before first frame when not required;
* mixing security gate, diagnostics upload, and analytics into one startup block;
* starting fraud SDK for users or features that do not need it yet.

Do not break:

* security policy;
* fraud-prevention requirements;
* compliance requirements;
* sensitive-content gating;
* payment or account-risk gates;
* audit and incident-response requirements.

## Logging, Monitoring, and Diagnostics

Default stance:

Minimal logging/monitoring setup can be useful early. Heavy upload, enrichment, or historical processing usually should not block launch.

Prefer:

* lightweight local logger setup;
* in-memory buffering;
* async upload;
* deferred enrichment;
* bounded log file reads;
* small launch-specific diagnostics markers;
* stable signposts around SDK startup phases.

Red flags:

* reading large log files before first frame;
* uploading logs synchronously;
* collecting broad device metadata during launch;
* starting multiple monitoring SDKs through one blocking wrapper;
* blocking first screen on diagnostics readiness.

Do not break:

* minimum diagnostics needed for startup incidents;
* privacy and consent rules;
* required audit logging;
* crash/report correlation if product depends on it.

## Safe Patch Heuristics

When the agent is allowed to edit code, prefer small, reversible changes.

Good patch candidates:

* split `initializeAllSDKs()` into named SDK phases;
* introduce an app-owned facade that can buffer early events locally before starting the vendor network stack;
* keep crash handler installation early but defer upload and enrichment;
* replace direct vendor calls during launch with an app-owned buffer or facade;
* move analytics upload, session enrichment, or user property sync after first visible UI;
* move ad mediation startup to the first ad surface;
* use cached/default remote config for launch and refresh later;
* time-bound attribution or deep-link resolution and provide fallback routing;
* preserve deep-link or notification payload before deferring attribution or push SDK startup;
* move feature-only SDK startup to the feature entry point;
* add idempotency guards to SDK startup wrappers;
* bound post-first-frame SDK work so it does not hurt first interaction;
* add cancellation or timeout behavior to post-launch SDK work;
* add signposts around SDK startup phases;
* separate consent check from vendor network startup.

Risky patch candidates requiring extra care:

* delaying crash handler installation without diagnostic trade-off review;
* delaying consent, security, fraud, payment, privacy, or compliance gates without product/legal/security review;
* deferring deep-link or notification routing without fallback and payload preservation;
* moving SDK startup later when visible UI depends on SDK state but has no loading/failure mode;
* changing vendor-supported startup sequence without documentation;
* hiding SDK startup in async tasks that still compete with first interaction;
* replacing synchronous launch work with unbounded concurrent task fan-out;
* changing SDK startup ownership without idempotency;
* disabling SDK auto-start behavior without checking vendor constraints;
* changing binary packaging or linkage as a substitute for startup policy.

If correctness is uncertain, recommend measurement, product review, vendor documentation, or a smaller phase split before behavior-changing edits.

## Review Checklist

Use this checklist when reviewing third-party SDK startup on the launch path.

* [ ] Are all SDK startup entry points identified?
* [ ] Are app-owned wrappers inspected before blaming the vendor SDK?
* [ ] Are hidden startup triggers identified?
* [ ] Is each SDK classified by launch necessity?
* [ ] Is minimal required setup separated from deferrable work?
* [ ] Are `configure`, `start`, `enable collection`, `identify`, `flush`, `sync`, and `upload` treated as separate responsibilities where possible?
* [ ] Are crash, consent, security, fraud, deep-link, push, feature flag, and remote config correctness risks preserved?
* [ ] Are vendor-supported startup modes checked?
* [ ] Are network calls, uploads, scans, preloads, and enrichment kept off the first-frame path unless required?
* [ ] Is post-first-frame SDK work bounded so it does not hurt first interaction?
* [ ] Are feature-specific SDKs started at feature boundaries?
* [ ] Are duplicate SDK startups avoided across AppDelegate, SceneDelegate, SwiftUI App, root models, `.task`, `.onAppear`, and dependency containers?
* [ ] Are SDK startup wrappers idempotent?
* [ ] Are async deferrals bounded and cancellable where appropriate?
* [ ] Are loading, failure, fallback, and degraded states defined?
* [ ] Is production correctness checked after deferral: crash capture, event delivery, attribution capture, push routing, consent enforcement, and security gate behavior?
* [ ] Are signposts or measurements available to validate improvement?
* [ ] Is the recommendation tied to first-frame, first-interaction, SDK correctness, and production validation?

## Agent Guidance

When applying this reference, produce an SDK startup-oriented review:

```markdown id="ixm4fx"
### SDK startup surface

List where SDK startup occurs: AppDelegate, SceneDelegate, SwiftUI App, root model, dependency container, wrapper, `.task`, `.onAppear`, hidden static access, or vendor auto-start.

### SDK classification

Classify each SDK as launch-critical synchronous, launch-critical but reducible, first-interaction required, post-first-frame acceptable, feature-specific/lazy, or background/maintenance.

### Minimal required setup

Describe the smallest setup that must remain early.

### Deferrable work

List uploads, network refreshes, enrichment, scans, preloads, cleanup, feature modules, or sync work that can move later.

### Correctness risks

Call out crash coverage, consent, privacy, routing, attribution, push, security, fraud, compliance, diagnostics, kill switches, or experiment assignment concerns.

### Unknowns / evidence needed

List missing vendor documentation, product requirements, consent/security constraints, measurement, startup owner, or proof that the SDK actually runs before first frame or first interaction.

### Recommended change

Suggest the smallest safe phase split, lazy startup, facade, idempotency guard, timeout, fallback, or measurement change.

### Validation

Explain how to verify first frame, first interaction, SDK behavior, diagnostics, event delivery, attribution capture, push routing, consent enforcement, security gate behavior, and production correctness after the change.
```

Keep SDK recommendations tied to correctness. Do not recommend deferral only because an SDK is third-party.

## Boundary With Other References

Use this reference for third-party SDK startup policy and app-owned SDK wrapper behavior.

Read `references/launch-taxonomy-and-targets.md` when the issue involves:

* cold, warm, prewarmed, resume, first install, or update launch terminology;
* launch target selection;
* measurement scenario classification;
* whether two numbers are comparable.

Read `references/pre-main-dyld-and-static-initializers.md` when the issue involves:

* SDK behavior from `+load`;
* constructor functions;
* ObjC categories;
* static initializer mechanics;
* load-time hooks before app lifecycle code.

Read `references/linking-strategy.md` when the issue involves:

* vendored SDK packaging;
* dynamic frameworks;
* static libraries;
* mergeable libraries;
* modularization and launch-time linking trade-offs;
* binary layout;
* order-file considerations.

Read `references/launch-orchestration-and-dependency-graph.md` when the issue involves:

* SDK startup as part of a larger startup graph;
* critical path analysis;
* startup step dependencies;
* hidden ordering;
* safe parallelism;
* failure policy;
* dependency-chain optimization.

Read `references/appdelegate-scenedelegate-and-first-frame.md` when the issue involves:

* `UIApplicationDelegate`;
* `UISceneDelegate`;
* lifecycle callback implementation;
* window setup;
* root view controller creation;
* first-frame readiness;
* main-thread lifecycle work.

Read `references/swiftui-app-launch.md` when the issue involves:

* SwiftUI `App`;
* `WindowGroup`;
* root view setup;
* observable state;
* `.task`;
* `.onAppear`;
* `scenePhase`;
* `@UIApplicationDelegateAdaptor`;
* environment initialization.

Read `references/metrics-instruments-xctest-metrickit.md` when the issue involves:

* Instruments;
* Time Profiler;
* signposts;
* XCTest launch metrics;
* MetricKit;
* Xcode Organizer;
* CI baselines;
* production monitoring.

Do not read all references by default.
