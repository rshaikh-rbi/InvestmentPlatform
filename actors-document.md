# Actors - AI Investment Alert Platform

## 1. Primary Actors (Direct Users)

| Actor | Description | Key Interactions |
|-------|-------------|------------------|
| **Guest** | Unauthenticated visitor exploring the platform | View landing page, see platform features, register |
| **Free User** | Registered but non-subscribed user | View past recommendations (>7 days old), see performance stats, upgrade to paid plan |
| **Subscriber (Basic)** | Paid user on basic tier | Receive up to 5 alerts/day, track performance, manage preferences |
| **Subscriber (Premium)** | Paid user on premium tier | Receive up to 15 alerts/day, full historical access, priority alerts |
| **Admin** | Platform administrator | Manage data sources, configure AI rules, user management, view dashboards |

---

## 2. Secondary Actors (System/External)

| Actor | Description | Key Interactions |
|-------|-------------|------------------|
| **Data Source API** | External financial data providers (NSE, MoneyControl, screener.in) | Provide stock data, recommendations, price feeds |
| **Payment Gateway** | Razorpay/Stripe | Process subscriptions, handle renewals, issue refunds |
| **Notification Service** | Push notification provider (FCM/APNs) | Deliver real-time alerts to mobile devices |
| **Email Service** | Transactional email provider (SendGrid/SES) | Send OTP, digests, renewal reminders |
| **AI Scoring Engine** | Internal validation system | Score recommendations, filter low-confidence picks |
| **Scheduler/Cron** | Background job processor | Price snapshots, data refresh, performance calculation |

---

## 3. Actor-Role Matrix

| Actor | Register | Subscribe | View Alerts | Receive Push | Configure AI | Manage Users |
|-------|----------|-----------|-------------|--------------|--------------|--------------|
| Guest | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| Free User | — | ✓ | Limited | ✗ | ✗ | ✗ |
| Subscriber (Basic) | — | ✓ | ✓ | ✓ (5/day) | ✗ | ✗ |
| Subscriber (Premium) | — | ✓ | ✓ | ✓ (15/day) | ✗ | ✗ |
| Admin | — | — | ✓ | ✓ | ✓ | ✓ |

---

## 4. Actor Descriptions

### 4.1 Guest
- **Goal:** Explore platform value proposition before committing
- **Entry Point:** Landing page, marketing campaigns
- **Exit Point:** Registration or bounce

### 4.2 Free User
- **Goal:** Evaluate platform quality using historical data
- **Permissions:** Read-only access to aged recommendations
- **Conversion Trigger:** Seeing consistent hit-rate in past alerts

### 4.3 Subscriber (Basic)
- **Goal:** Receive validated stock alerts within budget
- **Limits:** 5 alerts/day, standard sectors
- **Value:** Filtered, AI-validated recommendations

### 4.4 Subscriber (Premium)
- **Goal:** Maximize coverage and speed of alerts
- **Limits:** 15 alerts/day, all sectors, priority delivery
- **Value:** First access to high-confidence opportunities

### 4.5 Admin
- **Goal:** Ensure platform reliability and quality
- **Responsibilities:** 
  - Monitor data ingestion health
  - Tune AI scoring parameters
  - Handle user escalations
  - Compliance oversight

---

## 5. External System Actors

### 5.1 Data Source API
- **Examples:** NSE India API, MoneyControl, Screener.in, TradingView
- **Interaction:** Scheduled polling or webhook-based ingestion
- **Failure Handling:** Stale data flagging, admin alerts

### 5.2 Payment Gateway
- **Examples:** Razorpay, Stripe
- **Interaction:** Subscription creation, renewal, cancellation
- **Webhook Events:** Payment success, failure, refund

### 5.3 Notification Service
- **Examples:** Firebase Cloud Messaging (FCM), Apple Push Notification (APNs)
- **Interaction:** Real-time alert delivery to mobile apps
- **SLA:** Delivery within 2 minutes of AI approval

### 5.4 Email Service
- **Examples:** SendGrid, AWS SES
- **Interaction:** Transactional emails (OTP, digests, reminders)
- **Types:** Instant alerts, daily digest, weekly summary

---

*Document Version: 1.0*  
*Last Updated: November 2024*
