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

```
┌─────────────────────────────────────────────────────────────────┐
│                    BOUNDED CONTEXTS                              │
├─────────────────┬─────────────────┬─────────────────────────────┤
│ User Identity   │ Subscription    │ Data Ingestion              │
│ (FR1)           │ & Billing (FR2) │ (FR3, FR4)                  │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ AI Scoring      │ Alert Delivery  │ Performance                 │
│ (FR5)           │ (FR6)           │ Tracking (FR7)              │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ Content Access  │ Administration  │                             │
│ (FR8)           │ (FR9, FR10)     │                             │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

---

## 3. Microservices Catalog

### 3.1 Service Overview

| # | Service Name | Domain | Primary Responsibility |
|---|--------------|--------|------------------------|
| 1 | **User Service** | Identity | Registration, authentication, profile, KYC |
| 2 | **Subscription Service** | Billing | Plans, payments, renewals, grace period |
| 3 | **Ingestion Service** | Data | Source connectivity, normalization, de-duplication |
| 4 | **AI Scoring Service** | Intelligence | Fundamental & technical analysis, scoring |
| 5 | **Alert Service** | Notification | Alert routing, rate limiting, delivery orchestration |
| 6 | **Notification Service** | Delivery | Push, email, SMS channel management |
| 7 | **Performance Service** | Analytics | Price tracking, hit/miss calculation, statistics |
| 8 | **Admin Service** | Operations | Source config, AI tuning, user management |
| 9 | **API Gateway** | Infrastructure | Routing, auth, rate limiting, API versioning |

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
```
POST /api/v1/users/register
POST /api/v1/users/verify-otp
POST /api/v1/users/login
GET  /api/v1/users/profile
PUT  /api/v1/users/profile
```

**Dependencies:** Email/SMS provider (for OTP)

**FR Mapping:** FR1.1, FR1.2, FR1.3, FR1.4, FR10.2

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
```
GET  /api/v1/plans
POST /api/v1/subscriptions/create-order
POST /api/v1/subscriptions/webhook (Payment Gateway)
GET  /api/v1/subscriptions/status
POST /api/v1/subscriptions/cancel
```

**Dependencies:** Payment Gateway, User Service

**FR Mapping:** FR2.1, FR2.2, FR2.3, FR2.4

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
- `raw_ingestion_logs` - Raw API responses
- `normalized_recommendations` - Cleaned data
- `ingestion_health` - Success/failure metrics

**Key APIs:**
```
POST /api/v1/ingestion/trigger (Manual trigger)
GET  /api/v1/ingestion/health
GET  /api/v1/ingestion/sources
```

**Events Published:**
- `recommendation.ingested` → AI Scoring Service

**Dependencies:** External Data APIs, Message Queue

**FR Mapping:** FR3.1, FR3.2, FR3.3, FR3.4, FR4.1, FR4.2, FR4.3

---

### 4.4 AI Scoring Service

**Bounded Context:** Intelligent Validation

**Responsibility:**
- Fundamental analysis (PE, debt/equity, promoter holding, earnings)
- Technical analysis (RSI, moving averages, volume)
- Composite confidence scoring (0-100)
- Threshold-based filtering (default: 60)
- Rejection logging with reasons
- Support admin weight configuration

**Why Separate Service:**
- Compute-intensive; may need GPU/specialized instances
- ML model versioning independent of other services
- A/B testing of scoring algorithms
- Can be scaled based on ingestion volume

**Data Owned:**
- `scoring_config` - Weights and thresholds
- `scoring_results` - Score breakdown per recommendation
- `rejection_log` - Rejected items with reasons

**Key APIs:**
```
POST /api/v1/scoring/score (Internal)
GET  /api/v1/scoring/config
PUT  /api/v1/scoring/config (Admin)
GET  /api/v1/scoring/rejected
```

**Events Consumed:**
- `recommendation.ingested` ← Ingestion Service

**Events Published:**
- `recommendation.approved` → Alert Service
- `recommendation.rejected` → Admin Dashboard

**FR Mapping:** FR5.1, FR5.2, FR5.3, FR5.4, FR5.5

---

### 4.5 Alert Service

**Bounded Context:** Alert Orchestration

**Responsibility:**
- Receive approved recommendations
- Determine eligible subscribers per alert
- Apply rate limits (5/day Basic, 15/day Premium)
- Attach SEBI disclaimer to all alerts
- Route to appropriate notification channel
- Store alerts in in-app center
- Track alert delivery status

**Why Separate Service:**
- High throughput during market hours
- Rate limiting logic isolated
- Different SLA (< 2 min delivery)
- Decoupled from scoring for reliability

**Data Owned:**
- `alerts` - Alert records with status
- `user_alert_counts` - Daily rate limit tracking
- `alert_preferences` - User notification preferences
- `in_app_alerts` - In-app alert center

**Key APIs:**
```
GET  /api/v1/alerts (User's alerts)
GET  /api/v1/alerts/historical (Free users)
PUT  /api/v1/alerts/{id}/read
GET  /api/v1/alerts/preferences
PUT  /api/v1/alerts/preferences
```

**Events Consumed:**
- `recommendation.approved` ← AI Scoring Service

**Events Published:**
- `alert.send.push` → Notification Service
- `alert.send.email` → Notification Service

**FR Mapping:** FR6.1, FR6.2, FR6.3, FR6.4, FR8.1, FR8.2, FR10.1

---

### 4.6 Notification Service

**Bounded Context:** Multi-Channel Delivery

**Responsibility:**
- Push notification delivery (FCM/APNs)
- Email delivery (transactional + digest)
- SMS delivery (OTP, critical alerts)
- Template management
- Delivery tracking and retry
- Digest aggregation (daily/weekly)

**Why Separate Service:**
- Channel-specific SDKs and configurations
- Retry logic independent of alert logic
- Vendor switching without affecting other services
- Different scaling for each channel

**Data Owned:**
- `notification_templates` - Message templates
- `delivery_logs` - Send attempts and status
- `device_tokens` - Push notification tokens
- `digest_queue` - Pending digest items

**Key APIs:**
```
POST /api/v1/notifications/push
POST /api/v1/notifications/email
POST /api/v1/notifications/sms
GET  /api/v1/notifications/status/{id}
```

**Events Consumed:**
- `alert.send.push` ← Alert Service
- `alert.send.email` ← Alert Service

**Dependencies:** FCM, APNs, SendGrid/SES, SMS Provider

**FR Mapping:** FR6.1, FR6.2 (delivery aspect)

---

### 4.7 Performance Service

**Bounded Context:** Analytics & Tracking

**Responsibility:**
- Price snapshot at alert time
- Track prices at horizons (7d, 30d, 90d)
- Calculate return percentage
- Determine Hit/Miss status
- Aggregate success rates by source, sector, period
- User personal tracking ("I acted on this")

**Why Separate Service:**
- Batch processing (EOD jobs)
- Heavy database operations
- Analytics queries isolated from transactional
- Can use separate read-optimized database

**Data Owned:**
- `price_snapshots` - Historical price records
- `performance_results` - Hit/miss per alert
- `aggregated_stats` - Pre-computed statistics
- `user_actions` - Personal tracking flags

**Key APIs:**
```
GET  /api/v1/performance/stats
GET  /api/v1/performance/by-source
GET  /api/v1/performance/by-sector
POST /api/v1/performance/track/{alertId}
```

**Dependencies:** Price Feed API

**FR Mapping:** FR7.1, FR7.2, FR7.3, FR7.4

---

### 4.8 Admin Service

**Bounded Context:** Platform Operations

**Responsibility:**
- Data source configuration (CRUD)
- AI scoring weight management
- Manual override of rejected recommendations
- User management (extend/revoke access)
- Ingestion health dashboard
- Audit trail for all admin actions

**Why Separate Service:**
- Different access control (admin-only)
- Audit logging requirements
- Low traffic; can be smaller instance
- Configuration changes isolated

**Data Owned:**
- `admin_users` - Admin credentials
- `audit_logs` - All admin actions
- `system_config` - Platform configurations

**Key APIs:**
```
GET  /api/v1/admin/sources
POST /api/v1/admin/sources
PUT  /api/v1/admin/sources/{id}
GET  /api/v1/admin/ai-config
PUT  /api/v1/admin/ai-config
GET  /api/v1/admin/rejected
POST /api/v1/admin/override/{id}
GET  /api/v1/admin/users
PUT  /api/v1/admin/users/{id}/access
GET  /api/v1/admin/health
```

**FR Mapping:** FR9.1, FR9.2, FR9.3, FR9.4, FR5.5

---

### 4.9 API Gateway

**Bounded Context:** Infrastructure / Cross-Cutting

**Responsibility:**
- Request routing to services
- JWT validation
- Rate limiting (API level)
- Request/response logging
- API versioning
- CORS handling
- SSL termination

**Why Separate:**
- Single entry point for security
- Centralized logging
- Simplified client integration
- Blue-green deployment support

**Technology:** Kong / AWS API Gateway / Nginx

---

## 5. Service Communication

### 5.1 Synchronous (REST/gRPC)

| From | To | Purpose |
|------|----|---------|
| API Gateway | All Services | Request routing |
| Alert Service | User Service | Get subscriber list |
| Alert Service | Subscription Service | Verify active subscription |
| Admin Service | All Services | Configuration updates |

### 5.2 Asynchronous (Message Queue)

| Event | Publisher | Consumer(s) |
|-------|-----------|-------------|
| `recommendation.ingested` | Ingestion Service | AI Scoring Service |
| `recommendation.approved` | AI Scoring Service | Alert Service |
| `recommendation.rejected` | AI Scoring Service | Admin Service |
| `alert.send.push` | Alert Service | Notification Service |
| `alert.send.email` | Alert Service | Notification Service |
| `subscription.created` | Subscription Service | User Service |
| `subscription.expired` | Subscription Service | User Service, Alert Service |

---

## 6. Data Ownership Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    DATA OWNERSHIP MAP                        │
├─────────────────┬───────────────────────────────────────────┤
│ User Service    │ users, roles, sessions, terms_acceptance  │
├─────────────────┼───────────────────────────────────────────┤
│ Subscription    │ plans, subscriptions, payments, invoices  │
├─────────────────┼───────────────────────────────────────────┤
│ Ingestion       │ sources, raw_logs, normalized_recs        │
├─────────────────┼───────────────────────────────────────────┤
│ AI Scoring      │ scoring_config, results, rejection_log    │
├─────────────────┼───────────────────────────────────────────┤
│ Alert Service   │ alerts, preferences, rate_limits          │
├─────────────────┼───────────────────────────────────────────┤
│ Notification    │ templates, delivery_logs, device_tokens   │
├─────────────────┼───────────────────────────────────────────┤
│ Performance     │ snapshots, hit_miss, stats, user_actions  │
├─────────────────┼───────────────────────────────────────────┤
│ Admin Service   │ admin_users, audit_logs, system_config    │
└─────────────────┴───────────────────────────────────────────┘
```

---

## 7. Deployment Considerations

### 7.1 Scaling Strategy

| Service | Scaling Trigger | Strategy |
|---------|-----------------|----------|
| Ingestion | Data source count | Horizontal (workers) |
| AI Scoring | Queue depth | Horizontal + Vertical |
| Alert Service | Market hours load | Time-based auto-scale |
| Notification | Delivery queue | Horizontal |
| Others | Request rate | Horizontal |

### 7.2 Startup-Friendly Deployment

**Phase 1 (MVP):** Deploy as modular monolith with clear boundaries
**Phase 2 (Traction):** Extract high-load services (Ingestion, Alert)
**Phase 3 (Scale):** Full microservices with dedicated databases

---

## 8. Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|----------------|-----------|
| API Gateway | Kong / AWS API Gateway | Managed, cost-effective |
| Message Queue | RabbitMQ / AWS SQS | Simple, reliable |
| Database | PostgreSQL | ACID, JSON support |
| Cache | Redis | Session, rate limiting |
| Container | Docker + Kubernetes | Portability |
| Monitoring | Prometheus + Grafana | Free tier available |

---

*Document Version: 1.0*
*Last Updated: November 2024*
