# Pega on AKS — C4 (C1–C4) + Sequence Diagrams

> Opinionated, vendor-aligned architecture sketches to understand **who talks to whom** and **why**. Diagrams use Mermaid. Assumes Pega 8.7+ on AKS with **externalized Kafka** (recommended) and SRS (Search & Reporting Service) backed by Elasticsearch/OpenSearch.

---

## C1 — System Context (AKS-hosted Pega Platform)

```mermaid
flowchart LR
U["Business/User"]
OPS["Ops/SRE"]
AGW["Azure App Gateway / WAF"]
INGRESS["K8s Ingress (AGIC)"]


subgraph AKS["AKS Cluster"]
WEB["Pega Web Pods"]
BATCH["Pega Batch Pods"]
SRS["Pega SRS Pods"]
SVC["K8s Services"]
end


SQL[("Azure SQL DB")]
BLOB[("Azure Blob Storage")]
KAFKA[("Kafka / Event Hubs")]
ES[("Elasticsearch / OpenSearch")]
ENTRA["Microsoft Entra ID"]
OBS["Observability"]


U -->|HTTPS| AGW
AGW --> INGRESS
INGRESS --> SVC
SVC --> WEB


OPS -->|helm/kubectl| AKS


WEB -->|JDBC TLS| SQL
BATCH -->|JDBC TLS| SQL


WEB -->|Repository API| BLOB
BATCH -->|Repository API| BLOB


WEB -->|Kafka| KAFKA
BATCH -->|Kafka| KAFKA


%% --- hard break between sections ---


SRS -->|HTTP| ES
WEB --> SRS


WEB -->|OIDC/SAML| ENTRA


WEB -->|metrics/logs| OBS
BATCH -->|metrics/logs| OBS
```

---

## C2 — Containers inside the System

```mermaid
C4Container
title C2: Pega on AKS — Containers
System_Boundary(aks, "AKS Cluster") {
  Container(ing, "Ingress (AGIC-managed)", "K8s Ingress", "Routes from App Gateway to Services")
  Container(web, "Pega Web Pods", "Java", "UI, REST/Services, Rule Resolution")
  Container(batch, "Pega Batch/Agents Pods", "Java", "Queue processors, Job schedulers, BIX")
  Container(srs, "Pega SRS Pods", "Java", "Search/Reporting; talks to ES/OS")
  Container(cm, "Config/Secrets", "K8s CM/Secrets", "DB, Kafka, OIDC endpoints, repo creds")
  Container(svc, "K8s Services", "ClusterIP/LoadBalancer", "Stable backends for pods")
}
Container_Ext(appgw, "Azure App Gateway", "L7 LB", "Public/Private entrypoint")
Container_Ext(sql, "Azure SQL", "PaaS DB", "Rules/Data schemas")
Container_Ext(kafka, "Kafka-Compat (Event Hubs)", "Messaging", "Externalized stream")
Container_Ext(storage, "Azure Blob", "Object store", "Repository, attachments")
Container_Ext(search, "Elasticsearch/OpenSearch", "Index", "Backs SRS")

Rel(appgw, ing, "HTTP/HTTPS")
Rel(ing, svc, "HTTP")
Rel(svc, web, "TCP (HTTP)")
Rel(svc, batch, "TCP")
Rel(web, sql, "JDBC over TLS")
Rel(batch, sql, "JDBC over TLS")
Rel(web, kafka, "Produce/Consume")
Rel(batch, kafka, "Consume/Produce")
Rel(srs, search, "Index/Search API")
Rel(web, storage, "Put/Get via Repository")
Rel(batch, storage, "Bulk Put/Get (BIX, file ops)")
```

---

## C3 — Key Components per Container

### C3.1 Pega Web Pod Components

```mermaid
C4Component
title C3: Pega Web Pod
Container_Boundary(web, "Pega Web Pod") {
  Component(ui, "UI/Portal", "DX/Rules-driven UI")
  Component(rest, "REST/SOAP Services", "API endpoints")
  Component(auth, "Auth Adapter", "OIDC/SAML with Entra ID")
  Component(rr, "Rule Resolution", "Applies rulesets from DB")
  Component(qp_client, "Queue Producer", "Kafka client")
  Component(repo, "Repository Adapter", "Azure Blob Repo")
}
Rel(auth, ui, "SSO session")
Rel(rr, rest, "Evaluate rules")
Rel(qp_client, rest, "Emit events/queue items")
Rel(repo, ui, "Upload/Download artifacts")
```

### C3.2 Pega Batch/Agents Pod Components

```mermaid
C4Component
title C3: Pega Batch/Agents Pod
Container_Boundary(batch, "Pega Batch/Agents Pod") {
  Component(qp_cons, "Queue Processors", "Kafka consumer groups")
  Component(agents, "Background Agents", "Scheduled jobs")
  Component(bix, "BIX Extractor", "Bulk export to storage/DB")
}
Rel(qp_cons, agents, "Trigger downstream work")
Rel(bix, agents, "Scheduled exports")
```

### C3.3 SRS (Search & Reporting Service)

```mermaid
C4Component
title C3: Pega SRS Pod
Container_Boundary(srs, "SRS Pod") {
  Component(idx, "Indexer", "Writes to ES/OS")
  Component(qry, "Query API", "Faceted search/analytics")
}
Rel(idx, qry, "Index → Query coherence")
```

### C3.4 Ingress Path

```mermaid
C4Component
title C3: Ingress Path (AGIC)
Container_Boundary(ing, "K8s Ingress (AGIC)") {
  Component(rule, "Ingress Rules", "Host/path routing")
  Component(tls, "TLS Termination", "Certs via App Gateway")
}
```

---

## C4 — Code/Deployment-Level View (K8s/Helm resources)

```mermaid
flowchart TB
subgraph HELM["Helm Charts"]
A["pega values.yaml"]
B["addons values - AGIC & metrics"]
end


subgraph WORKLOADS["K8s Workloads"]
D["Deployment: pega-web"]
E["Deployment: pega-batch"]
F["StatefulSet: srs"]
H["Job: DB schema setup/upgrade"]
end


subgraph INFRA["K8s Infra"]
I["Service: web-svc"]
J["Service: batch-svc"]
K["Ingress: routes to web-svc"]
L["ConfigMap: app settings"]
M["Secret: db/kafka/repo creds"]
N["HPA: web autoscale"]
O["PDB: disruption budget"]
end


subgraph AZURE["Azure"]
P["App Gateway"]
Q["Azure SQL"]
R["Azure Blob"]
S["Event Hubs (Kafka)"]
T["Elasticsearch / OpenSearch"]
end


A --> D
A --> E
A --> F
A --> H
K --> I --> D
J --> E
P --> K
D --> Q
E --> Q
D --> R
E --> R
D --> S
E --> S
F --> T
N --> D
L --> D
L --> E
L --> F
M --> D
M --> E
M --> F
O --> D
O --> E
O --> F
```

---

# Sequence Diagrams — Comprehensive Scenarios

## S1 — User Login & Portal Load (OIDC with Entra ID)

```mermaid
sequenceDiagram
  autonumber
  participant U as User Browser
  participant AG as App Gateway
  participant IG as K8s Ingress
  participant W as Pega Web Pod
  participant ID as Entra ID (OIDC)
  participant DB as Azure SQL
  participant SRS as SRS

  U->>AG: HTTPS GET /prweb
  AG->>IG: Route via host/path
  IG->>W: Forward request
  W-->>U: 302 to ID (OIDC auth)
  U->>ID: Authenticate (MFA/Password)
  ID-->>U: Auth code
  U->>W: Callback with code
  W->>ID: Token exchange
  ID-->>W: ID/Access token
  W->>DB: Load operator, access roles
  W->>SRS: Warmup search facets (optional)
  W-->>U: Portal HTML/CSS/JS + session cookie
```

## S2 — Case Creation with Background Processing

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant W as Pega Web Pod
  participant DB as Azure SQL
  participant K as Kafka (Event Hubs)
  participant B as Pega Batch Pod

  U->>W: Submit new case
  W->>DB: Insert case work object, audit
  W->>K: Produce message (Queue Processor topic)
  Note over W,K: SASL_SSL with client id/secret
  B->>K: Consume message (consumer group)
  B->>DB: Apply business rules, update case
  B-->>W: Optional notify via DB/state
  W-->>U: Case ID + status
```

## S3 — File/Document Attachment to Azure Blob (Repository)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant W as Pega Web Pod
  participant Repo as Azure Blob Repo
  participant DB as Azure SQL

  U->>W: Upload document
  W->>Repo: PUT object (SAS/Repo credentials)
  Repo-->>W: 201 Created + object URI
  W->>DB: Persist metadata & URI
  W-->>U: Attachment linked to case
```

## S4 — Search/Reporting via SRS + Elasticsearch

```mermaid
sequenceDiagram
  autonumber
  participant W as Pega Web Pod
  participant S as SRS Pod
  participant ES as Elasticsearch/OpenSearch
  participant DB as Azure SQL

  W->>DB: Commit case changes
  W->>S: Publish index event (internal API)
  S->>ES: Index/Update document
  W->>S: Query facets/results
  S->>ES: Execute query
  ES-->>S: Hits/aggregations
  S-->>W: Result set
```

## S5 — Deployment/Upgrade via Helm (Schema, Rollout)

```mermaid
sequenceDiagram
  autonumber
  participant OPS as Ops/SRE
  participant Helm as Helm CLI
  participant API as K8s API Server
  participant Job as Helm Hook Job
  participant DB as Azure SQL
  participant Web as Deployment: pega-web
  participant Batch as Deployment: pega-batch

  OPS->>Helm: helm upgrade --install pega ...
  Helm->>API: Create/Update manifests
  API->>Job: Run pre-install/upgrade hooks
  Job->>DB: Apply rules/data schema changes
  Job-->>API: Succeeded
  API->>Web: RollingUpdate (new ReplicaSet)
  API->>Batch: RollingUpdate
  Web-->>OPS: Ready
  Batch-->>OPS: Ready
```

## S6 — Autoscaling Web Tier (HPA)

```mermaid
sequenceDiagram
autonumber
participant HPA as K8s HPA
participant MET as Metrics API
participant WEB as Pega Web Deployment
participant API as K8s API Server


HPA->>MET: Get CPU/QPS metrics
MET-->>HPA: Current utilization
alt Above target
HPA->>API: Scale WEB replicas up
API->>WEB: Create Pods
WEB-->>HPA: New pods Ready
else Below target
HPA->>API: Scale WEB replicas down
API->>WEB: Terminate Pods (respect PDB)
end
```

## S7 — Externalized Kafka vs Legacy Stream Tier

```mermaid
sequenceDiagram
  autonumber
  participant W as Pega Web Pod
  participant B as Pega Batch Pod
  participant K as External Kafka (Event Hubs)
  participant ST as (Deprecated) Stream Tier

  Note over W,B: Recommended: External Kafka (SASL_SSL)
  W->>K: Produce queue item
  B->>K: Consume queue item
  Note over ST: Legacy stream tier is deprecated 8.7+
```

## S8 — Blue/Green or Canary via Helm Values

```mermaid
sequenceDiagram
  autonumber
  participant OPS as Ops/SRE
  participant Helm as Helm
  participant API as K8s API Server
  participant V1 as Web v1 RS
  participant V2 as Web v2 RS
  participant IG as Ingress

  OPS->>Helm: Set canary weight or bg labels
  Helm->>API: Apply manifests (V2)
  API->>V2: Launch pods
  IG->>V1: 90% traffic (e.g.)
  IG->>V2: 10% traffic (e.g.)
  OPS->>IG: Observe metrics/errors
  alt Healthy
    IG->>V2: Ramp to 100%
    API->>V1: Scale down
  else Issues
    IG->>V1: Rollback traffic
    API->>V2: Scale down
  end
```

---

## Notes & Guardrails

* **Kafka**: Prefer **externalized** Kafka (e.g., Azure Event Hubs Kafka endpoint). Lock down with **SASL_SSL**, per-namespace creds.
* **Search**: Use **SRS** backed by **Elasticsearch/OpenSearch** managed service if available.
* **DB**: Separate **rules** and **data** schemas on **Azure SQL**; enforce TLS and least-privileged logins.
* **Ingress**: AGIC + App Gateway for enterprise-grade L7 WAF, path/host routing, TLS.
* **Scaling**: HPA for Web; Batch usually scales by throughput needs; protect with **PDB**, **Requests/Limits**, **Pod Anti-Affinity**.
* **Secrets**: Store in **K8s Secrets**; consider **Azure Key Vault CSI Driver** for rotation.
* **Upgrades**: Helm hooks must gate DB migrations; blue/green/canary to minimize risk.
* **Observability**: Ship **metrics/logs/traces** (Azure Monitor/Prom/Grafana/ELK).
* **Storage**: Prefer **Azure Blob** Repo for large attachments; DB BLOBs for simplicity-only PoCs.
* **Networking**: Private endpoints for SQL/Storage/Event Hubs where possible.

---

## Checklist to Map Doc → Values.yaml

* `provider: aks`
* `stream.enabled: false` and **externalized Kafka** block configured (bootstrap, securityProtocol, jaas, trust/key stores or EH SAS)
* `jdbc.url`, `jdbc.driverClass`, `jdbc.dbType: mssql`, `jdbc.driverUri`
* `global.search.externalSrs: true` (if chart supports) and ES endpoints for SRS
* `ingress:` (AGIC) host/path rules
* `repository:` section for Azure Blob Repo (account/SAS/container)
* `resources`, `affinity`, `tolerations`, `podDisruptionBudget` for each role
* `hpa:` for web tier; optional for batch
* `init/upgrade jobs:` enabled and ordered
