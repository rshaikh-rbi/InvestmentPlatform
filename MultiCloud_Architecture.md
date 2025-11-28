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

### 1.2 Why This Split?

**Azure for Core Infrastructure:**
- Enterprise-grade Kubernetes (AKS) with excellent Windows/.NET support if needed
- Strong compliance certifications for financial services
- Azure Service Bus for reliable messaging
- Integrated identity management (Azure AD)
- Cost-effective PostgreSQL Flexible Server

**Google Cloud for AI/ML:**
- Vertex AI text-embedding-005 is best-in-class for semantic embeddings
- 40% cost savings compared to Azure OpenAI embeddings
- Native vector search with Vertex AI Search
- Cloud Run provides scale-to-zero for AI workloads (cost optimization)
- Gemini Pro available if LLM capabilities needed later

---

## 2. Architecture Diagram

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
│                                                                              │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ │
│  │   User     │ │Subscription│ │ Ingestion  │ │   Alert    │ │   Admin    │ │
│  │  Service   │ │  Service   │ │  Service   │ │  Service   │ │  Service   │ │
│  │  (Python)  │ │  (Python)  │ │  (Python)  │ │  (Python)  │ │  (Python)  │ │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘ └────────────┘ │
│                                                                              │
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
│    LAYER      │    │      BUS        │    │                                 │
│               │    │  (Event Queue)  │    │  ┌─────────────────────────┐   │
│ ┌───────────┐ │    └────────┬────────┘    │  │  AI Scoring Service     │   │
│ │PostgreSQL │ │             │             │  │  (Cloud Run + Vertex)   │   │
│ │ Flexible  │ │             │             │  └─────────────────────────┘   │
│ └───────────┘ │             │             │                                 │
│               │             │             │  ┌─────────────────────────┐   │
│ ┌───────────┐ │             └─────────────┼─▶│  Vertex AI Embeddings   │   │
│ │   Redis   │ │                           │  │  (text-embedding-005)   │   │
│ │  Cache    │ │                           │  └─────────────────────────┘   │
│ └───────────┘ │                           │                                 │
│               │                           │  ┌─────────────────────────┐   │
│ ┌───────────┐ │                           │  │  Vertex AI Search       │   │
│ │   Blob    │ │                           │  │  (Vector Store)         │   │
│ │  Storage  │ │                           │  └─────────────────────────┘   │
│ └───────────┘ │                           │                                 │
└───────────────┘                           └─────────────────────────────────┘
```

---

## 3. Azure Infrastructure Components

### 3.1 Azure Kubernetes Service (AKS)

**Cluster Configuration:**
```yaml
Cluster Name: invest-aks-prod
Region: Central India
Kubernetes Version: 1.28.x

Node Pools:
  - Name: system
    VM Size: Standard_D2s_v3
    Count: 2 (fixed)
    Purpose: System pods, ingress
    
  - Name: workload
    VM Size: Standard_D4s_v3
    Count: 2-10 (autoscale)
    Purpose: Application services
    
  - Name: spot
    VM Size: Standard_D4s_v3
    Count: 0-5 (autoscale)
    Purpose: Batch jobs, cost optimization
    Priority: Spot (preemptible)
```

**Services Deployed on AKS:**

| Service | Min Replicas | Max Replicas | CPU | Memory |
|---------|--------------|--------------|-----|--------|
| User Service | 2 | 5 | 500m | 512Mi |
| Subscription Service | 2 | 4 | 500m | 512Mi |
| Ingestion Service | 2 | 8 | 1000m | 1Gi |
| Alert Service | 3 | 15 | 1000m | 1Gi |
| Notification Service | 2 | 10 | 500m | 512Mi |
| Performance Service | 1 | 3 | 1000m | 1Gi |
| Admin Service | 1 | 2 | 250m | 256Mi |
| Kong Gateway | 2 | 5 | 500m | 512Mi |

### 3.2 Azure Database for PostgreSQL (Flexible Server)

**Configuration:**
- **Tier:** General Purpose
- **Compute:** Standard_D2ds_v4 (2 vCores, 8GB RAM)
- **Storage:** 128 GB with auto-grow
- **High Availability:** Zone-redundant
- **Backup:** 7-day retention, geo-redundant
- **Region:** Central India

**Database Separation:**
```
investdb_users          - User Service
investdb_subscriptions  - Subscription Service  
investdb_ingestion      - Ingestion Service
investdb_alerts         - Alert Service
investdb_performance    - Performance Service
investdb_admin          - Admin Service
investdb_notifications  - Notification Service
```

### 3.3 Azure Cache for Redis

**Configuration:**
- **Tier:** Standard C1 (1GB cache)
- **Region:** Central India

**Use Cases:**
- JWT session storage
- Rate limiting counters (user alert limits)
- API response caching
- Distributed locks

### 3.4 Azure Service Bus

**Configuration:**
- **Tier:** Standard
- **Region:** Central India

**Topics and Subscriptions:**

| Topic | Publisher | Subscribers |
|-------|-----------|-------------|
| `recommendation-ingested` | Ingestion Service | AI Scoring (GCP) |
| `recommendation-approved` | AI Scoring (GCP) | Alert Service |
| `recommendation-rejected` | AI Scoring (GCP) | Admin Service |
| `alert-push` | Alert Service | Notification Service |
| `alert-email` | Alert Service | Notification Service |
| `subscription-events` | Subscription Service | User Service, Alert Service |

### 3.5 Azure Front Door

**Configuration:**
- **Tier:** Standard
- **WAF Policy:** OWASP 3.2 rule set
- **Origin:** AKS Ingress Controller
- **Caching:** Static assets (1 day TTL)
- **SSL:** Managed certificate

---

## 4. Google Cloud Components

### 4.1 AI Scoring Service (Cloud Run)

**Why Cloud Run?**
- Scale to zero during non-market hours (cost savings)
- Native low-latency access to Vertex AI
- GPU support available if needed
- Pay-per-request pricing

**Configuration:**
```yaml
Service Name: ai-scoring-service
Region: asia-south1 (Mumbai)
Memory: 2Gi
CPU: 2
Concurrency: 80
Min Instances: 0
Max Instances: 20
Timeout: 60s
```

**Responsibilities:**
1. Consume messages from Azure Service Bus
2. Generate embeddings using Vertex AI
3. Check duplicates via Vertex AI Search
4. Calculate fundamental + technical scores
5. Publish results back to Azure Service Bus

### 4.2 Vertex AI Embeddings

**Model:** `text-embedding-005`
- Dimensions: 768
- Max tokens: 2048
- Cost: ~$0.0001 per 1K tokens

**Use Cases:**
1. **Recommendation Embeddings** - Convert stock recommendations to vectors
2. **Semantic De-duplication** - Find similar recommendations across sources
3. **News Sentiment** - Embed news headlines for sentiment analysis

**Sample Code:**
```python
from google.cloud import aiplatform
from vertexai.language_models import TextEmbeddingModel

def get_embedding(text: str) -> list[float]:
    model = TextEmbeddingModel.from_pretrained("text-embedding-005")
    embeddings = model.get_embeddings([text])
    return embeddings[0].values

# Example usage
recommendation_text = "Buy INFY target 1800 horizon 30 days"
vector = get_embedding(recommendation_text)
```

### 4.3 Vertex AI Search (Vector Store)

**Configuration:**
- **Data Store Type:** Unstructured (with embeddings)
- **Index:** Approximate Nearest Neighbor (ANN)
- **Dimensions:** 768

**Use Cases:**
1. **Duplicate Detection** - Find if similar recommendation already exists
2. **Semantic Search** - "IT sector recommendations with high confidence"
3. **Related Alerts** - Show similar historical alerts

---

## 5. Cross-Cloud Communication

### 5.1 Data Flow: Recommendation Processing

```
Step 1: Ingestion Service (Azure AKS)
        │
        │  Normalizes data from NSE, MoneyControl, Screener
        │  Publishes to Azure Service Bus
        ▼
Step 2: Azure Service Bus
        │  Topic: recommendation-ingested
        │
        ▼
Step 3: AI Scoring Service (GCP Cloud Run)
        │
        ├──▶ Vertex AI Embeddings
        │    Generate 768-dim vector
        │
        ├──▶ Vertex AI Search  
        │    Check for duplicates (cosine similarity > 0.95)
        │
        ├──▶ Scoring Logic
        │    Fundamental: PE, Debt, Promoter, Earnings (0-50)
        │    Technical: RSI, MA, Volume (0-50)
        │    Composite: 0-100
        │
        │  Publishes to Azure Service Bus
        ▼
Step 4: Azure Service Bus
        │  Topic: recommendation-approved (if score >= 60)
        │  Topic: recommendation-rejected (if score < 60)
        ▼
Step 5: Alert Service (Azure AKS)
        │
        │  Routes to subscribers
        ▼
Step 6: Notification Service (Azure AKS)
        │
        │  Delivers via FCM/Email
        ▼
Step 7: User receives alert
```

### 5.2 Authentication: Azure ↔ GCP

**Azure to GCP (Workload Identity Federation):**
```yaml
# GCP Workload Identity Pool
Pool: azure-identity-pool
Provider: azure-oidc-provider
Issuer: https://sts.windows.net/{azure-tenant-id}/
Attribute Mapping:
  google.subject: assertion.sub
  attribute.tenant: assertion.tid
```

**GCP to Azure (Service Principal):**
```yaml
# Azure AD App Registration
App Name: gcp-ai-scoring
Client ID: {generated}
Client Secret: stored in GCP Secret Manager
Permissions:
  - Azure Service Bus Data Receiver
  - Azure Service Bus Data Sender
```

---

## 6. Service-to-Cloud Mapping Summary

| Service | Cloud | Compute | Database | Cache | Messaging |
|---------|-------|---------|----------|-------|-----------|
| User Service | Azure | AKS | PostgreSQL | Redis | - |
| Subscription Service | Azure | AKS | PostgreSQL | Redis | Service Bus |
| Ingestion Service | Azure | AKS | PostgreSQL | Redis | Service Bus |
| **AI Scoring Service** | **GCP** | **Cloud Run** | - | - | Service Bus |
| Alert Service | Azure | AKS | PostgreSQL | Redis | Service Bus |
| Notification Service | Azure | AKS | PostgreSQL | Redis | Service Bus |
| Performance Service | Azure | AKS | PostgreSQL | - | - |
| Admin Service | Azure | AKS | PostgreSQL | - | - |
| API Gateway | Azure | AKS | PostgreSQL | Redis | - |

---

## 7. Cost Estimation (Monthly)

### 7.1 Azure Costs

| Resource | Configuration | Monthly Cost (USD) |
|----------|---------------|-------------------|
| AKS Cluster | 4-6 nodes D4s_v3 | $400 - $600 |
| PostgreSQL | D2ds_v4, 128GB | $150 |
| Redis Cache | Standard C1 | $50 |
| Service Bus | Standard | $10 |
| Blob Storage | 100GB Cool | $5 |
| Front Door | Standard | $35 |
| Bandwidth | 500GB egress | $40 |
| **Azure Total** | | **$690 - $890** |

### 7.2 Google Cloud Costs

| Resource | Configuration | Monthly Cost (USD) |
|----------|---------------|-------------------|
| Cloud Run | Scale-to-zero | $30 - $80 |
| Vertex AI Embeddings | 1M requests | $20 |
| Vertex AI Search | 10K queries | $50 |
| Cloud Storage | 10GB | $1 |
| Networking | Egress | $20 |
| **GCP Total** | | **$121 - $171** |

### 7.3 Total Monthly Cost

| Phase | Traffic Level | Estimated Cost |
|-------|---------------|----------------|
| MVP | Low | $800 - $1,000 |
| Growth | Medium | $1,000 - $1,300 |
| Scale | High | $1,300 - $2,000 |

---

## 8. Security Architecture

### 8.1 Network Security

| Layer | Azure | GCP |
|-------|-------|-----|
| Edge | Front Door WAF | - |
| Network | VNet + NSG | VPC + Firewall |
| Cluster | AKS Network Policy | - |
| Service | mTLS (optional) | Cloud Run auth |

### 8.2 Secrets Management

| Secret Type | Storage Location |
|-------------|-----------------|
| Database credentials | Azure Key Vault |
| API keys (external) | Azure Key Vault |
| GCP service account | GCP Secret Manager |
| Azure credentials (for GCP) | GCP Secret Manager |

### 8.3 Data Residency

| Data Type | Location | Compliance |
|-----------|----------|------------|
| User PII | Azure Central India | India data residency |
| Financial data | Azure Central India | RBI guidelines |
| AI models | GCP Mumbai | Processed, not stored |
| Embeddings | GCP Mumbai | Derived data only |

---

## 9. Monitoring Strategy

### 9.1 Unified Dashboard (Grafana)

**Data Sources:**
- Azure Monitor (metrics, logs)
- GCP Cloud Monitoring (AI service metrics)
- Prometheus (AKS custom metrics)

**Key Dashboards:**
1. **Platform Health** - All services status
2. **Ingestion Pipeline** - Source health, data freshness
3. **AI Scoring** - Latency, approval rate, costs
4. **Alert Delivery** - Delivery success, latency
5. **Business Metrics** - Active users, alerts sent, conversions

### 9.2 Alerting

| Metric | Threshold | Severity | Action |
|--------|-----------|----------|--------|
| Ingestion failure rate | > 20% for 1 hour | Critical | Page on-call |
| AI scoring latency | > 30 seconds | High | Notify team |
| Alert delivery latency | > 2 minutes | Critical | Page on-call |
| API error rate | > 1% | High | Notify team |
| GCP spend anomaly | > 50% daily increase | Medium | Review |

---

## 10. Implementation Phases

### Phase 1: Azure Foundation (Weeks 1-3)
- Set up Azure subscription and resource groups
- Deploy AKS cluster with node pools
- Configure PostgreSQL, Redis, Service Bus
- Set up Azure Front Door
- Deploy Kong API Gateway

### Phase 2: Core Services (Weeks 4-6)
- Deploy User, Subscription, Admin services
- Deploy Ingestion Service (mock AI for now)
- Deploy Alert, Notification services
- Deploy Performance Service
- End-to-end testing with mock data

### Phase 3: GCP Integration (Weeks 7-8)
- Set up GCP project and enable APIs
- Configure Workload Identity Federation
- Deploy AI Scoring Service to Cloud Run
- Set up Vertex AI Embeddings
- Configure Vertex AI Search
- Cross-cloud messaging testing

### Phase 4: Production Readiness (Weeks 9-10)
- Security hardening
- Monitoring and alerting setup
- Load testing
- DR testing
- Documentation

### Phase 5: Launch (Weeks 11-12)
- Staged rollout
- Production monitoring
- Performance tuning
- Cost optimization

---

*Document Version: 1.0*
*Architecture: Multi-Cloud (Azure + GCP)*
*Last Updated: November 2024*
