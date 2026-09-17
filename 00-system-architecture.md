# Level 0: Master System Architecture

This document outlines the overarching architectural blueprint of the **Beacon Analytics & High-Performance Event Tracking Engine**, detailing the data plane (telemetry ingestion and delivery) and the control plane (GitOps, CI/CD, and HCP Terraform Cloud automation).

---

## 1. Master System Flowchart

```mermaid
flowchart TD
    %% =========================================================================
    %% CLIENT TIER
    %% =========================================================================
    subgraph ClientTier["Level 1: Client & Browser Tier"]
        Browser["User Browser\n(Web Application)"]
        WebTag["Beacon JavaScript Tag\n(< 5 KB, IIFE, Zero-Blocking)"]
        Browser -->|1. Loads snippet| WebTag
        WebTag -->|2. GET /client.js| Ingress
        WebTag -->|4. Ingest Event POST /v1/sync\n(navigator.sendBeacon / keepalive)| Ingress
    end

    %% =========================================================================
    %% INGRESS & RUNTIME TIER
    %% =========================================================================
    subgraph IngressTier["Level 2 & 3: Ingress & Compute Edge"]
        Ingress{"Ingress Gateway\n(Traefik / GKE L4 LB / Cloud Run Ingress)"}
        
        subgraph EdgeK8s["Option A: Edge / K8s Compute (k3s / GKE)"]
            K8sService["K8s Service: beacon-service\n(:80 / NodePort 30080)"]
            K8sPod["Spin WASM Pod\n(RuntimeClass: wasm-spin)"]
            K8sShim["containerd-shim-spin-v2\n(Direct WASI Execution)"]
            K8sService --> K8sPod --> K8sShim
        end

        subgraph ServerlessCloud["Option B: Serverless Cloud Run (europe-west1)"]
            CR_Dev["beacon-server-dev\n(SA: sa-beacon-runtime-dev)"]
            CR_Stage["beacon-server-stage\n(SA: sa-beacon-runtime-stage)"]
            CR_Prod["beacon-server-prod\n(SA: sa-beacon-runtime-prod)"]
        end

        Ingress -->|Route /client.js & /v1/sync| K8sService
        Ingress -->|Managed HTTPS Ingress| ServerlessCloud
    end

    %% =========================================================================
    %% TELEMETRY SINKS
    %% =========================================================================
    subgraph TelemetryTier["Telemetry & Storage Pipeline"]
        StdoutStream["Structured JSON to stdout\n(Zero local state)"]
        FluentBit["Log Aggregator / Cloud Logging Agent"]
        CloudLogging["Google Cloud Logging"]
        BigQuery["Analytics Sink / BigQuery / Cloud Storage"]

        K8sShim -->|stdout| StdoutStream
        ServerlessCloud -->|stdout| StdoutStream
        StdoutStream --> FluentBit --> CloudLogging --> BigQuery
    end

    %% =========================================================================
    %% CONTROL PLANE & GITOPS
    %% =========================================================================
    subgraph ControlPlane["Levels 5 & 6: GitOps, CI/CD & Zero-Trust Control Plane"]
        GitRepos["GitHub Repositories\n(web-tag, beacon, deploy,\nterraform-central, gh-actions-central)"]
        GHA["GitHub Actions CI/CD\n(Gitleaks + Trivy + Cargo/WASM)"]
        TFC["HCP Terraform Cloud\n(Workspaces: dev, stage, prod)"]
        WIF["GCP Workload Identity Federation\n(Pool: tfc-pool)"]
        GAR["Google Artifact Registry\n(beacon-repo-dev / stage / prod)"]

        GitRepos -->|PR / Push| GHA
        GHA -->|Trigger Runs| TFC
        GHA -->|OIDC Auth (github-provider)| WIF
        TFC -->|OIDC Auth (tfc-provider)| WIF
        WIF -->|Push Image| GAR
        WIF -->|Deploy Infrastructure| ServerlessCloud
        WIF -->|Provision Cluster & Manifests| EdgeK8s
    end
```

---

## 2. Key Architectural Tenets

### 1. Sub-Millisecond Cold Starts via WebAssembly
* By compiling the ingestion server in Rust targeting the **Spin / WASI runtime**, process startup times drop below `1 ms`, eliminating container cold starts common in serverless platforms.
* Execution happens either inside lightweight OCI containers (Cloud Run) or natively via `containerd-shim-spin-v2` (`RuntimeClass/wasm-spin` in k3s and GKE).

### 2. Zero-Blocking Client Footprint
* The telemetry tracker is packaged as an ultra-compact (`< 5 KB` gzipped) JavaScript bundle.
* Uses non-blocking scheduling (`requestIdleCallback`, asynchronous script loading, `navigator.sendBeacon`, and `fetch` with `keepalive: true`), ensuring zero layout shift or main thread performance degradation.

### 3. Ephemeral Handshake & Stateless Security
* Dynamic tag delivery at `/client.js` injects an ephemeral token valid for short duration sessions.
* Telemetry payloads at `/v1/sync` are validated against this signature, discarding bot traffic and replay spam before emitting structured telemetry to stdout.

### 4. Zero-Trust Identity & Access Boundary
* **No static GCP service account keys** are stored in GitHub Secrets or HCP Terraform.
* Authentication occurs strictly through Workload Identity Federation (WIF) with scoped attribute mapping.
* Cloud Run and GKE worker nodes execute under dedicated, least-privilege service accounts with write permissions restricted solely to standard logging streams (`roles/logging.logWriter`).
