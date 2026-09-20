# Publication Series: High-Throughput AdTech Ingestion on Rust WASM, Kubernetes & BigQuery

A complete, production-grade 4-part article series designed for publication on **Medium**, **Dev.to**, and engineering blogs.

---

## Series Table of Contents

| Part | Title | Focus & Core Highlights | Link |
| :---: | :--- | :--- | :---: |
| **01** | **Why We Ditched Node.js for Rust WASM: Building a Sub-Millisecond Event Ingestion Engine Under 5MB** | Ingestion latency, WebAssembly vs. Docker, Clean/Hexagonal Architecture in Rust, Multi-touch Attribution Domain, Native `fp.js` Fingerprinting (<3.75 KB gzip). | [Read Article](part-1-why-we-ditched-nodejs-for-rust-wasm.md) |
| **02** | **Running WebAssembly Workloads in Production Kubernetes: The Ultimate Lightweight Sidecar Pattern** | GKE Pod design, Spin WASM + Vector 0.43.0 Sidecar, avoiding synchronous queue anti-patterns, Shared Volume (`emptyDir`) NDJSON stream, 512MB On-Disk Buffer, and Pub/Sub Ordering Keys (`account_id`). | [Read Article](part-2-wasm-in-kubernetes-sidecar.md) |
| **03** | **Handling 1,000+ RPS for $30/Month: Elastic Kubernetes Autoscaling with Spot VMs and Rust WASM** | FinOps breakdown, GCP Spot VMs (60-80% discount), HPA v2 Fast Scale-Up (+100%/15s) with zero delay vs. 5-min stabilized scale-down, GKE Cluster Autoscaler (1 to 3 nodes), and live production metrics. | [Read Article](part-3-elastic-autoscaling-finops.md) |
| **04** | **Zero-Click GitOps: How We Ship Infrastructure, WASM, and BigQuery Data Marts via HCP Terraform and GitHub Actions** | Multi-repo GitOps philosophy, CI Quality Gates (Gitleaks, Trivy, Clippy WASM, wasm-opt), HCP Terraform Speculative Plans, and Real-Time BigQuery Lakehouse SQL Marts (DAU, Attribution, Unit Economics). | [Read Article](part-4-zero-click-gitops-lakehouse.md) |

---

## Publishing Guidelines for Medium & Dev.to

* **Tags / Topics:** `#rust` `#webassembly` `#kubernetes` `#devops` `#gcp` `#cloud` `#finops` `#bigquery`
* **Target Audience:** Senior Software Engineers, Solution Architects, DevOps/SREs, CTOs.
* **Format:** Markdown compatible with Dev.to Front Matter and Medium standard markdown import.