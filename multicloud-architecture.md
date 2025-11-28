# Multi-Cloud Architecture Strategy
## AI Investment Alert Platform

---

## 1. Architecture Overview

### 1.1 Cloud Distribution Strategy

| Capability | Cloud Provider | Rationale |
|------------|---------------|-----------|
| **Core Services & Infrastructure** | Microsoft Azure | AKS, managed services, enterprise reliability |
| **AI/ML & Embeddings** | Google Cloud (Vertex AI) | Superior AI capabilities, cost-effective embeddings |
| **Search & Indexing** | Google Cloud (Vertex AI Search) | Native vector search, semantic understanding |

### 1.2 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENTS                                         │
│                    (Flutter Mobile App / Admin Web)                          │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AZURE FRONT DOOR (CDN + WAF)                         │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼───────────────────────────────────────────┐
│                                                                              │
│                        AZURE KUBERNETES SERVICE (AKS)                        │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ │
│  │   User     │ │Subscription│ │ Ingestion  │ │   Alert    │ │   Admin    │ │
│  │  Service   │ │  Service   │ │  Service   │ │  Service   │ │  Service   │ │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘ └────────────┘ │
│  ┌────────────┐ ┌────────────┐ ┌────────────────────────────────────────┐   │
│  │Notification│ │Performance │ │           API Gateway (Kong)           │   │
│  │  Service   │ │  Service   │ └────────────────────────────────────────┘   │
│  └────────────┘ └────────────┘                                              │
│                                                                              │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
┌───────────────┐    ┌─────────────────┐    ┌─────────────────────────────────┐
│  AZURE DATA   │    │  AZURE SERVICE  │    │      GOOGLE CLOUD PLATFORM      │
│               │    │      BUS        │    │                                 │
│ ┌───────────┐ │    │  (Event Queue)  │    │  ┌─────────────────────────┐   │
│ │PostgreSQL │ │    └────────┬────────┘    │  │  AI Scoring Service     │   │
│ │ Flexible  │ │             │             │  │  (Cloud Run + Vertex)   │   │
│ └───────────┘ │             │             │  └─────────────────────────┘   │
│ ┌───────────┐ │             │             │  ┌─────────────────────────┐   │
│ │   Redis   │ │             └─────────────┼─▶│  Vertex AI Embeddings   │   │
│ │  Cache    │ │                           │  │  (text-embedding-005)   │   │
│ └───────────┘ │                           │  └─────────────────────────┘   │
│ ┌───────────┐ │                           │  ┌─────────────────────────┐   │
│ │   Blob    │ │                           │  │  Vertex AI Search       │   │
│ │  Storage  │ │                           │  │  (Vector Store)         │   │
│ └───────────┘ │                           │  └─────────────────────────┘   │
└───────────────┘                           └─────────────────────────────────┘
```

---

## 2. Azure Infrastructure (Core Platform)

### 2.1 Azure Kubernetes Service (AKS)

**Configuration:**
- **Node Pools:**
  - System Pool: 2 nodes (Standard_D2s_v3) - Always on
  - User Pool: 2-10 nodes (Standard_D4s_v3) - Auto-scaling
  - Spot Pool: 0-5 nodes (Standard_D4s_v3) - Cost optimization for batch jobs

- **Namespaces:**
  - `prod` - Production workloads
  - `staging` - Pre-production testing
  - `monitoring` - Prometheus, Grafana
  - `ingress` - Kong API Gateway

**Services Deployed on AKS:**

| Service | Replicas (Min/Max) | Resource Limits | Scaling Trigger |
|---------|-------------------|-----------------|-----------------|
| User Service | 2/5 | 512Mi, 0.5 CPU | CPU > 70% |
| Subscription Service | 2/4 | 512Mi, 0.5 CPU | CPU > 70% |
| Ingestion Service | 2/8 | 1Gi, 1 CPU | Queue depth |
| Alert Service | 3/15 | 1Gi, 1 CPU | Market hours + Queue |
| Notification Service | 2/10 | 512Mi, 0.5 CPU | Queue depth |
| Performance Service | 1/3 | 1Gi, 1 CPU | Scheduled (EOD) |
| Admin Service | 1/2 | 256Mi, 0.25 CPU | Minimal |
| Kong Gateway | 2/5 | 512Mi, 0.5 CPU | Request rate |

### 2.2 Azure Database for PostgreSQL (Flexible Server)

**Configuration:**
- **Tier:** General Purpose
- **Compute:** Standard_D2ds_v4 (2 vCores, 8GB RAM)
- **Storage:** 128 GB (auto-grow enabled)
- **High Availability:** Zone-redundant (production)
- **Backup:** 7-day retention, geo-redundant

**Database per Service:**
```
├── investdb_users        (User Service)
├── investdb_subscriptions (Subscription Service)
├── investdb_ingestion    (Ingestion Service)
├── investdb_alerts       (Alert Service)
├── investdb_performance  (Performance Service)
├── investdb_admin        (Admin Service)
└── investdb_notifications (Notification Service)
```

### 2.3 Azure Cache for Redis

**Configuration:**
- **Tier:** Standard C1 (1GB)
- **Use Cases:**
  - Session management (JWT tokens)
  - Rate limiting counters
  - Alert delivery deduplication
  - API response caching

### 2.4 Azure Service Bus

**Configuration:**
- **Tier:** Standard
- **Queues/Topics:**

| Queue/Topic | Publisher | Subscriber | Purpose |
|-------------|-----------|------------|---------|
| `recommendation-ingested` | Ingestion | AI Scoring (GCP) | Trigger scoring |
| `recommendation-approved` | AI Scoring (GCP) | Alert Service | Trigger alerts |
| `recommendation-rejected` | AI Scoring (GCP) | Admin Service | Dashboard |
| `alert-push` | Alert Service | Notification | Push delivery |
| `alert-email` | Alert Service | Notification | Email delivery |
| `subscription-events` | Subscription | User, Alert | Lifecycle events |

### 2.5 Azure Blob Storage

**Containers:**
- `raw-ingestion-logs` - 90-day retention, cool tier
- `documents` - Terms, invoices, reports
- `exports` - User data exports

### 2.6 Azure Front Door

**Configuration:**
- **WAF Policy:** OWASP 3.2 rules
- **CDN:** Static assets caching
- **SSL:** Managed certificates
- **Routing:** Path-based to AKS ingress

---

## 3. Google Cloud Platform (AI/ML Layer)

### 3.1 Why Google Cloud for AI?

| Capability | Google Advantage | Azure Alternative | Decision |
|------------|-----------------|-------------------|----------|
| Text Embeddings | Vertex AI text-embedding-005 (best-in-class) | Azure OpenAI ada-002 | GCP - 40% cost savings |
| Semantic Search | Vertex AI Search (native vector) | Azure AI Search | GCP - Better relevance |
| Custom ML | Vertex AI AutoML | Azure ML | GCP - Simpler fine-tuning |
| LLM Integration | Gemini Pro (if needed) | Azure OpenAI GPT-4 | GCP - Cost-effective |

### 3.2 Vertex AI Embeddings

**Use Cases:**
1. **Stock Recommendation Embeddings** - Semantic similarity for de-duplication
2. **News Sentiment Analysis** - Embed news headlines for sentiment scoring
3. **User Query Understanding** - Natural language search over alerts

**Model:** `text-embedding-005`
- Dimensions: 768
- Max tokens: 2048
- Cost: ~$0.0001 per 1K tokens

**Implementation:**
```python
from google.cloud import aiplatform

def generate_embedding(text: str) -> list[float]:
    """Generate embedding using Vertex AI"""
    model = aiplatform.TextEmbeddingModel.from_pretrained("text-embedding-005")
    embeddings = model.get_embeddings([text])
    return embeddings[0].values
```

### 3.3 Vertex AI Search (Vector Store)

**Configuration:**
- **Data Store:** Stock recommendations corpus
- **Index Type:** Approximate Nearest Neighbor (ANN)
- **Dimensions:** 768 (matching embedding model)

**Use Cases:**
1. **Duplicate Detection** - Find similar recommendations across sources
2. **Historical Search** - "Show me IT sector recommendations from last month"
3. **Related Alerts** - "Similar opportunities to this alert"

### 3.4 AI Scoring Service (Cloud Run)

**Why Cloud Run instead of AKS for AI Scoring:**
- Auto-scales to zero (cost savings during non-market hours)
- Native GCP networking to Vertex AI (low latency)
- GPU support if needed for future ML models
- Pay-per-request pricing

**Configuration:**
- **Memory:** 2Gi
- **CPU:** 2
- **Concurrency:** 80
- **Min Instances:** 0 (scale to zero)
- **Max Instances:** 20
- **Timeout:** 60s

**Container Image:** Hosted in Google Artifact Registry

---

## 4. Cross-Cloud Communication

### 4.1 Connectivity Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         AZURE                                    │
│  ┌─────────────┐                                                │
│  │ AKS Cluster │                                                │
│  │             │──────┐                                         │
│  │ Ingestion   │      │                                         │
│  │ Service     │      │    ┌─────────────────────────────────┐  │
│  └─────────────┘      │    │      Azure Service Bus          │  │
│                       ├───▶│  Topic: recommendation-ingested │  │
│  ┌─────────────┐      │    └───────────────┬─────────────────┘  │
│  │ Alert       │      │                    │                    │
│  │ Service     │◀─────┘                    │                    │
│  └─────────────┘                           │                    │
└────────────────────────────────────────────┼────────────────────┘
                                             │
                              ┌──────────────┼──────────────┐
                              │   INTERNET   │              │
                              │   (HTTPS)    │              │
                              └──────────────┼──────────────┘
                                             │
┌────────────────────────────────────────────▼────────────────────┐
│                        GOOGLE CLOUD                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                  Cloud Run                                   ││
│  │  ┌───────────────────────────────────────────────────────┐  ││
│  │  │              AI Scoring Service                        │  ││
│  │  │                                                        │  ││
│  │  │  1. Receive from Azure Service Bus (Pull)              │  ││
│  │  │  2. Generate Embeddings (Vertex AI)                    │  ││
│  │  │  3. Check Duplicates (Vertex Search)                   │  ││
│  │  │  4. Score Fundamentals + Technicals                    │  ││
│  │  │  5. Publish Result to Azure Service Bus                │  ││
│  │  │                                                        │  ││
│  │  └───────────────────────────────────────────────────────┘  ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ Vertex AI       │  │ Vertex AI       │  │ Cloud Storage   │  │
│  │ Embeddings API  │  │ Search          │  │ (Model artifacts)│  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Authentication Between Clouds

**Azure → GCP:**
- Use **Workload Identity Federation** (no service account keys)
- Azure Managed Identity → GCP IAM binding

**GCP → Azure:**
- GCP Service Account with Azure AD App Registration
- OAuth 2.0 client credentials flow
- Azure Service Bus SAS tokens

### 4.3 Data Flow: Recommendation Scoring

```
1. Ingestion Service (Azure AKS)
   │
   │  Publishes: recommendation.ingested
   ▼
2. Azure Service Bus (Topic)
   │
   │  Subscription: ai-scoring-sub
   ▼
3. AI Scoring Service (GCP Cloud Run)
   │
   ├──▶ Vertex AI Embeddings (generate vector)
   │
   ├──▶ Vertex AI Search (duplicate check)
   │
   ├──▶ Scoring Logic (fundamental + technical)
   │
   │  Publishes: recommendation.approved / rejected
   ▼
4. Azure Service Bus (Topic)
   │
   ▼
5. Alert Service (Azure AKS)
```

---

## 5. Service-to-Cloud Mapping

| Service | Deployed On | Database | Cache | Queue |
|---------|-------------|----------|-------|-------|
| User Service | Azure AKS | Azure PostgreSQL | Azure Redis | - |
| Subscription Service | Azure AKS | Azure PostgreSQL | Azure Redis | Azure Service Bus |
| Ingestion Service | Azure AKS | Azure PostgreSQL | Azure Redis | Azure Service Bus |
| **AI Scoring Service** | **GCP Cloud Run** | - | - | Azure Service Bus |
| Alert Service | Azure AKS | Azure PostgreSQL | Azure Redis | Azure Service Bus |
| Notification Service | Azure AKS | Azure PostgreSQL | Azure Redis | Azure Service Bus |
| Performance Service | Azure AKS | Azure PostgreSQL | - | - |
| Admin Service | Azure AKS | Azure PostgreSQL | - | - |
| API Gateway (Kong) | Azure AKS | Azure PostgreSQL | Azure Redis | - |

---

## 6. Cost Estimation (Monthly)

### 6.1 Azure Costs

| Resource | Configuration | Est. Monthly Cost |
|----------|---------------|-------------------|
| AKS Cluster | 4-6 nodes (D4s_v3) | $400 - $600 |
| PostgreSQL Flexible | D2ds_v4, 128GB | $150 |
| Redis Cache | Standard C1 | $50 |
| Service Bus | Standard tier | $10 |
| Blob Storage | 100GB, Cool tier | $5 |
| Front Door | Standard tier | $35 |
| Bandwidth | ~500GB egress | $40 |
| **Azure Subtotal** | | **~$700 - $900** |

### 6.2 Google Cloud Costs

| Resource | Configuration | Est. Monthly Cost |
|----------|---------------|-------------------|
| Cloud Run | Scale-to-zero, 2vCPU/2GB | $30 - $80 |
| Vertex AI Embeddings | ~1M requests/month | $20 |
| Vertex AI Search | 10K queries/month | $50 |
| Cloud Storage | 10GB artifacts | $1 |
| Networking | Cross-cloud egress | $20 |
| **GCP Subtotal** | | **~$120 - $170** |

### 6.3 Total Estimated Cost

| Scenario | Monthly Cost |
|----------|--------------|
| **MVP (Low traffic)** | $700 - $900 |
| **Growth (Medium traffic)** | $900 - $1,200 |
| **Scale (High traffic)** | $1,200 - $2,000 |

---

## 7. Security Considerations

### 7.1 Network Security

| Layer | Azure | GCP |
|-------|-------|-----|
| Perimeter | Azure Front Door WAF | Cloud Armor (if needed) |
| VNet/VPC | Azure VNet with NSGs | VPC with firewall rules |
| Service Mesh | Istio on AKS (optional) | - |
| Secrets | Azure Key Vault | GCP Secret Manager |

### 7.2 Identity & Access

| Scenario | Solution |
|----------|----------|
| User Authentication | Azure AD B2C or Auth0 |
| Service-to-Service (Azure) | Managed Identity |
| Service-to-Service (GCP) | Workload Identity Federation |
| Cross-Cloud Auth | OAuth 2.0 + OIDC federation |
| API Keys | Azure Key Vault (rotated) |

### 7.3 Compliance

| Requirement | Implementation |
|-------------|----------------|
| Data Residency | Azure India Central, GCP Mumbai |
| Encryption at Rest | Azure SSE, GCP default encryption |
| Encryption in Transit | TLS 1.3 everywhere |
| Audit Logging | Azure Monitor + GCP Cloud Logging |
| SEBI Disclaimer | Enforced in Alert Service |

---

## 8. Monitoring & Observability

### 8.1 Unified Observability Stack

```
┌─────────────────────────────────────────────────────────────┐
│                    GRAFANA (Unified Dashboard)               │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│   │   Metrics   │  │    Logs     │  │   Traces    │         │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
└──────────┼────────────────┼────────────────┼────────────────┘
           │                │                │
    ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
    │ Prometheus  │  │    Loki     │  │   Tempo     │
    │ (Azure)     │  │  (Azure)    │  │  (Azure)    │
    └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
           │                │                │
    ┌──────┴────────────────┴────────────────┴──────┐
    │              Azure Monitor Agent              │
    │                    +                          │
    │           GCP Cloud Monitoring API            │
    └───────────────────────────────────────────────┘
```

### 8.2 Key Metrics to Monitor

| Service | Critical Metrics | Alert Threshold |
|---------|-----------------|-----------------|
| Ingestion | Source success rate | < 80% in 1 hour |
| AI Scoring | Scoring latency | > 30 seconds |
| AI Scoring | Approval rate | < 10% or > 90% |
| Alert | Delivery latency | > 2 minutes |
| Alert | Push success rate | < 95% |
| API Gateway | Error rate | > 1% |
| API Gateway | P99 latency | > 500ms |

---

## 9. Disaster Recovery

### 9.1 RPO/RTO Targets

| Component | RPO | RTO |
|-----------|-----|-----|
| User Data | 1 hour | 4 hours |
| Subscriptions | 1 hour | 2 hours |
| Alerts | 15 minutes | 1 hour |
| AI Scoring | N/A (stateless) | 5 minutes |

### 9.2 Backup Strategy

| Data | Backup Location | Frequency | Retention |
|------|-----------------|-----------|-----------|
| PostgreSQL | Azure Backup (geo) | Continuous | 7 days |
| Redis | Azure Backup | Daily | 3 days |
| Blob Storage | GRS replication | Real-time | 90 days |
| GCP Artifacts | GCS multi-region | Daily | 30 days |

---

## 10. Implementation Roadmap

### Phase 1: Foundation (Weeks 1-4)
- [ ] Set up Azure subscription and resource groups
- [ ] Create AKS cluster with node pools
- [ ] Deploy PostgreSQL and Redis
- [ ] Set up Azure Service Bus
- [ ] Configure Azure Front Door

### Phase 2: GCP Integration (Weeks 5-6)
- [ ] Set up GCP project
- [ ] Enable Vertex AI APIs
- [ ] Configure Workload Identity Federation
- [ ] Deploy AI Scoring Service to Cloud Run
- [ ] Set up Vertex AI Search data store

### Phase 3: Services Deployment (Weeks 7-10)
- [ ] Deploy core services to AKS
- [ ] Configure cross-cloud messaging
- [ ] Set up CI/CD pipelines
- [ ] Implement monitoring stack
- [ ] Security hardening

### Phase 4: Testing & Launch (Weeks 11-12)
- [ ] Load testing
- [ ] Security audit
- [ ] DR testing
- [ ] Production launch

---

*Document Version: 1.0*
*Architecture: Multi-Cloud (Azure + GCP)*
*Last Updated: November 2024*
