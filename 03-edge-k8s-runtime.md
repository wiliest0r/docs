# Level 3: Edge & Kubernetes Runtime Layer (`deploy` & GKE)

This document details the container and WebAssembly orchestration layer across both **Local/Edge (WSL2 k3s)** and **Cloud (GCP GKE Standard)** environments.

---

## 1. Dual-Runtime Orchestration Flowchart

```mermaid
flowchart TD
    %% =========================================================================
    %% TRAFFIC SOURCE
    %% =========================================================================
    Client["Client Traffic\n(Browser / SDK)"]

    %% =========================================================================
    %% EDGE / LOCAL K3S TOPOLOGY
    %% =========================================================================
    subgraph EdgeTopology["Environment 1: Local / Edge k3s Cluster (WSL2)"]
        EdgeIngress["Traefik Ingress Controller\n(Port 80 / 443)"]
        EdgeService["Service: beacon-service\n(Type: NodePort, Port 80 -> 30080)"]
        EdgePod["Pod: beacon-server\n(Image: ghcr.io/wiliest0r/beacon-wasm:v0.1.0)"]
        
        subgraph SpinCRI["Containerd Spin CRI Integration"]
            EdgeRC["RuntimeClass: wasm-spin\n(handler: spin)"]
            EdgeShim["containerd-shim-spin-v2\n(/usr/local/bin/containerd-shim-spin-v2)"]
            EdgeConfig["/var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl\n[plugins.'io.containerd.grpc.v1.cri'.containerd.runtimes.spin]"]
        end

        EdgeIngress --> EdgeService --> EdgePod
        EdgePod -->|runtimeClassName| EdgeRC --> EdgeConfig --> EdgeShim
    end

    %% =========================================================================
    %% CLOUD GKE TOPOLOGY
    %% =========================================================================
    subgraph CloudGKE["Environment 2: GCP GKE Standard Cluster (playtests-beacon)"]
        GCP_LB["Google Cloud External Network Load Balancer\n(Regional Public IPv4)"]
        GKE_Service["Service: beacon-service\n(Namespace: beacon, Type: LoadBalancer)"]
        GKE_Deployment["Deployment: beacon-server\n(Namespace: beacon, Replicas: 1)"]
        GKE_Pod["Pod: beacon-server\n(CPU: 50m-250m, RAM: 64Mi-256Mi)"]
        
        subgraph GKENodePool["Managed Node Pool: beacon-node-pool-dev"]
            GKE_Node["Worker Node: e2-medium (europe-west1-b)\n(Autoscaling: 1 to 2 nodes, Shielded VM)"]
            GKE_SA["Node SA: sa-gke-nodes-dev\n- logging.logWriter\n- monitoring.metricWriter\n- monitoring.viewer\n- artifactregistry.reader"]
        end

        subgraph GKENetwork["VPC-Native Networking: beacon-vpc-dev"]
            GKE_Subnet["Subnet: beacon-gke-subnet-dev\n- Nodes CIDR: 10.10.0.0/20\n- Secondary Pods CIDR: 10.20.0.0/16\n- Secondary Services CIDR: 10.30.0.0/20"]
            GKE_WI["Workload Identity Pool:\nplaytests-beacon.svc.id.goog"]
        end

        GCP_LB --> GKE_Service --> GKE_Deployment --> GKE_Pod
        GKE_Pod --> GKE_Node
        GKE_Node --> GKE_SA
        GKE_Node --> GKE_Subnet
        GKE_Pod --> GKE_WI
    end

    Client -->|Local Edge Testing| EdgeIngress
    Client -->|Cloud Production Ingress| GCP_LB
```

---

## 2. Infrastructure Comparison

| Metric / Dimension | Local / Edge k3s | Cloud GKE Standard |
| :--- | :--- | :--- |
| **Location** | Local WSL2 / Ubuntu Edge Node | GCP `europe-west1-b` (Single Zone) |
| **Management Cost** | $0.00 (Self-hosted) | $0.00 Control Plane (Google Cloud Free Tier Zonal Cluster) |
| **Ingress Engine** | Traefik v2 (k3s embedded) | Google Cloud External L4 Network Load Balancer |
| **Container Engine** | containerd with Spin shim | Google Container-Optimized OS (COS) |
| **WASM Execution** | `containerd-shim-spin-v2` | Native container / Spin shim daemonset |
| **Network Model** | Flannel (Host-GW / VXLAN) | VPC-Native secondary CIDR ranges (Pods & Services) |
| **Identity / IAM** | Local Kubeconfig / RBAC | GCP Workload Identity Federation (`.svc.id.goog`) |
| **IaC Automation** | Shell scripts (`deploy/k3s-setup.sh`) | HCP Terraform declarative IaC (`gke.tf` & `k8s_manifests.tf`) |

---

## 3. Kubernetes Declarative Manifest Structure

In accordance with GitOps standards, all Kubernetes cloud resources are provisioned directly through HCP Terraform's declarative `kubernetes` provider:

1. **Namespace (`beacon`)**: Dedicated boundary isolating telemetry pods from system services.
2. **RuntimeClass (`wasm-spin`)**: Registered runtime handler pointing to `spin` for native WASM execution.
3. **Deployment (`beacon-server`)**: Configures zero-downtime rolling updates, resource limits (`250m` CPU / `256Mi` RAM), and environment variable binding (`ENVIRONMENT=dev`, `APP_VERSION=0.1.0`).
4. **Service (`beacon-service`)**: Exposes port `80` with `type: LoadBalancer`, triggering GCP Cloud Load Balancing to allocate a static public IPv4 address.
