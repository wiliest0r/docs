# Level 2: Backend Spin WASM Processing Engine (`beacon`)

This document details the architecture and request-processing pipeline of the **Beacon** HTTP ingestion engine written in Rust and executed within the **Spin WebAssembly (WASM/WASI)** runtime.

---

## 1. Request Ingestion & Processing Pipeline

```mermaid
flowchart TD
    %% =========================================================================
    %% INCOMING REQUEST
    %% =========================================================================
    Req["Incoming HTTP Request\n(Via Ingress / Cloud Run / Service)"] --> Router{"Path & Method Router"}

    %% =========================================================================
    %% HEALTH CHECK
    %% =========================================================================
    Router -->|GET /healthz| Health["Health & Liveness Probe\n- Verify runtime status\n- Check memory allocations"]
    Health --> ResHealth["HTTP 200 OK\n{\"status\": \"healthy\"}"]

    %% =========================================================================
    %% DYNAMIC WEB TAG DELIVERY
    %% =========================================================================
    Router -->|GET /client.js| TagServing["In-Memory Web Tag Serving"]
    subgraph DynamicTag["Dynamic Token Injection"]
        FetchTemplate["Load Embedded JS Tag\n(Cached in WASM linear memory)"]
        GenToken["Generate Ephemeral Session Token\n(HMAC-SHA256 signature + expiry)"]
        InjectToken["Interpolate Token & Origin into Tag Template"]
        SetTagHeaders["Set Response Headers\n- Content-Type: application/javascript\n- Cache-Control: private, max-age=60"]
        FetchTemplate --> GenToken --> InjectToken --> SetTagHeaders
    end
    TagServing --> DynamicTag --> ResTag["HTTP 200 OK\n(Deliver Personalized Tag)"]

    %% =========================================================================
    %% CORS PREFLIGHT
    %% =========================================================================
    Router -->|OPTIONS /v1/sync| CORS["CORS Pre-Flight Handler\n- Access-Control-Allow-Origin: *\n- Access-Control-Allow-Methods: POST, OPTIONS\n- Access-Control-Allow-Headers: Content-Type, X-Beacon-Token"]
    CORS --> ResCORS["HTTP 204 No Content"]

    %% =========================================================================
    %% TELEMETRY INGESTION PIPELINE
    %% =========================================================================
    Router -->|POST /v1/sync| IngestPipeline["Telemetry Ingestion Pipeline"]
    
    subgraph SecurityPartition["1. Anti-Spam & Token Validation"]
        CheckToken{"Validate X-Beacon-Token\n(Signature valid & unexpired?)"}
        RateLimit{"Rate Limiting & Partition\n(Client IP Hash / Token bucket)"}
        DropSpam["HTTP 403 Forbidden / 429 Too Many Requests\n(Discard Malicious Payload)"]
        CheckToken -->|Invalid| DropSpam
        CheckToken -->|Valid| RateLimit
        RateLimit -->|Exceeded| DropSpam
    end

    subgraph DataNormalization["2. Decompression & Normalization"]
        CheckEncoding{"Content-Encoding\nGzip / Deflate?"}
        Decompress["Decompress Body Stream"]
        ParseJSON["Parse & Validate JSON Schema\n(Events, UserProps, DeviceContext)"]
        EnrichMeta["Enrich Metadata\n(Ingest Timestamp UTC, Geo-IP, User-Agent)"]
        
        RateLimit -->|Pass| CheckEncoding
        CheckEncoding -->|Compressed| Decompress --> ParseJSON
        CheckEncoding -->|Uncompressed| ParseJSON
        ParseJSON --> EnrichMeta
    end

    subgraph OutputFormatting["3. Telemetry Stream Output"]
        FormatParquet["Format Normalized JSON Record\n(Parquet / BigQuery schema compliant)"]
        StdoutLog["Write to stdout Stream\n(Zero local file state, instant flush)"]
        FormatParquet --> StdoutLog
    end

    IngestPipeline --> SecurityPartition
    EnrichMeta --> OutputFormatting
    StdoutLog --> ResIngest["HTTP 204 No Content\n(Zero-Body Immediate Response)"]

    %% =========================================================================
    %% CATCH-ALL
    %% =========================================================================
    Router -->|Other Paths| NotFound["HTTP 404 Not Found"]
```

---

## 2. Key Technical Specifications

### Memory Footprint & Concurrency
* **Stateless Model**: Every incoming HTTP request is processed independently in a WebAssembly sandbox instance, guaranteeing complete memory isolation between requests.
* **Instant Start (< 1 ms)**: Because Spin does not boot a Linux kernel or container filesystem, execution begins in sub-millisecond time.
* **Peak Memory Usage**: `< 16 MB` per worker instance.

### Security Guarantees
* **Ephemeral Handshake**: Telemetry can only be ingested if accompanied by a cryptographically signed token generated during the `/client.js` retrieval step, neutralizing headless spam crawlers.
* **CORS Compliance**: Permissive preflight handling allows cross-origin tracking across authorized tenant domains.
* **Log-Oriented Ingestion**: All normalized records are flushed to `stdout`. Persistence, buffering, and loading into analytics sinks (e.g. BigQuery, Google Cloud Storage) are handled entirely out-of-band by the logging infrastructure.
