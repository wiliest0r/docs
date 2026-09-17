# Level 5: Zero-Trust Security, IAM & Workload Identity Federation (WIF)

This document specifies the identity and access management architecture governing **Google Cloud Platform (`playtests-beacon`)**, **HCP Terraform (`PlayTests`)**, and **GitHub Actions (`wiliest0r`)**.

---

## 1. Zero-Trust OIDC Federation & IAM Boundaries

```mermaid
flowchart TD
    %% =========================================================================
    %% OIDC ISSUERS
    %% =========================================================================
    subgraph OIDCIssuers["1. Cryptographic OIDC Identity Issuers"]
        GH_OIDC["GitHub Actions OIDC Token\n- Issuer: https://token.actions.githubusercontent.com\n- Subject: repo:wiliest0r/beacon:ref:refs/heads/*\n- Audience: //iam.googleapis.com/..."]
        TFC_OIDC["HCP Terraform OIDC Token\n- Issuer: https://app.terraform.io\n- Subject: organization:PlayTests:project:*:workspace:terraform-central-*\n- Audience: aws.workload.identity / gcp"]
    end

    %% =========================================================================
    %% WORKLOAD IDENTITY FEDERATION
    %% =========================================================================
    subgraph WIFPool["2. GCP Workload Identity Pool: tfc-pool (Project 847948858817)"]
        GH_Provider["WIF Provider: github-provider\nAttribute Condition Filter:\nassertion.repository == 'wiliest0r/beacon'"]
        TFC_Provider["WIF Provider: tfc-provider\nAttribute Condition Filter:\nassertion.terraform_organization_name == 'PlayTests' &&\nassertion.terraform_workspace_name in ['terraform-central-dev', 'terraform-central-stage', 'terraform-central-prod']"]
    end

    GH_OIDC -->|Presents JWT| GH_Provider
    TFC_OIDC -->|Presents JWT| TFC_Provider

    %% =========================================================================
    %% SERVICE ACCOUNTS (LEAST PRIVILEGE)
    %% =========================================================================
    subgraph ServiceAccounts["3. Scoped Service Accounts (No Static Keys)"]
        SA_Deployer["sa-gha-deployer@playtests-beacon.iam.gserviceaccount.com\n(Scoped CI/CD Deployer)"]
        SA_Executor["sa-terraform-executor@playtests-beacon.iam.gserviceaccount.com\n(Scoped IaC Manager - NO OWNER ROLE)"]
        SA_Runtime["sa-beacon-runtime-{dev,stage,prod}\n(Isolated Cloud Run Execution)"]
        SA_GKE["sa-gke-nodes-dev\n(Isolated GKE Node Execution)"]
    end

    GH_Provider -->|Impersonates via roles/iam.workloadIdentityUser| SA_Deployer
    TFC_Provider -->|Impersonates via roles/iam.workloadIdentityUser| SA_Executor

    %% =========================================================================
    %% PERMISSION MAPPINGS
    %% =========================================================================
    subgraph GCPPermissions["4. Granular IAM Role Grants"]
        subgraph DeployerRoles["Deployer Scoped Roles"]
            R_AR_Write["roles/artifactregistry.writer"]
            R_Run_Dev["roles/run.developer"]
            R_SA_User_Scoped["roles/iam.serviceAccountUser\n(Strictly on sa-beacon-runtime-* SAs)"]
        end

        subgraph ExecutorRoles["IaC Scoped Roles (Least Privilege)"]
            R_Run_Admin["roles/run.admin"]
            R_AR_Admin["roles/artifactregistry.admin"]
            R_GKE_Admin["roles/container.admin"]
            R_Net_Admin["roles/compute.networkAdmin"]
            R_IAM_Admin["roles/iam.serviceAccountAdmin"]
            R_Res_Admin["roles/resourcemanager.projectIamAdmin"]
            R_SU_Consumer["roles/serviceusage.serviceUsageConsumer"]
        end

        subgraph RuntimeRoles["Container & Node Minimal Roles"]
            R_Log["roles/logging.logWriter"]
            R_Metrics["roles/monitoring.metricWriter"]
            R_AR_Read["roles/artifactregistry.reader"]
        end
    end

    SA_Deployer --> DeployerRoles
    SA_Executor --> ExecutorRoles
    SA_Runtime --> R_Log
    SA_GKE --> R_Log & R_Metrics & R_AR_Read
```

---

## 2. Workload Identity Federation (WIF) Claim Matrix

### HCP Terraform Federation (`tfc-provider`)
* **Issuer**: `https://app.terraform.io`
* **Audience**: `projects/847948858817/locations/global/workloadIdentityPools/tfc-pool/providers/tfc-provider`
* **IAM Binding on `sa-terraform-executor`**:
  ```
  principalSet://iam.googleapis.com/projects/847948858817/locations/global/workloadIdentityPools/tfc-pool/attribute.terraform_workspace_name/terraform-central-dev
  principalSet://iam.googleapis.com/projects/847948858817/locations/global/workloadIdentityPools/tfc-pool/attribute.terraform_workspace_name/terraform-central-stage
  principalSet://iam.googleapis.com/projects/847948858817/locations/global/workloadIdentityPools/tfc-pool/attribute.terraform_workspace_name/terraform-central-prod
  ```
  *Result*: Any attempt to authenticate from unauthorized workspaces or other HCP Terraform organizations is rejected at the GCP identity boundary.

### GitHub Actions Federation (`github-provider`)
* **Issuer**: `https://token.actions.githubusercontent.com`
* **Attribute Mapping**:
  * `google.subject` = `assertion.sub`
  * `attribute.repository` = `assertion.repository`
  * `attribute.repository_owner` = `assertion.repository_owner`
  * `attribute.ref` = `assertion.ref`
  * `attribute.environment` = `assertion.environment`
* **IAM Binding on `sa-gha-deployer`**:
  ```
  principalSet://iam.googleapis.com/projects/847948858817/locations/global/workloadIdentityPools/tfc-pool/attribute.repository/wiliest0r/beacon
  ```
  *Result*: Only workflows originating from `wiliest0r/beacon` can request credentials to push images and trigger Cloud Run deployments.

---

## 3. Principle of Least Privilege (PoLP) IAM Summary

| Role | Target Identity | Justification |
| :--- | :--- | :--- |
| `roles/run.admin` | `sa-terraform-executor` | Create, configure, and manage Cloud Run services |
| `roles/container.admin` | `sa-terraform-executor` | Provision and configure GKE Standard clusters and node pools |
| `roles/compute.networkAdmin` | `sa-terraform-executor` | Provision dedicated VPC, subnets, and secondary IP ranges |
| `roles/artifactregistry.admin` | `sa-terraform-executor` | Provision Artifact Registry repositories and enforce tag immutability |
| `roles/iam.serviceAccountAdmin` | `sa-terraform-executor` | Create dedicated runtime and node service accounts |
| `roles/resourcemanager.projectIamAdmin` | `sa-terraform-executor` | Bind minimal IAM roles to provisioned service accounts |
| `roles/artifactregistry.writer` | `sa-gha-deployer` | Push compiled container / WASM OCI images from CI/CD |
| `roles/run.developer` | `sa-gha-deployer` | Update Cloud Run container image references to newly pushed tags |
| `roles/iam.serviceAccountUser` | `sa-gha-deployer` (Resource-scoped) | Impersonate `sa-beacon-runtime-*` during Cloud Run revision deployment |
| `roles/logging.logWriter` | `sa-beacon-runtime-*` | Write structured application stdout telemetry to Google Cloud Logging |
| `roles/logging.logWriter` + `monitoring.metricWriter` | `sa-gke-nodes-dev` | Export node diagnostics and metrics to Cloud Operations |
