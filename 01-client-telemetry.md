# Level 1: Client Web Tag Telemetry Architecture (`web-tag`)

This document specifies the client-side telemetry tracker architecture, detailing the lifecycle of the JavaScript beacon tag from loading to event transmission.

---

## 1. Client Tag Lifecycle & Transport Flowchart

```mermaid
flowchart TD
    subgraph Bootstrap["1. Initialization & Bootstrap"]
        ScriptLoad["Client Browser executes snippet\n<script async src='https://host/client.js' data-project-id='...'>"]
        ConfigParse["Parse Configuration & Options\n(Project ID, Endpoint, Sampling, Debug)"]
        InitQueue["Initialize Memory & Offline Buffers\n(window.beacon global queue)"]
        ScriptLoad --> ConfigParse --> InitQueue
    end

    subgraph Listeners["2. Telemetry Observers & Listeners"]
        SPA["SPA Navigation Listener\n(Patched pushState/replaceState + popstate)"]
        CWV["Core Web Vitals Observer\n(LCP, FID, CLS, INP via PerformanceObserver)"]
        Errors["Error & Diagnostic Observer\n(window.onerror + unhandledrejection)"]
        UserEvents["Custom & Auto-Tracked Events\n(Click, Submit, Scroll depth)"]
    end

    subgraph ConsentGate["3. Consent Management (CMP) Gate"]
        CheckConsent{"Is Tracking Consent Granted?\n(GDPR / CCPA / TCF v2.2)"}
        BufferEvents["Buffer Events in Memory / IndexedDB\n(Wait for consent resolution)"]
        DropEvents["Drop Non-Essential Events\n(If consent rejected)"]
        ProcessEvents["Enrich Event with Session Context\n(Session ID, Ephemeral Token, Viewport, Timestamp)"]
    end

    subgraph QueueManager["4. Batching & Flush Manager"]
        EventQueue["Internal Event Queue"]
        FlushTrigger{"Flush Trigger Condition?\n- Max batch size (10 items)\n- Flush timer (5000 ms)\n- visibilitychange / beforeunload"}
    end

    subgraph TransportEngine["5. Zero-Blocking Transport Engine"]
        CheckBeacon{"navigator.sendBeacon\navailable?"}
        SendBeacon["navigator.sendBeacon('/v1/sync', payload)\n(Guaranteed delivery on page close)"]
        CheckFetch{"fetch with keepalive\navailable?"}
        SendFetch["window.fetch('/v1/sync', { keepalive: true })\n(Modern asynchronous transport)"]
        SendXHR["Synchronous / Asynchronous XMLHttpRequest\n(Legacy browser fallback)"]
    end

    subgraph Endpoint["6. Ingestion Response"]
        Success["HTTP 204 No Content\n(Telemetry successfully accepted)"]
    end

    InitQueue --> SPA & CWV & Errors & UserEvents
    SPA & CWV & Errors & UserEvents --> CheckConsent
    CheckConsent -->|Pending| BufferEvents
    CheckConsent -->|Rejected| DropEvents
    CheckConsent -->|Granted| ProcessEvents
    BufferEvents -->|Consent Granted Later| ProcessEvents
    ProcessEvents --> EventQueue
    EventQueue --> FlushTrigger
    FlushTrigger -->|Condition Met| CheckBeacon
    CheckBeacon -->|Yes| SendBeacon
    CheckBeacon -->|No| CheckFetch
    CheckFetch -->|Yes| SendFetch
    CheckFetch -->|No| SendXHR
    SendBeacon & SendFetch & SendXHR --> Success
```

---

## 2. Technical Specifications

### Budget & Footprint Constraints
* **Total Gzipped Bundle Size**: `< 5 KB`
* **Zero External Dependencies**: Pure Vanilla ES2022+ with automated minification via `esbuild`.
* **Zero Main Thread Jitter**: Performance measurements run inside `requestIdleCallback` or decoupled microtasks.

### Transport Strategy Hierarchy
1. **`navigator.sendBeacon()`**: Primary delivery channel. Does not block document teardown and guarantees transmission during user navigation or tab closure.
2. **`fetch()` with `keepalive: true`**: Secondary modern transport used when payload structure or headers require custom POST configurations.
3. **`XMLHttpRequest`**: Graceful degradation fallback for older browser environments.

### Privacy & Consent Compliance
* Ingestion honors Google Consent Mode v2 and IAB Europe TCF v2.2 signals.
* When consent state is pending, telemetry events are queued in a transient memory buffer. If consent is explicitly denied, non-essential buffers are purged without making network calls.
