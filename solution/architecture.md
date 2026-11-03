# Corrected system architecture

Proposed target design; not deployed. Managed services are shown outside the VPC boundary.

```mermaid
flowchart LR
  USERS[Users] -->|HTTPS| LB[Existing HTTPS load balancer]

  subgraph ONPREM[On premises]
    ADO[Azure DevOps Server 2022<br/>Git and pipeline]
  end

  subgraph GCP[Google Cloud — production project, me-west1]
    subgraph VPC[Existing VPC]
      AGENT[Ubuntu pipeline agent<br/>attached service account]
      subgraph GKE[Regional private GKE Standard<br/>nodes across zones]
        WEB_SVC[Frontend Service] --> WEB[Frontend pods]
        API_SVC[API Service<br/>ready endpoints] --> API[API replicas]
        WORKER[Background worker pods]
        MIGRATE[One-off migration Job]
      end
    end
    AR[Artifact Registry<br/>immutable image digests]
    SQL[Cloud SQL PostgreSQL<br/>regional HA, private IP, PITR]
    GCS[Private Cloud Storage<br/>7-day soft delete]
    SM[Secret Manager]
  end

  ADO <-->|Existing site-to-site VPN| AGENT
  AGENT -->|Push tested image| AR
  AGENT -->|Deploy via private API<br/>service account + Kubernetes RBAC| GKE
  LB --> WEB_SVC
  LB --> API_SVC
  API -->|Private IP| SQL
  WORKER -->|Private IP| SQL
  MIGRATE -->|Private IP| SQL
  API -->|HTTPS, Workload Identity/IAM| GCS
  WORKER -->|HTTPS, Workload Identity/IAM| GCS
  API -.->|Workload Identity| SM
  WORKER -.->|Workload Identity| SM
  GKE -->|Pull image| AR
```

The agent connects to Azure DevOps over the existing VPN, pushes to Artifact Registry, and deploys through the private GKE API. GKE pods use Workload Identity; no service-account key files are used. Cloud SQL is reached over private IP; Cloud Storage is a Google-managed service outside the VPC, accessed by authenticated API calls rather than as a VPC resource.

Regional GKE capacity and Cloud SQL HA target a single-zone failure. PITR and bucket soft delete address recovery, not availability. All have cost/quota implications; regional disaster recovery is out of scope.
