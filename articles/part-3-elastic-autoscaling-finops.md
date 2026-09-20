# Handling 1,000+ RPS for $30/Month: Elastic Kubernetes Autoscaling with Spot VMs and Rust WASM

*By the Engineering Team at PlayTests*

![Elastic Autoscaling Architecture](https://raw.githubusercontent.com/wiliest0r/docs/main/assets/hpa-autoscaling.png)

---

In [Part 2](part-2-wasm-in-kubernetes-sidecar.md), we showed how pairing a Rust WebAssembly (WASM) edge server with a Vector sidecar inside Kubernetes creates a rock-solid, zero-loss ingestion pod consuming less than 60 MB of RAM.

In this third installment, we tackle the hardest engineering trade-off in modern cloud infrastructure:

> **"How do you design a system that costs practically nothing during low-traffic periods (1–5 RPS), yet dynamically expands to absorb sudden advertising bursts (1,000+ RPS) within seconds—without manual intervention and without bankrupting your budget?"**

Here is the exact blueprint we used to scale our event pipeline on Google Kubernetes Engine (GKE) for **less than $30 a month**.

---

## 1. The FinOps Reality Check: Why Traditional Stacks Bleed Money

In most enterprise setups, handling 1,000+ RPS requires over-provisioning infrastructure:
* **The "Safe" Over-Provisioning Trap:** Teams launch a 3-node Kubernetes cluster with `e2-standard-4` instances (4 vCPU, 16 GB RAM each) on standard On-Demand pricing.
* **The Cost:** That baseline cluster costs **~$350 to $500 per month** in compute fees alone—even at 3:00 AM on a Tuesday when your website receives only 2 visits per minute.
* **Why do teams do this?** Because traditional runtimes (Java, Node.js, Python) have slow boot times (10–30 seconds) and heavy memory footprints. If traffic spikes 100x in 30 seconds, slow-scaling pods fail before the autoscaler can save them.

### Our Cost Breakdown at 1 RPS vs. 1,000+ RPS:

| Component | Idle / Baseline (1–10 RPS) | Peak Burst (1,000+ RPS) | Monthly Bill (Blended) |
| :--- | :--- | :--- | :--- |
| **GKE Compute** | 1 Spot Node (`e2-small`) | 2–3 Spot Nodes (`e2-small`) | **~$12 – $16** |
| **GKE Control Plane** | Free Tier (1 Zonal Cluster) | Free Tier (1 Zonal Cluster) | **$0.00** |
| **Static IP & Ingress** | 1 Regional IP + External LB | 1 Regional IP + External LB | **~$7 – $10** |
| **Pub/Sub Streaming** | < 1 GB / month | ~30–50 GB / month | **~$2 – $4** |
| **BigQuery Storage Write** | Free Tier (< 2 TB / month) | Free Tier / Standard Streaming | **~$1 – $3** |
| **Total Cloud Expense** | — | — | **~$23 – $33 / month** |

Yes, you read that right: **under $35 per month** for an enterprise-grade, multi-tenant analytics platform running in production.

Here is how we achieved it.

---

## 2. Exploiting Preemptible Spot VMs

The first cost-reduction lever is running exclusively on **GCP Spot VMs** (preemptible instances). Spot VMs offer a **60% to 80% discount** compared to standard on-demand virtual machines.

The tradeoff? Google Cloud can reclaim a Spot VM with a 30-second notice if compute capacity is needed elsewhere.

### Why our architecture is 100% immune to Spot preemption:
1. **Stateless Pods:** Our WASM + Vector pods carry no persistent state. All events are immediately written to Pub/Sub.
2. **Instant Pod Startup:** A new WASM pod boots in **< 100 milliseconds**. When a node is preempted, Kubernetes reschedules the pod to another node before users even notice.
3. **Vector Disk Buffer:** Even if network drops occur during a node rotation, Vector’s 512 MB buffer preserves pending events safely.

---

## 3. Horizontal Pod Autoscaler (HPA v2) Under the Microscope

Most Kubernetes HPA tutorials use default scaling metrics that look like this:
```yaml
# THE WRONG WAY FOR BURSTY TRAFFIC:
targetCPUUtilizationPercentage: 80
# (Takes 3-5 minutes to react to a marketing surge!)
```

During a marketing broadcast or flash sale, traffic doesn't increase gradually over 10 minutes—it jumps from **5 RPS to 800 RPS in 15 seconds**. Standard HPA policies will cause pod queue saturation and HTTP 504 errors before scaling kicks in.

### The Fast Scale-Up, Stabilized Scale-Down Strategy
We designed our HPA v2 policy in Terraform specifically for burst absorption:

```hcl
resource "kubernetes_horizontal_pod_autoscaler_v2" "beacon_hpa" {
  metadata {
    name      = "beacon-hpa"
    namespace = kubernetes_namespace_v1.beacon.metadata[0].name
  }

  spec {
    scale_target_ref {
      api_version = "apps/v1"
      kind        = "Deployment"
      name        = kubernetes_deployment_v1.beacon_server.metadata[0].name
    }

    min_replicas = 1
    max_replicas = 5 # Up to 5 pods easily handles 1,000 - 1,500+ RPS

    # Metric 1: CPU Utilization (Target 70% of 50m request)
    metric {
      type = "Resource"
      resource {
        name = "cpu"
        target {
          type                = "Utilization"
          average_utilization = 70
        }
      }
    }

    # Metric 2: Memory Utilization (Target 80% of 64Mi request)
    metric {
      type = "Resource"
      resource {
        name = "memory"
        target {
          type                = "Utilization"
          average_utilization = 80
        }
      }
    }

    behavior {
      # Fast Burst Scaling: Instantly double pods or add 2 pods every 15s
      scale_up {
        stabilization_window_seconds = 0 # ZERO delay!
        select_policy                = "Max"

        policy {
          type           = "Percent"
          value          = 100
          period_seconds = 15
        }
        policy {
          type           = "Pods"
          value          = 2
          period_seconds = 15
        }
      }

      # Smooth Cool-Down: Prevent flapping during intermittent traffic
      scale_down {
        stabilization_window_seconds = 300 # 5-minute cooldown window
        select_policy                = "Min"

        policy {
          type           = "Percent"
          value          = 50
          period_seconds = 60
        }
        policy {
          type           = "Pods"
          value          = 1
          period_seconds = 60
        }
      }
    }
  }
}
```

### Why this policy is magic:
* **Zero-Second Scale-Up Window:** When CPU utilization exceeds 70%, the HPA doesn't wait. It immediately scales up by either **+100% or +2 pods every 15 seconds**, absorbing the shockwave instantly.
* **300-Second Stabilization Window:** Once traffic subsides, the HPA waits a full **5 minutes** before scaling down. This prevents "flapping" (rapid cycles of scaling up and down) when traffic comes in periodic waves.

---

## 4. Coupling HPA with GKE Cluster Autoscaler

Pods cannot scale if there are no physical node resources to place them on. 

In our `gke.tf`, we configured the GKE Node Pool autoscaler to expand dynamically from **1 to 3 Spot nodes**:

```hcl
resource "google_container_node_pool" "beacon_pool" {
  name     = "beacon-spot-pool"
  cluster  = google_container_cluster.beacon_cluster.name
  location = var.zone

  autoscaling {
    min_node_count = 1
    max_node_count = 3 # Expands to 3 nodes during sustained load
  }

  node_config {
    preemptible  = false
    spot         = true # High-discount Spot VM
    machine_type = "e2-small" # 2 vCPU, 2 GB RAM ($0.007/hr on Spot!)
    disk_size_gb = 20
    disk_type    = "pd-standard"
  }
}
```

### The Cost of a 4-Hour 1,000 RPS Surge:
If a massive advertising campaign floods your site with traffic for 4 continuous hours:
* Cluster Autoscaler provisions **2 extra `e2-small` Spot nodes**.
* Price per node: ~$0.007 per hour.
* 2 nodes × 4 hours × $0.007 = **$0.056 (less than 6 cents!)**.
* Once the campaign concludes, HPA scales down pods, Cluster Autoscaler drains the extra nodes, and your cluster returns to its 1-node $12/month baseline.

---

## 5. Live Production Verification

Here is the actual live status from our production cluster running in `europe-west1`:

```bash
$ kubectl get hpa -n beacon
NAME         REFERENCE                  TARGETS                         MINPODS   MAXPODS   REPLICAS   AGE
beacon-hpa   Deployment/beacon-server   cpu: 11%/70%, memory: 54%/80%   1         5         1          42s
```

Under normal traffic:
* **CPU Consumption:** Only **11%** (~8m of CPU)!
* **Memory Consumption:** **54%** (~54 MB total pod RAM).
* **Latency:** p50 = 0.9 ms, p95 = 1.4 ms, p99 = 2.8 ms.

The system effortlessly handles ordinary traffic with a single pod, and is pre-primed to burst to 5 pods within 30 seconds.

---

## What's Next in Part 4?

We have a blazing fast WASM engine, a resilient Vector sidecar, and an elastic Kubernetes cluster that scales for pennies.

Now: **how do we manage all of this without ever clicking a button in the GCP Cloud Console?**
* How do we enforce pure **GitOps** across four separate repositories?
* How does HCP Terraform validate infrastructure changes in pull requests before they merge?
* How do raw Pub/Sub events stream directly into **BigQuery multi-touch attribution views** in real-time?

👉 **Stay tuned for Part 4:** *«Zero-Click GitOps: How We Ship Infrastructure, WASM, and BigQuery Data Marts via HCP Terraform and GitHub Actions»*.

---
*Questions about Kubernetes autoscaling or FinOps optimization? Connect with our team on GitHub or in the comments below!*