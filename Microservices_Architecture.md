# Microservices Architecture - AI Investment Alert Platform

## 1. Executive Summary

This document outlines the microservices architecture for the AI-powered investment alert platform. Services have been identified using **Domain-Driven Design (DDD)** principles, analyzing bounded contexts from functional requirements, and ensuring each service has a single responsibility with clear ownership.

---

## 2. Service Identification Methodology

### 2.1 Approach Used

| Technique | Application |
|-----------|-------------|
| **Domain-Driven Design** | Identified bounded contexts from business capabilities |
| **Single Responsibility Principle** | Each service owns one business capability |
| **Data Ownership** | Each service owns its data; no shared databases |
| **Sequence Diagram Analysis** | Mapped actors and interactions to service boundaries |
| **Scalability Requirements** | Separated high-load components (ingestion, alerts) |
| **Team Autonomy** | Services sized for 2-pizza team ownership |

### 2.2 Bounded Contexts Identified

From the functional requirements (FR1-FR10), we identified **8 bounded contexts**:

| Context | Functional Requirements | Core Capability |
|---------|------------------------|-----------------|
| User Identity | FR1 | Registration, Auth, KYC |
| Subscription & Billing | FR2 | Plans, Payments, Renewals |
| Data Ingestion | FR3, FR4 | Source connectivity, Normalization |
| AI Scoring | FR5 | Validation, Scoring, Filtering |
| Alert Delivery | FR6 | Routing, Rate limiting, Delivery |
| Performance Tracking | FR7 | Price tracking, Hit/Miss stats |
| Content Access | FR8 | Historical view, Gating |
| Administration | FR9, FR10 | Config, Compliance, User mgmt |

---

## 3. Microservices Catalog

### 3.1 Service Overview

| # | Service Name | Domain | Primary Responsibility | FR Mapping |
|---|--------------|--------|------------------------|------------|
| 1 | User Service | Identity | Registration, authentication, profile, KYC | FR1.1-FR1.4, FR10.2 |
| 2 | Subscription Service | Billing | Plans, payments, renewals, grace period | FR2.1-FR2.4 |
| 3 | Ingestion Service | Data | Source connectivity, normalization, de-duplication | FR3.1-FR3.4, FR4.1-FR4.3 |
| 4 | AI Scoring Service | Intelligence | Fundamental & technical analysis, scoring | FR5.1-FR5.5 |
| 5 | Alert Service | Notification | Alert routing, rate limiting, delivery orchestration | FR6.1-FR6.4, FR8.1-FR8.2, FR10.1 |
| 6 | Notification Service | Delivery | Push, email, SMS channel management | FR6.1, FR6.2 |
| 7 | Performance Service | Analytics | Price tracking, hit/miss calculation, statistics | FR7.1-FR7.4 |
| 8 | Admin Service | Operations | Source config, AI tuning, user management | FR9.1-FR9.4, FR5.5 |
| 9 | API Gateway | Infrastructure | Routing, auth, rate limiting, API versioning | Cross-cutting |

---

## 4. Detailed Service Specifications

### 4.1 User Service

**Bounded Context:** User Identity & Access

**Responsibility:**
- User registration with OTP verification
- KYC-light capture (PAN for Indian users)
- Authentication (JWT issuance)
- Profile management
- Role management (Guest → Free → Subscriber)
- Terms acceptance tracking

**Why Separate Service:**
- Identity is a cross-cutting concern used by all services
- Security-sensitive; needs isolated audit logging
- Compliance requirements (KYC) need dedicated ownership
- Scales independently based on registration/login traffic

**Data Owned:**
- `users` - Profile, credentials, KYC identifiers
- `user_roles` - Role assignments and history
- `terms_acceptance` - Consent records with timestamps
- `sessions` - Active JWT tokens

**Key APIs:**
- POST /api/v1/users/register
- POST /api/v1/users/verify-otp
- POST /api/v1/users/login
- GET /api/v1/users/profile
- PUT /api/v1/users/profile

**Dependencies:** Email/SMS provider (for OTP)

---

### 4.2 Subscription Service

**Bounded Context:** Billing & Monetization

**Responsibility:**
- Plan management (Free, Basic, Premium)
- Payment gateway integration (Razorpay/Stripe)
- Subscription lifecycle (create, renew, cancel)
- Grace period handling (48 hours)
- Renewal reminders (3-day advance)
- Invoice generation

**Why Separate Service:**
- Financial transactions need isolation for audit
- Payment gateway webhooks need dedicated handling
- Different scaling pattern (bursty during renewals)
- PCI-DSS compliance boundary

**Data Owned:**
- `plans` - Plan definitions and pricing
- `subscriptions` - Active/expired subscriptions
- `payments` - Transaction records
- `invoices` - Billing documents

**Key APIs:**
- GET /api/v1/plans
- POST /api/v1/subscriptions/create-order
- POST /api/v1/subscriptions/webhook
- GET /api/v1/subscriptions/status
- POST /api/v1/subscriptions/cancel

**Dependencies:** Payment Gateway (Razorpay/Stripe), User Service

---

### 4.3 Ingestion Service

**Bounded Context:** Data Acquisition & Normalization

**Responsibility:**
- Connect to multiple data sources (NSE, MoneyControl, Screener.in)
- Schedule-based polling (configurable per source)
- Raw data logging (90-day retention)
- Normalization to unified stock schema
- De-duplication within 24-hour window
- Sector and market cap tagging
- Stale data detection and flagging

**Why Separate Service:**
- High I/O operations; needs horizontal scaling
- Different failure modes than user-facing services
- Source-specific adapters can be deployed independently
- Scheduled jobs isolated from request-response services

**Data Owned:**
- `data_sources` - Source configurations
- `raw_ingestion_logs` - Raw API responses (90-day retention)
- `normalized_recommendations` - Cleaned, unified data
- `ingestion_health` - Success/failure metrics per source

**Key APIs:**
- POST /api/v1/ingestion/trigger (Manual)
- GET /api/v1/ingestion/health
- GET /api/v1/ingestion/sources

**Events Published:**
- `recommendation.ingested` → AI Scoring Service

**Dependencies:** External Data Source APIs, Message Queue

---

### 4.4 AI Scoring Service

**Bounded Context:** Intelligent Validation

**Responsibility:**
- Fundamental analysis scoring (PE, debt/equity, promoter holding, earnings)
- Technical analysis scoring (RSI, moving averages, volume)
- Composite confidence scoring (0-100 scale)
- Threshold-based filtering (default: 60)
- Rejection logging with detailed reasons
- Support admin-configurable weights

**Why Separate Service:**
- Compute-intensive; may need specialized instances
- ML model versioning independent of other services
- A/B testing of scoring algorithms
- Scales based on ingestion volume, not user traffic

**Data Owned:**
- `scoring_config` - Weights and thresholds
- `scoring_results` - Score breakdown per recommendation
- `rejection_log` - Rejected items with reasons

**Key APIs:**
- POST /api/v1/scoring/score (Internal)
- GET /api/v1/scoring/config
- PUT /api/v1/scoring/config (Admin only)
- GET /api/v1/scoring/rejected

**Events Consumed:**
- `recommendation.ingested` ← Ingestion Service

**Events Published:**
- `recommendation.approved` → Alert Service
- `recommendation.rejected` → Admin Service (dashboard)

---

### 4.5 Alert Service

**Bounded Context:** Alert Orchestration

**Responsibility:**
- Receive approved recommendations from AI Scoring
- Determine eligible subscribers per alert
- Apply rate limits (5/day Basic, 15/day Premium)
- Attach SEBI disclaimer to all alerts
- Route to appropriate notification channels
- Maintain in-app alert center
- Track alert delivery status

**Why Separate Service:**
- High throughput during market hours (9:00-15:30 IST)
- Rate limiting logic needs dedicated state management
- Different SLA requirements (< 2 min delivery)
- Decoupled from scoring for reliability

**Data Owned:**
- `alerts` - Alert records with delivery status
- `user_alert_counts` - Daily rate limit tracking
- `alert_preferences` - User notification preferences
- `in_app_alerts` - In-app alert center data

**Key APIs:**
- GET /api/v1/alerts (User's alerts)
- GET /api/v1/alerts/historical (Free users - >7 days)
- PUT /api/v1/alerts/{id}/read
- GET /api/v1/alerts/preferences
- PUT /api/v1/alerts/preferences

**Events Consumed:**
- `recommendation.approved` ← AI Scoring Service

**Events Published:**
- `alert.send.push` → Notification Service
- `alert.send.email` → Notification Service

---

### 4.6 Notification Service

**Bounded Context:** Multi-Channel Delivery

**Responsibility:**
- Push notification delivery (FCM for Android, APNs for iOS)
- Email delivery (transactional alerts + digest compilation)
- SMS delivery (OTP, critical alerts if configured)
- Template management for all channels
- Delivery tracking with retry logic
- Digest aggregation (daily/weekly batching)

**Why Separate Service:**
- Channel-specific SDKs and vendor configurations
- Retry logic independent of business logic
- Vendor switching without affecting other services
- Different scaling characteristics per channel

**Data Owned:**
- `notification_templates` - Message templates per channel
- `delivery_logs` - Send attempts, status, retries
- `device_tokens` - FCM/APNs tokens per user
- `digest_queue` - Pending digest items

**Key APIs:**
- POST /api/v1/notifications/push (Internal)
- POST /api/v1/notifications/email (Internal)
- POST /api/v1/notifications/sms (Internal)
- GET /api/v1/notifications/status/{id}

**Events Consumed:**
- `alert.send.push` ← Alert Service
- `alert.send.email` ← Alert Service
- `otp.send` ← User Service

**Dependencies:** FCM, APNs, SendGrid/AWS SES, SMS Provider

---

### 4.7 Performance Service

**Bounded Context:** Analytics & Tracking

**Responsibility:**
- Capture price snapshot at alert time
- Track prices at defined horizons (7d, 30d, 90d)
- Calculate return percentage
- Determine Hit/Miss status against target
- Aggregate success rates by source, sector, time period
- Support user personal tracking ("I acted on this")

**Why Separate Service:**
- Batch processing pattern (EOD scheduled jobs)
- Heavy database read/write operations
- Analytics queries isolated from transactional workloads
- Can leverage read-optimized database replicas

**Data Owned:**
- `price_snapshots` - Historical price records
- `performance_results` - Hit/miss per alert
- `aggregated_stats` - Pre-computed statistics
- `user_actions` - Personal "acted on" tracking

**Key APIs:**
- GET /api/v1/performance/stats
- GET /api/v1/performance/by-source
- GET /api/v1/performance/by-sector
- GET /api/v1/performance/by-period
- POST /api/v1/performance/track/{alertId}

**Dependencies:** Price Feed API (for current prices)

---

### 4.8 Admin Service

**Bounded Context:** Platform Operations

**Responsibility:**
- Data source configuration (add/edit/disable without deployment)
- AI scoring weight and threshold management
- Manual override of rejected recommendations
- User management (view, extend/revoke access)
- Ingestion health dashboard
- Complete audit trail for all admin actions

**Why Separate Service:**
- Different access control model (admin-only)
- Strict audit logging requirements
- Low traffic; can run on smaller instances
- Configuration changes isolated from runtime services

**Data Owned:**
- `admin_users` - Admin credentials and permissions
- `audit_logs` - Complete trail of all admin actions
- `system_config` - Platform-wide configurations

**Key APIs:**
- GET /api/v1/admin/sources
- POST /api/v1/admin/sources
- PUT /api/v1/admin/sources/{id}
- DELETE /api/v1/admin/sources/{id}
- GET /api/v1/admin/ai-config
- PUT /api/v1/admin/ai-config
- GET /api/v1/admin/rejected
- POST /api/v1/admin/override/{id}
- GET /api/v1/admin/users
- PUT /api/v1/admin/users/{id}/access
- GET /api/v1/admin/health/ingestion

---

### 4.9 API Gateway

**Bounded Context:** Infrastructure / Cross-Cutting

**Responsibility:**
- Single entry point for all client requests
- Request routing to appropriate services
- JWT validation and authorization
- API-level rate limiting
- Request/response logging
- API versioning support
- CORS handling
- SSL/TLS termination

**Why Separate:**
- Centralized security enforcement
- Simplified client integration (single endpoint)
- Enables blue-green deployments
- Cross-cutting concerns in one place

**Technology Options:** Kong, AWS API Gateway, Nginx + Lua

---

## 5. Service Communication Patterns

### 5.1 Synchronous Communication (REST)

| From Service | To Service | Purpose | Protocol |
|--------------|------------|---------|----------|
| API Gateway | All Services | Request routing | REST/HTTP |
| Alert Service | User Service | Get subscriber list | REST |
| Alert Service | Subscription Service | Verify active subscription | REST |
| Performance Service | Alert Service | Get alert details | REST |
| Admin Service | All Services | Configuration reads/writes | REST |

### 5.2 Asynchronous Communication (Events)

| Event Name | Publisher | Consumer(s) | Purpose |
|------------|-----------|-------------|---------|
| `recommendation.ingested` | Ingestion Service | AI Scoring Service | Trigger scoring |
| `recommendation.approved` | AI Scoring Service | Alert Service | Trigger alert delivery |
| `recommendation.rejected` | AI Scoring Service | Admin Service | Dashboard update |
| `alert.send.push` | Alert Service | Notification Service | Push delivery |
| `alert.send.email` | Alert Service | Notification Service | Email delivery |
| `subscription.created` | Subscription Service | User Service | Update user role |
| `subscription.expired` | Subscription Service | User Service, Alert Service | Downgrade access |
| `subscription.renewed` | Subscription Service | User Service | Extend access |

---

## 6. Data Ownership Matrix

| Service | Owned Tables | Shared Via |
|---------|--------------|------------|
| User Service | users, user_roles, sessions, terms_acceptance | API / Events |
| Subscription Service | plans, subscriptions, payments, invoices | API / Events |
| Ingestion Service | data_sources, raw_logs, normalized_recommendations | Events |
| AI Scoring Service | scoring_config, scoring_results, rejection_log | Events |
| Alert Service | alerts, alert_preferences, rate_limits, in_app_alerts | API |
| Notification Service | templates, delivery_logs, device_tokens, digest_queue | Internal |
| Performance Service | price_snapshots, performance_results, stats, user_actions | API |
| Admin Service | admin_users, audit_logs, system_config | API |

**Principle:** No direct database access across services. All data sharing via APIs or events.

---

## 7. Deployment Strategy (Startup-Friendly)

### Phase 1: MVP (0-6 months)
- Deploy as **modular monolith** with clear service boundaries
- Single database with schema separation
- Services as modules, not separate deployments
- Focus on validating product-market fit

### Phase 2: Early Traction (6-12 months)
- Extract **Ingestion Service** (high I/O, independent scaling)
- Extract **Alert Service** (different SLA, market-hours scaling)
- Introduce message queue for async communication
- Remaining services stay as monolith

### Phase 3: Scale (12+ months)
- Full microservices deployment
- Dedicated databases per service
- Kubernetes orchestration
- Advanced observability stack

---

## 8. Technology Stack Recommendations

| Component | Recommendation | Rationale |
|-----------|----------------|-----------|
| Language | Python (FastAPI) | Your expertise, async support |
| API Gateway | Kong / AWS API Gateway | Managed, plugins available |
| Message Queue | RabbitMQ / AWS SQS | Simple, reliable, cost-effective |
| Database | PostgreSQL | ACID compliance, JSON support |
| Cache | Redis | Session storage, rate limiting |
| Search | Elasticsearch (later) | Alert search, analytics |
| Container | Docker | Portability, consistency |
| Orchestration | Kubernetes (EKS/GKE) | Auto-scaling, self-healing |
| Monitoring | Prometheus + Grafana | Free, comprehensive |
| Logging | ELK Stack / CloudWatch | Centralized logs |
| CI/CD | GitHub Actions | Cost-effective, integrated |

---

## 9. Cost Optimization Notes

1. **Start with managed services** - Avoid infrastructure overhead
2. **Use serverless where possible** - Lambda for scheduled jobs
3. **Right-size instances** - Admin Service needs minimal resources
4. **Auto-scaling based on market hours** - Scale up 8:30-16:00 IST only
5. **Reserved instances** - After traffic patterns are established
6. **Single region initially** - Multi-region only when needed

---

*Document Version: 1.0*
*Last Updated: November 2024*
