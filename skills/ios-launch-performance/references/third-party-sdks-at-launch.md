# Third-Party SDKs at Launch

Use this reference when a launch investigation involves vendor SDKs or app-owned wrappers around vendor SDKs: analytics, crash reporting, ads, attribution, remote config, feature flags, experimentation, push, consent, security, fraud prevention, logging, monitoring, diagnostics, or other services started during app startup.

Keep this file focused on **SDK startup strategy**: which parts must run on the launch path, which parts can be reduced or deferred, and what correctness risks must be preserved when changing startup order.

## Scope Boundary

This file covers:

* whether a vendor SDK must start before first frame, before first interaction, or later
* whether SDK startup can be split into lightweight setup and deferred work
* SDK startup from `AppDelegate`, `SceneDelegate`, SwiftUI app entry points, dependency containers, bootstrap coordinators, and app-owned facades
* analytics, crash reporting, ads, attribution, remote config, feature flags, experimentation, push, consent, security, fraud, logging, monitoring, diagnostics, and observability SDKs
* launch-specific correctness risks caused by delaying or reordering a vendor SDK
* vendor-supported lazy, deferred, offline, cached, minimal, manual-start, or background modes
* hidden SDK startup through app-owned wrappers, dependency graph construction, root view/model creation, or automatic vendor behavior
* how to review SDK initialization without guessing vendor semantics

This file does not cover:

* dyld loading, Objective-C `+load`, constructors, runtime registration, or static initializer mechanics; use `pre-main-dyld-and-static-initializers.md`
* static frameworks, dynamic frameworks, mergeable libraries, package manager layout, or binary inspection; use `linking-strategy.md`
* general `UIApplicationDelegate`, `UISceneDelegate`, root window, first-screen routing, or main-thread launch work; use `appdelegate-scenedelegate-and-first-frame.md`
* SwiftUI `@main App`, root view, observable state, `.task`, or `.onAppear` behavior; use `swiftui-app-launch.md`
* launch taxonomy, cold/warm/prewarmed/resume classification, or first-frame target interpretation; use `launch-taxonomy-and-targets.md`
* Instruments, XCTest, MetricKit, Organizer, signpost design, or CI baseline setup; use `metrics-instruments-xctest-metrickit.md`
* general privacy, legal, compliance, or security architecture review unless the decision changes launch-time SDK startup

## Core Model

Every SDK initialized on the launch path must answer one question:

```text
What user-visible, diagnostic, security, routing, compliance, or correctness problem occurs if this SDK starts after first frame or after first interaction?
```

Avoid both extremes:

* Do not assume every SDK can be delayed safely.
* Do not assume a vendor recommendation to initialize during app launch means every part of that SDK must run synchronously before the first frame.

Many SDK integrations can be split into phases:

```text
minimal local configuration
→ critical handler/delegate installation
→ cached/default state exposure
→ first visible UI
→ network/session/upload/preload work
→ feature-specific module startup on first use
```

The review goal is to keep only the truly launch-critical portion on the startup path and move the rest to explicit, measurable readiness points.

## Agent Inspection Procedure

When repository access is available, inspect real initialization sites instead of giving generic advice.

### 1. Find startup and bootstrap code

Search for app entry points, startup coordinators, and dependency setup:

```sh
rg "didFinishLaunching|willFinishLaunching|configurationForConnecting|willConnectTo|@main|AppDelegate|SceneDelegate|Bootstrap|Startup|AppInitializer|Launch|DependencyContainer|ServiceLocator|Assembler|CompositionRoot" .
```

### 2. Find SDK categories and wrappers

Search for common vendor categories, product names, and app-owned abstractions:

```sh
rg "Analytics|Crash|Crashlytics|Sentry|Bugsnag|Firebase|Amplitude|Mixpanel|AppsFlyer|Adjust|Branch|AdMob|GoogleMobileAds|RemoteConfig|FeatureFlag|Experiment|ABTest|Push|Notification|OneSignal|Security|Fraud|Jailbreak|Attestation|Monitoring|Logger|Telemetry|Consent|Tracking|SDKManager|ThirdParty" .
```

### 3. Inspect dependency manifests

Check dependency manifests when they exist:

* `Package.swift`
* `Package.resolved`
* `Podfile`
* `Podfile.lock`
* `Cartfile`
* `Cartfile.resolved`
* Tuist, Bazel, Buck, or custom project generator manifests
* vendored `.framework` and `.xcframework` directories
* app target build settings and scripts that may start vendor setup indirectly

Do not turn this into a linking review. If the issue is package layout, dynamic framework count, binary size, or pre-main loading, switch to `linking-strategy.md` or `pre-main-dyld-and-static-initializers.md`.

### 4. Inspect app-owned wrappers first

Many codebases hide vendor startup behind app-owned names such as:

* `AnalyticsService`
* `CrashReporter`
* `RemoteConfigProvider`
* `FeatureFlagClient`
* `PushNotificationService`
* `AttributionService`
* `SecurityProvider`
* `FraudDetector`
* `MonitoringClient`
* `ConsentManager`
* `SDKManager`
* `ThirdPartyServices`
* `AppBootstrapper`

Review what these wrappers actually do during construction and startup. A harmless-looking wrapper may synchronously initialize several vendors, read large local stores, access keychain, start network calls, register delegates, or create background queues.

### 5. Check for hidden startup triggers

SDK work may start before the obvious `start()` or `configure()` call.

Look for:

* eager singletons
* globals or static properties touched during launch
* dependency container registration that creates instances eagerly
* property wrappers or lazy values accessed by the root screen
* SwiftUI root view/model initializers that touch SDK services
* automatic collection/session tracking enabled by default
* Info.plist-driven or build-setting-driven vendor behavior
* Objective-C categories, `+load`, constructor functions, or other load-time hooks

If startup occurs through load-time hooks or static initialization, route that part of the investigation to `pre-main-dyld-and-static-initializers.md`.

## SDK Startup Classification

Classify each SDK or SDK wrapper before recommending deferral.

### Launch-critical and synchronous

Use this classification only when delaying initialization can break correctness, security, diagnostic coverage, launch routing, or mandatory compliance behavior.

Possible examples:

* crash handler installation needed to capture startup crashes
* security, fraud, or integrity gate required before showing sensitive content
* deep link or notification router needed to choose the initial screen
* local feature flag or kill-switch state required to choose the launch surface
* notification categories or delegates required for launch-from-notification behavior
* consent state required before any tracking SDK starts
* mandatory regulatory or compliance gate that must precede visible content or tracking

Even when this classification is justified, review whether the synchronous portion can be made smaller.

### Launch-critical but reducible

Use this when the SDK must participate in launch but does not need its full workload immediately.

Typical strategy:

* install handlers or delegates early
* configure keys and local options early
* read only small cached state synchronously when required
* expose safe defaults immediately
* postpone uploads, session sync, network fetches, device scans, preloads, large file reads, and persistence maintenance

### First-interaction required

Use this when the SDK is needed before the first meaningful user action, but not before the first visible UI.

Examples:

* analytics event pipeline before the first tappable screen event
* experiment assignment before a specific interaction
* fraud or risk check before login, payment, transfer, or checkout
* push permission flow before a notification onboarding step
* monitoring required before an expensive user-initiated flow begins

### Post-first-frame acceptable

Use this when the SDK improves observability, personalization, monetization, diagnostics, or background readiness but is not needed for first frame.

Examples:

* analytics session upload
* remote config refresh when cached/default values exist
* ad SDK initialization when the first screen has no ad slot
* attribution sync when immediate routing does not depend on network attribution
* noncritical monitoring setup
* secondary logging sinks
* previous log or crash upload

### Feature-specific and lazy

Use this when the SDK is only needed after entering a specific feature.

Examples:

* ad mediation for an ad-supported screen
* chat/support SDK for the help center
* payment risk SDK for checkout or transfer
* map/search/location vendor for a later tab
* social login SDK for a specific login method
* video, audio, scanning, or document-processing SDK for a later flow

### Background-only or maintenance

Use this when the work should not affect first frame or early interaction.

Examples:

* cache pruning
* log upload
* pending analytics flush
* remote config refresh that does not affect launch UI
* ad preloading for later screens
* token sync that can retry later
* SDK health checks
* old attachment cleanup

## Category Guidance

### Analytics and event pipelines

Analytics usually does not need full synchronous startup before first frame.

Prefer:

* cheap local event buffering through an app-owned facade
* cached user/session identity when already available
* nonblocking session start
* upload/flush after visible UI or later
* consent gating before tracking when required
* no synchronous network call for user properties, experiments, attribution, or device metadata during launch
* one early app-owned event buffer instead of several vendor SDKs starting independently

Keep synchronous launch work only when the app must record a launch event before any possible crash or termination and the local write is cheap.

Red flags:

* analytics initialization performs network I/O before first frame
* analytics reads large local stores synchronously
* analytics blocks on consent, user profile, remote config, attribution, or experiment assignment
* multiple analytics SDKs start separately instead of using a small app-owned buffer
* event tracking from root view/model initialization forces vendor startup

### Crash reporting and diagnostics

Crash reporting can have a valid reason to install handlers early. That does not mean all crash SDK work must block launch.

Prefer:

* minimal handler installation early
* deferred crash report upload
* deferred symbol, attachment, breadcrumb, and log enrichment
* bounded file reads for previous crash state
* no synchronous network work during launch
* no large log directory scan before first frame
* one clear owner for crash and diagnostic setup

Be careful before deferring the handler itself. If startup crashes matter, delaying handler installation may reduce diagnostic coverage.

Red flags:

* uploading previous crash reports synchronously during startup
* scanning large log directories before first frame
* attaching large device/app state snapshots immediately
* starting multiple overlapping crash, logging, and monitoring SDKs without a clear ownership model
* crash reporting setup hidden behind a broad `initializeAllSDKs` step

### Ads and monetization SDKs

Ad SDKs are usually poor candidates for launch-critical synchronous initialization unless the first visible screen immediately contains an ad and the product explicitly accepts that trade-off.

Prefer:

* initialize after first visible UI when the first screen has no ad slot
* initialize on first ad surface entry when possible
* preload ads only after interaction-critical launch work is complete
* respect consent and tracking authorization state before ad requests
* keep mediation and network adapter startup out of the critical path
* avoid starting all adapters when only one later surface needs ads

Red flags:

* initializing ad mediation in launch lifecycle code when no ad appears on the first screen
* loading or preloading ads before root UI is visible
* blocking first frame on consent, tracking, or ad configuration fetch
* initializing many ad network adapters unconditionally
* starting ad SDKs before consent or tracking state is known

### Attribution, install tracking, and deep linking

Attribution SDKs often combine routing and measurement. Do not treat them as one indivisible launch task.

Prefer:

* parse launch URL, universal link, or notification payload early when it determines the first screen
* route with local payload data when possible
* define fallback routing when network attribution is delayed or unavailable
* defer attribution upload or campaign sync when it does not change immediate routing
* use cached campaign/install state when safe
* time-bound any attribution operation that can affect first-screen routing

Red flags:

* blocking root UI until a network attribution response returns
* starting attribution only to upload install/session data before first frame
* mixing deep-link routing with analytics upload in one synchronous startup call
* no fallback when attribution is slow, offline, or unavailable
* losing universal link or notification routing because the entire SDK was deferred

### Remote config

Remote config should not make launch depend on the network unless the app has a strict correctness requirement.

Prefer:

* safe local defaults
* last-known-good cached config
* an explicit minimal config subset needed for launch
* bounded local reads
* asynchronous refresh after first frame
* staged activation of new config after the app is ready
* separate local config read from remote refresh

Launch-critical remote config should be rare and justified. If a value is required before first frame, prefer shipping a default in the app bundle or storing the last-known-good value locally.

Red flags:

* waiting for remote config fetch before creating root UI
* decoding a large config file synchronously on the main thread
* coupling remote config refresh to dependency container construction
* failing closed in a way that blocks launch for noncritical features
* refreshing config only to update features that are not visible on the first screen

### Feature flags and experimentation

Feature flags can be launch-critical when they choose the initial app surface, enable a safety kill switch, or control a high-risk path. Most flag refresh work is not launch-critical.

Prefer:

* local defaults compiled into the app
* cached flag snapshots loaded cheaply
* deterministic fallback behavior
* separating read-only launch flag access from remote refresh
* refreshing and activating new values after first frame or at a controlled readiness point
* avoiding network waits on the launch path
* keeping experiment assignment off the critical path unless it controls the first visible screen

Red flags:

* remote flag fetch before root UI
* JSON parsing of a large flag payload on the main thread
* flag client initialization that transitively starts analytics, attribution, and remote config
* hidden feature-flag reads that force SDK initialization during root view/model creation
* no documented fallback when the flag service is unavailable

### Push notifications

Push support can have legitimate launch-time configuration requirements. Separate lightweight notification configuration from vendor network startup.

Prefer:

* register notification categories and delegates at launch when required for actionable notifications or launch-time notification routing
* parse notification launch payload early if it decides the initial route
* defer token upload, vendor sync, topic subscription refresh, and marketing automation startup when not required for first frame
* avoid prompting for notification permission during generic app launch unless the onboarding flow intentionally does so
* preserve correct behavior for launches caused by tapping a notification

Red flags:

* starting a full push marketing SDK synchronously only to upload a token
* blocking first frame on token registration or vendor subscription sync
* losing notification launch routing because push setup was deferred without a fallback
* requesting notification permission before the UI explains the value
* coupling push setup with analytics, attribution, or marketing SDK startup unnecessarily

### Security, fraud, attestation, and integrity SDKs

Security-related SDKs require more caution than analytics or ads. Some apps must gate sensitive content, payments, transfers, login, or regulated flows before allowing interaction.

Prefer:

* minimal synchronous checks only for gates truly required before showing the first screen
* cached or locally computed risk state when acceptable
* asynchronous risk enrichment after first frame
* feature-level checks before sensitive actions
* explicit timeout and fallback behavior for network-based checks
* clear product/security decision about fail-open vs fail-closed behavior
* keeping non-sensitive launch UI independent from full risk enrichment when acceptable

Do not defer security SDKs blindly. In high-risk apps, a small amount of launch-time security work may be correct. The performance review should narrow and bound the work, not remove the control.

Red flags:

* network attestation blocking launch with no timeout
* expensive device scan before showing any non-sensitive UI
* security SDK initialized multiple times through different wrappers
* unclear fallback when security service is slow or unavailable
* large keychain or file-system scans on the main thread
* security checks hidden inside unrelated startup services

### Logging, monitoring, and observability

Observability is important, but launch should not be blocked by heavy logging infrastructure.

Prefer:

* tiny in-memory or local buffered logging during early launch
* deferred remote transport setup
* deferred upload of previous logs
* sampling where appropriate
* avoiding synchronous persistence on the main thread
* sharing environment/device metadata collection across observability tools when possible

Red flags:

* opening a large logging database before first frame
* compressing or uploading logs during launch
* initializing multiple monitoring SDKs with duplicate device/environment collection
* blocking launch on logger flush or remote transport readiness
* synchronous breadcrumb enrichment that scans large app state

### Consent and privacy SDKs

Consent state may be required before analytics, ads, attribution, or tracking starts. That does not always mean a full consent SDK must block first frame.

Prefer:

* cheap cached consent state for startup decisions
* no tracking before consent when consent is required
* showing consent UI as part of an explicit onboarding or privacy flow
* deferred vendor startup until consent is known
* separating consent read from vendor initialization
* a clear ordering contract between consent, analytics, ads, and attribution

Red flags:

* starting tracking SDKs before consent is resolved
* blocking launch on a network consent refresh when cached state is available
* coupling consent SDK startup to unrelated SDK initialization
* inconsistent ordering between consent, analytics, attribution, and ads
* no fallback for missing or expired consent state

## Vendor Initialization Strategy

Use this procedure when reviewing a vendor SDK on the launch path.

### 1. Identify the entry point

Find where the SDK is started and who calls it.

Look for:

* direct calls from app/scene lifecycle methods
* calls from a bootstrapper or dependency container
* eager singleton construction
* property wrappers or lazy globals touched during root UI creation
* SwiftUI root model initialization that starts services
* Objective-C categories or runtime hooks that register work implicitly
* automatic vendor behavior enabled by configuration files or build settings

If the SDK starts through `+load`, constructor functions, or static initializers, switch to `pre-main-dyld-and-static-initializers.md` for that part of the investigation.

### 2. Split configuration from work

For each SDK, separate:

* local configuration
* delegate/handler installation
* cached state read
* network request
* upload/flush
* device scan
* database open or migration
* keychain access
* permission prompt
* feature module preload
* background maintenance

Only keep the required subset before first frame.

### 3. Check vendor-supported modes

Look for official support for:

* delayed initialization
* manual start
* offline mode
* cached configuration
* disable automatic session tracking
* disable automatic event collection
* disable automatic network upload during startup
* lazy module loading
* background upload
* separate handler installation and transport startup
* no-op or test mode for CI

Do not invent unsupported initialization sequences for security, crash, payment, attribution, or compliance-sensitive SDKs without checking vendor documentation.

### 4. Define the readiness point

Replace vague delay rules with explicit readiness points.

Good readiness points include:

* after first visible UI is committed, verified by measurement
* after first interaction-critical work is complete
* after authentication state is known
* after consent state is available
* when a feature screen is entered
* when network becomes available
* when the app becomes active and has enough idle time for background startup

Avoid arbitrary fixed delays as the only mechanism. A delay may hide the cost in one trace but still cause jank during early interaction.

### 5. Preserve correctness contracts

Before deferring an SDK, check whether it affects:

* startup crash capture
* notification or deep-link routing
* security gating
* consent and tracking order
* compliance behavior
* kill switches
* initial screen selection
* payment, login, transfer, or regulated flows
* data integrity or migration safety

If correctness depends on the SDK, reduce the launch-time work rather than blindly moving it later.

### 6. Avoid the post-first-frame stampede

Moving all SDKs after the first frame can create a new problem: CPU, I/O, network, lock, and main-thread contention immediately after the UI appears.

Prefer:

* staggered readiness points based on actual need
* bounded concurrency
* background QoS for maintenance work
* cancellation or retry policies for noncritical startup
* avoiding simultaneous large file reads, uploads, device scans, and JSON decoding
* validating first interaction latency after deferral

If deferral improves first draw but causes early interaction jank, the launch issue has been moved rather than solved.

### 7. Add measurement around app-owned wrappers

When the app owns a wrapper such as `SDKManager.start()`, add app-level timing around each vendor startup step rather than measuring only the combined wrapper.

Prefer measuring:

* time spent before first frame
* main-thread time
* synchronous file/keychain/database time
* network requests started during launch
* first interaction latency after deferral
* whether deferred startup causes hitches shortly after launch
* whether SDK work repeats across multiple wrappers

Detailed tool setup belongs in `metrics-instruments-xctest-metrickit.md`.

## Decision Rules

* Do not use blanket rules such as “defer all SDKs” or “initialize all SDKs in launch lifecycle code.” Classify each SDK by launch correctness and user impact.
* Keep the minimal launch-critical part early; move network, upload, preload, scan, refresh, and maintenance work later when safe.
* Prefer cached/default state over launch-time network dependency.
* Prefer app-owned facades that can buffer events, expose cached state, and initialize vendors lazily.
* Do not block first frame on analytics, ads, attribution upload, log upload, remote config refresh, or noncritical experiment refresh unless the product explicitly accepts the trade-off.
* Do not defer crash handler installation, notification routing, security gates, consent ordering, or first-screen feature flags without understanding the correctness impact.
* Treat vendor SDK documentation as the source of truth for supported initialization modes.
* Treat SDK initialization as launch-path work until measurement proves it does not affect first frame or early responsiveness.
* Avoid starting SDKs from global/static initializers, Objective-C `+load`, constructors, dependency container construction, or root view/model initializers unless there is a strong reason.
* If an SDK must start early, make that path small, deterministic, bounded, and visible in measurement.
* Do not confuse “moved off the main thread” with “removed from launch impact.” Background work can still contend for CPU, I/O, locks, memory, and network.
* When several SDKs depend on the same state, load that state once through an app-owned layer instead of letting every vendor scan or fetch it independently.

## Red Flags

Flag these during review:

* one broad `initializeAllSDKs()` step called synchronously during launch with no per-SDK classification
* vendor initialization hidden inside dependency container construction
* SDK startup that performs synchronous network, keychain, file, database, or JSON parsing on the main thread
* crash, analytics, logs, monitoring, and attribution all starting separate device/environment scans
* ad mediation initialized before any ad surface exists
* remote config or feature flag fetch blocking root UI
* push token upload blocking first frame
* notification or universal-link routing broken by deferring an SDK without another routing path
* security SDK network check blocking launch without timeout or fallback
* consent SDK and tracking SDKs initialized in the wrong order
* SDK wrappers started from SwiftUI root view/model initialization
* launch performance claims based only on “moved to async” without trace evidence
* a post-first-frame burst where all deferred SDKs start at once and cause first interaction jank
* vendor startup repeated through multiple wrappers or feature modules
* no documented owner for launch-critical SDK ordering

## Recommended Review Output

When this reference is used, return SDK-specific findings instead of generic launch advice.

Use this structure:

```markdown
### SDK launch classification

- SDK or wrapper:
- Current startup point:
- Required before first frame: yes/no/unknown
- Required before first interaction: yes/no/unknown
- Correctness risk if deferred:
- Main launch cost suspicion:

### Recommendation

- Keep early:
- Move later:
- Make lazy:
- Replace with cached/default state:
- Vendor documentation to verify:

### Validation

- Local trace area to inspect:
- App-level signpost or timing to add:
- Production metric or regression check:
- Risk to retest after deferral:
```

If the answer depends on unavailable vendor documentation, say so clearly and recommend the safest classification rather than guessing.

## Non-Goals

Do not turn this reference into a generic SDK integration guide. The only question here is how vendor startup affects app launch and early responsiveness.
