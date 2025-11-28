# API Specification Document
## AI Investment Alert Platform

**Version:** 1.0  
**Last Updated:** November 2024  
**Status:** Draft for Review

---

## Table of Contents
1. [Overview](#1-overview)
2. [Security Architecture](#2-security-architecture)
3. [Authentication & Authorization](#3-authentication--authorization)
4. [Rate Limiting Strategy](#4-rate-limiting-strategy)
5. [PII Data Handling](#5-pii-data-handling)
6. [API Standards & Conventions](#6-api-standards--conventions)
7. [Service API Specifications](#7-service-api-specifications)
8. [Error Handling](#8-error-handling)
9. [Audit & Compliance](#9-audit--compliance)
10. [API Versioning Strategy](#10-api-versioning-strategy)

---

## 1. Overview

### 1.1 Purpose
This document defines the API specifications for the AI Investment Alert Platform, ensuring consistency, security, and compliance across all microservices.

### 1.2 API Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                   │
│              (Mobile App / Admin Portal / Partners)              │
└─────────────────────────┬───────────────────────────────────────┘
                          │ HTTPS (TLS 1.3)
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY (Kong)                            │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐  │
│  │ Rate Limit  │ │ JWT Valid.  │ │ WAF Rules   │ │ Logging   │  │
│  └─────────────┘ └─────────────┘ └─────────────┘ └───────────┘  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
       ┌──────────────────┼──────────────────┐
       ▼                  ▼                  ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│ User Service│   │Alert Service│   │ Admin Svc   │
└─────────────┘   └─────────────┘   └─────────────┘
```

### 1.3 Base URLs

| Environment | Base URL |
|-------------|----------|
| Production | `https://api.investalert.in/v1` |
| Staging | `https://api-staging.investalert.in/v1` |
| Development | `https://api-dev.investalert.in/v1` |

---

## 2. Security Architecture

### 2.1 Security Layers

| Layer | Implementation | Purpose |
|-------|----------------|---------|
| Edge | Azure Front Door WAF | DDoS protection, OWASP rules |
| Transport | TLS 1.3 mandatory | Encryption in transit |
| Gateway | Kong API Gateway | Rate limiting, JWT validation |
| Application | Service-level auth | Business logic authorization |
| Data | AES-256 encryption | PII encryption at rest |

### 2.2 Security Headers (All Responses)

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
X-Request-ID: {uuid}
```

### 2.3 CORS Policy

```yaml
Allowed Origins:
  - https://app.investalert.in
  - https://admin.investalert.in
Allowed Methods: GET, POST, PUT, DELETE, OPTIONS
Allowed Headers: Authorization, Content-Type, X-Request-ID
Max Age: 86400
Credentials: true
```

### 2.4 IP Whitelisting (Admin APIs)

Admin endpoints (`/admin/*`) restricted to:
- Office IP ranges (configurable)
- VPN endpoints
- Cloud NAT gateways (for internal services)

---

## 3. Authentication & Authorization

### 3.1 Authentication Methods

| Method | Use Case | Token Lifetime |
|--------|----------|----------------|
| JWT Bearer Token | User/Admin authentication | 24 hours |
| Refresh Token | Token renewal | 30 days |
| API Key | Service-to-service | No expiry (rotated quarterly) |
| OTP | Registration/Login verification | 5 minutes |

### 3.2 JWT Token Structure

```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user-uuid",
    "email": "user@example.com",
    "role": "PREMIUM",
    "permissions": ["alerts:read", "profile:write"],
    "subscription_status": "active",
    "iat": 1699000000,
    "exp": 1699086400,
    "iss": "investalert.in",
    "aud": "investalert-api"
  }
}
```

### 3.3 Role-Based Access Control (RBAC)

| Role | Permissions |
|------|-------------|
| GUEST | `register`, `plans:read` |
| FREE | `profile:read/write`, `alerts:historical`, `stats:read` |
| BASIC | FREE + `alerts:realtime` (5/day), `preferences:write` |
| PREMIUM | BASIC + `alerts:realtime` (15/day), `alerts:priority` |
| ADMIN | All + `admin:*`, `audit:read`, `config:write` |

### 3.4 Authorization Header Format

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 3.5 Service-to-Service Authentication

Internal services use mTLS + API keys:

```http
X-Service-Name: ingestion-service
X-API-Key: {encrypted-api-key}
X-Request-ID: {correlation-id}
```

---

## 4. Rate Limiting Strategy

### 4.1 Rate Limit Tiers

| Tier | Requests/Min | Requests/Hour | Requests/Day | Burst |
|------|--------------|---------------|--------------|-------|
| Guest (Unauthenticated) | 10 | 100 | 500 | 15 |
| Free User | 30 | 500 | 2,000 | 50 |
| Basic Subscriber | 60 | 1,000 | 10,000 | 100 |
| Premium Subscriber | 120 | 3,000 | 30,000 | 200 |
| Admin | 300 | 10,000 | Unlimited | 500 |

### 4.2 Rate Limit Headers

```http
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1699000060
X-RateLimit-Retry-After: 30
```

### 4.3 Rate Limit Response (429)

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please retry after 30 seconds.",
    "retry_after": 30,
    "limit": 60,
    "window": "1m"
  }
}
```

### 4.4 Endpoint-Specific Limits

| Endpoint | Additional Limit | Reason |
|----------|------------------|--------|
| POST /auth/login | 5/min per IP | Brute force protection |
| POST /auth/register | 3/hour per IP | Spam prevention |
| POST /auth/otp/send | 3/5min per phone | OTP abuse prevention |
| GET /alerts | Subscription tier limit | Business rule |
| POST /admin/* | 100/min | Audit capacity |

### 4.5 Alert Delivery Rate Limits (Business Logic)

| Subscription | Max Alerts/Day | Priority |
|--------------|----------------|----------|
| Basic | 5 | Standard |
| Premium | 15 | High (first in queue) |

---

## 5. PII Data Handling

### 5.1 PII Classification

| Data Field | Classification | Storage | Transmission | Retention |
|------------|----------------|---------|--------------|-----------|
| Email | PII | Encrypted | TLS | Account lifetime + 90 days |
| Phone | PII | Encrypted | TLS | Account lifetime + 90 days |
| PAN | Sensitive PII | Encrypted + Masked | TLS | Account lifetime + 7 years |
| Name | PII | Encrypted | TLS | Account lifetime + 90 days |
| Device ID | Pseudonymous | Hashed | TLS | 1 year |
| IP Address | PII | Hashed | TLS | 90 days |

### 5.2 PII Encryption Standards

```yaml
Algorithm: AES-256-GCM
Key Management: Azure Key Vault
Key Rotation: 90 days
Field-Level Encryption: PAN, Phone
Database Encryption: TDE (Transparent Data Encryption)
```

### 5.3 PII in API Responses

**Masking Rules:**
- PAN: Show last 4 digits only → `XXXXXX1234`
- Phone: Show last 4 digits → `XXXXXX7890`
- Email: Partial mask → `u***@example.com`

**Example Response:**
```json
{
  "user": {
    "id": "uuid",
    "email": "u***r@example.com",
    "phone": "XXXXXX7890",
    "pan": "XXXXXX1234",
    "name": "John Doe",
    "kyc_verified": true
  }
}
```

### 5.4 PII Access Logging

All PII access is logged:
```json
{
  "timestamp": "2024-11-01T10:00:00Z",
  "action": "PII_ACCESS",
  "user_id": "admin-uuid",
  "target_user_id": "user-uuid",
  "fields_accessed": ["email", "phone"],
  "reason": "support_ticket_12345",
  "ip_address": "hash(ip)",
  "request_id": "req-uuid"
}
```

### 5.5 Data Subject Rights (GDPR/DPDP Compliance)

| Right | Endpoint | SLA |
|-------|----------|-----|
| Access | GET /users/me/data-export | 48 hours |
| Rectification | PUT /users/me/profile | Immediate |
| Erasure | DELETE /users/me | 30 days |
| Portability | GET /users/me/data-export?format=json | 48 hours |

---

## 6. API Standards & Conventions

### 6.1 URL Structure

```
https://api.investalert.in/v1/{service}/{resource}/{id}/{sub-resource}
```

**Examples:**
- `GET /v1/users/me/profile`
- `GET /v1/alerts?status=unread&limit=10`
- `POST /v1/subscriptions/create-order`
- `PUT /v1/admin/sources/{id}`

### 6.2 HTTP Methods

| Method | Usage | Idempotent |
|--------|-------|------------|
| GET | Retrieve resource(s) | Yes |
| POST | Create resource / Action | No |
| PUT | Full update | Yes |
| PATCH | Partial update | Yes |
| DELETE | Remove resource | Yes |

### 6.3 Request Headers

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer {token}
X-Request-ID: {client-generated-uuid}
X-Client-Version: 1.0.0
X-Platform: ios|android|web
Accept-Language: en-IN
```

### 6.4 Response Format (Standard)

**Success Response:**
```json
{
  "success": true,
  "data": { },
  "meta": {
    "request_id": "req-uuid",
    "timestamp": "2024-11-01T10:00:00Z",
    "version": "1.0"
  }
}
```

**Paginated Response:**
```json
{
  "success": true,
  "data": [ ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "has_next": true,
    "next_cursor": "eyJpZCI6MTAwfQ=="
  },
  "meta": { }
}
```

### 6.5 Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| page | int | Page number (1-based) |
| limit | int | Items per page (max: 100) |
| sort | string | Sort field (prefix `-` for desc) |
| filter | string | Filter expression |
| fields | string | Comma-separated fields to return |
| cursor | string | Cursor for cursor-based pagination |

### 6.6 Date/Time Format

All timestamps in ISO 8601 format with timezone:
```
2024-11-01T10:30:00+05:30
```

---

## 7. Service API Specifications

### 7.1 User Service APIs

#### 7.1.1 Register User
```yaml
POST /v1/auth/register
Rate Limit: 3/hour per IP
Auth: None

Request:
{
  "email": "user@example.com",
  "phone": "+919876543210",
  "name": "John Doe",
  "pan": "ABCDE1234F",
  "terms_accepted": true,
  "marketing_consent": false
}

Response (201):
{
  "success": true,
  "data": {
    "user_id": "uuid",
    "otp_sent_to": "phone",
    "otp_expires_at": "2024-11-01T10:05:00Z"
  }
}

Validations:
- Email: Valid format, unique
- Phone: Indian mobile (+91), unique
- PAN: Valid format (AAAAA0000A)
- Terms: Must be true
```

#### 7.1.2 Verify OTP
```yaml
POST /v1/auth/verify-otp
Rate Limit: 5/min per phone
Auth: None

Request:
{
  "phone": "+919876543210",
  "otp": "123456",
  "device_id": "device-uuid",
  "device_info": {
    "platform": "android",
    "version": "1.0.0"
  }
}

Response (200):
{
  "success": true,
  "data": {
    "access_token": "eyJhbG...",
    "refresh_token": "eyJhbG...",
    "token_type": "Bearer",
    "expires_in": 86400,
    "user": {
      "id": "uuid",
      "email": "u***r@example.com",
      "role": "FREE",
      "subscription": null
    }
  }
}

Error (401):
{
  "success": false,
  "error": {
    "code": "INVALID_OTP",
    "message": "Invalid or expired OTP"
  }
}
```

#### 7.1.3 Login
```yaml
POST /v1/auth/login
Rate Limit: 5/min per IP
Auth: None

Request:
{
  "phone": "+919876543210"
}

Response (200):
{
  "success": true,
  "data": {
    "otp_sent": true,
    "otp_expires_at": "2024-11-01T10:05:00Z",
    "masked_phone": "XXXXXX3210"
  }
}
```

#### 7.1.4 Refresh Token
```yaml
POST /v1/auth/refresh
Rate Limit: 10/hour per user
Auth: Refresh Token

Request:
{
  "refresh_token": "eyJhbG..."
}

Response (200):
{
  "success": true,
  "data": {
    "access_token": "eyJhbG...",
    "expires_in": 86400
  }
}
```

#### 7.1.5 Get Profile
```yaml
GET /v1/users/me/profile
Auth: Bearer Token
Roles: FREE, BASIC, PREMIUM, ADMIN

Response (200):
{
  "success": true,
  "data": {
    "id": "uuid",
    "email": "user@example.com",
    "phone": "XXXXXX3210",
    "name": "John Doe",
    "pan": "XXXXXX234F",
    "role": "PREMIUM",
    "kyc_status": "verified",
    "subscription": {
      "plan": "Premium",
      "status": "active",
      "expires_at": "2024-12-01T00:00:00Z",
      "alerts_today": 3,
      "alerts_limit": 15
    },
    "created_at": "2024-01-15T10:00:00Z"
  }
}
```

#### 7.1.6 Update Profile
```yaml
PUT /v1/users/me/profile
Auth: Bearer Token
Roles: FREE, BASIC, PREMIUM

Request:
{
  "name": "John Updated",
  "email": "newemail@example.com"
}

Response (200):
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "John Updated",
    "email_verification_sent": true
  }
}

Note: Email change requires re-verification
```

#### 7.1.7 Request Data Export (GDPR/DPDP)
```yaml
POST /v1/users/me/data-export
Auth: Bearer Token
Rate Limit: 1/day per user

Request:
{
  "format": "json",
  "include": ["profile", "alerts", "preferences", "activity"]
}

Response (202):
{
  "success": true,
  "data": {
    "export_id": "export-uuid",
    "status": "processing",
    "estimated_completion": "2024-11-01T12:00:00Z",
    "download_expires_at": "2024-11-08T12:00:00Z"
  }
}
```

#### 7.1.8 Delete Account
```yaml
DELETE /v1/users/me
Auth: Bearer Token
Rate Limit: 1/month per user

Request:
{
  "confirmation": "DELETE MY ACCOUNT",
  "reason": "optional feedback"
}

Response (202):
{
  "success": true,
  "data": {
    "deletion_scheduled_at": "2024-12-01T00:00:00Z",
    "message": "Account scheduled for deletion in 30 days. Login to cancel."
  }
}
```

---

### 7.2 Subscription Service APIs

#### 7.2.1 Get Plans
```yaml
GET /v1/plans
Auth: Optional
Cache: 1 hour

Response (200):
{
  "success": true,
  "data": [
    {
      "id": "plan-free",
      "name": "Free",
      "price": 0,
      "currency": "INR",
      "duration_days": null,
      "alerts_per_day": 0,
      "features": [
        "Historical alerts (>7 days)",
        "Performance statistics",
        "Basic analytics"
      ]
    },
    {
      "id": "plan-basic",
      "name": "Basic",
      "price": 299,
      "currency": "INR",
      "duration_days": 30,
      "alerts_per_day": 5,
      "features": [
        "Real-time alerts",
        "Up to 5 alerts/day",
        "Email notifications",
        "All sectors"
      ]
    },
    {
      "id": "plan-premium",
      "name": "Premium",
      "price": 799,
      "currency": "INR",
      "duration_days": 30,
      "alerts_per_day": 15,
      "features": [
        "Priority alerts",
        "Up to 15 alerts/day",
        "Push + Email notifications",
        "Early access",
        "Premium support"
      ]
    }
  ]
}
```

#### 7.2.2 Create Subscription Order
```yaml
POST /v1/subscriptions/create-order
Auth: Bearer Token
Roles: FREE, BASIC, PREMIUM

Request:
{
  "plan_id": "plan-premium",
  "coupon_code": "FIRST50"
}

Response (200):
{
  "success": true,
  "data": {
    "order_id": "order_uuid",
    "gateway_order_id": "razorpay_order_id",
    "amount": 399,
    "original_amount": 799,
    "discount": 400,
    "currency": "INR",
    "plan": {
      "id": "plan-premium",
      "name": "Premium"
    },
    "payment_link": "https://razorpay.com/pay/...",
    "expires_at": "2024-11-01T10:30:00Z"
  }
}
```

#### 7.2.3 Payment Webhook (Internal)
```yaml
POST /v1/subscriptions/webhook/razorpay
Auth: Webhook Signature Verification
Internal: Yes

Headers:
X-Razorpay-Signature: {hmac-sha256-signature}

Request:
{
  "event": "payment.captured",
  "payload": {
    "payment": {
      "entity": {
        "id": "pay_xxx",
        "order_id": "order_xxx",
        "amount": 79900,
        "status": "captured"
      }
    }
  }
}

Response (200):
{
  "success": true
}

Actions:
- Verify signature
- Update subscription status
- Update user role
- Send confirmation email
- Log audit event
```

#### 7.2.4 Get Subscription Status
```yaml
GET /v1/subscriptions/me
Auth: Bearer Token

Response (200):
{
  "success": true,
  "data": {
    "id": "sub-uuid",
    "plan": {
      "id": "plan-premium",
      "name": "Premium",
      "alerts_per_day": 15
    },
    "status": "active",
    "start_date": "2024-11-01T00:00:00Z",
    "end_date": "2024-12-01T00:00:00Z",
    "auto_renew": true,
    "days_remaining": 28,
    "usage": {
      "alerts_today": 3,
      "alerts_limit": 15,
      "alerts_this_month": 45
    },
    "payment_method": {
      "type": "card",
      "last4": "4242",
      "brand": "visa"
    }
  }
}
```

#### 7.2.5 Cancel Subscription
```yaml
POST /v1/subscriptions/me/cancel
Auth: Bearer Token

Request:
{
  "reason": "too_expensive",
  "feedback": "Optional feedback text"
}

Response (200):
{
  "success": true,
  "data": {
    "status": "cancelled",
    "access_until": "2024-12-01T00:00:00Z",
    "message": "Subscription cancelled. You'll retain access until the end of your billing period."
  }
}
```

---

### 7.3 Alert Service APIs

#### 7.3.1 Get Alerts (Paginated)
```yaml
GET /v1/alerts
Auth: Bearer Token
Roles: BASIC, PREMIUM (real-time), FREE (historical only)

Query Parameters:
- status: unread|read|all (default: all)
- sector: IT|PHARMA|BANKING|... (optional)
- min_confidence: 60-100 (optional)
- from_date: ISO date (optional)
- to_date: ISO date (optional)
- limit: 1-50 (default: 20)
- cursor: pagination cursor

Response (200):
{
  "success": true,
  "data": [
    {
      "id": "alert-uuid",
      "symbol": "INFY",
      "company_name": "Infosys Ltd",
      "sector": "IT",
      "current_price": 1450.50,
      "target_price": 1650.00,
      "potential_return": "13.8%",
      "confidence_score": 78,
      "horizon_days": 30,
      "sources_count": 3,
      "status": "unread",
      "created_at": "2024-11-01T09:30:00Z",
      "disclaimer": "This is not investment advice. Consult a SEBI registered advisor.",
      "performance": {
        "status": "pending",
        "current_return": "+2.3%"
      }
    }
  ],
  "pagination": {
    "limit": 20,
    "has_next": true,
    "next_cursor": "eyJpZCI6..."
  },
  "meta": {
    "alerts_today": 3,
    "alerts_limit": 15,
    "subscription": "premium"
  }
}

Note for FREE users:
- Only alerts older than 7 days returned
- Performance data visible
- Upgrade CTA included in response
```

#### 7.3.2 Get Alert Details
```yaml
GET /v1/alerts/{alert_id}
Auth: Bearer Token

Response (200):
{
  "success": true,
  "data": {
    "id": "alert-uuid",
    "symbol": "INFY",
    "company_name": "Infosys Ltd",
    "sector": "IT",
    "market_cap": "large",
    "current_price": 1450.50,
    "target_price": 1650.00,
    "potential_return": "13.8%",
    "confidence_score": 78,
    "horizon_days": 30,
    "scoring_breakdown": {
      "fundamental": {
        "total": 42,
        "pe_ratio": 12,
        "debt_equity": 10,
        "promoter_holding": 10,
        "earnings_growth": 10
      },
      "technical": {
        "total": 36,
        "rsi": 12,
        "moving_average": 12,
        "volume": 12
      }
    },
    "sources": [
      {"name": "MoneyControl", "date": "2024-11-01"},
      {"name": "Screener", "date": "2024-10-31"}
    ],
    "created_at": "2024-11-01T09:30:00Z",
    "disclaimer": "This is not investment advice...",
    "performance": {
      "price_at_alert": 1420.00,
      "current_price": 1450.50,
      "return_pct": 2.14,
      "status": "pending",
      "tracked_at": ["7d", "30d", "90d"]
    }
  }
}
```

#### 7.3.3 Mark Alert as Read
```yaml
PUT /v1/alerts/{alert_id}/read
Auth: Bearer Token

Response (200):
{
  "success": true,
  "data": {
    "id": "alert-uuid",
    "is_read": true,
    "read_at": "2024-11-01T10:00:00Z"
  }
}
```

#### 7.3.4 Get/Update Alert Preferences
```yaml
GET /v1/alerts/preferences
PUT /v1/alerts/preferences
Auth: Bearer Token
Roles: BASIC, PREMIUM

Request (PUT):
{
  "push_enabled": true,
  "email_enabled": true,
  "email_frequency": "instant",
  "quiet_hours": {
    "enabled": true,
    "start": "22:00",
    "end": "08:00"
  },
  "sectors_filter": ["IT", "PHARMA", "BANKING"],
  "min_confidence": 70
}

Response (200):
{
  "success": true,
  "data": {
    "push_enabled": true,
    "email_enabled": true,
    "email_frequency": "instant",
    "quiet_hours": {...},
    "sectors_filter": ["IT", "PHARMA", "BANKING"],
    "min_confidence": 70,
    "updated_at": "2024-11-01T10:00:00Z"
  }
}
```

#### 7.3.5 Track Personal Action
```yaml
POST /v1/alerts/{alert_id}/track
Auth: Bearer Token

Request:
{
  "acted_on": true,
  "notes": "Bought 100 shares"
}

Response (200):
{
  "success": true,
  "data": {
    "alert_id": "alert-uuid",
    "acted_on": true,
    "tracked_at": "2024-11-01T10:00:00Z"
  }
}
```

---

### 7.4 Performance Service APIs

#### 7.4.1 Get Platform Statistics
```yaml
GET /v1/performance/stats
Auth: Optional (more details for authenticated)
Cache: 15 minutes

Response (200):
{
  "success": true,
  "data": {
    "overall": {
      "total_alerts": 1250,
      "hit_rate": 68.5,
      "avg_return": 12.3,
      "total_tracked": 980
    },
    "by_horizon": {
      "7d": {"hit_rate": 72.1, "avg_return": 5.2},
      "30d": {"hit_rate": 68.5, "avg_return": 12.3},
      "90d": {"hit_rate": 65.2, "avg_return": 18.7}
    },
    "by_confidence": {
      "60-70": {"hit_rate": 62.0, "count": 320},
      "70-80": {"hit_rate": 70.5, "count": 450},
      "80-90": {"hit_rate": 78.2, "count": 280},
      "90-100": {"hit_rate": 85.1, "count": 150}
    },
    "period": "last_90_days",
    "last_updated": "2024-11-01T06:00:00Z"
  }
}
```

#### 7.4.2 Get Stats by Source
```yaml
GET /v1/performance/by-source
Auth: Bearer Token
Roles: BASIC, PREMIUM

Response (200):
{
  "success": true,
  "data": [
    {
      "source": "MoneyControl",
      "total_alerts": 450,
      "hit_rate": 71.2,
      "avg_return": 14.5
    },
    {
      "source": "Screener",
      "total_alerts": 380,
      "hit_rate": 66.8,
      "avg_return": 11.2
    }
  ]
}
```

#### 7.4.3 Get Stats by Sector
```yaml
GET /v1/performance/by-sector
Auth: Bearer Token

Response (200):
{
  "success": true,
  "data": [
    {
      "sector": "IT",
      "total_alerts": 280,
      "hit_rate": 72.5,
      "avg_return": 15.2
    },
    {
      "sector": "PHARMA",
      "total_alerts": 195,
      "hit_rate": 68.2,
      "avg_return": 11.8
    }
  ]
}
```

#### 7.4.4 Get Personal Performance
```yaml
GET /v1/performance/me
Auth: Bearer Token

Response (200):
{
  "success": true,
  "data": {
    "total_tracked": 25,
    "acted_on": 12,
    "hit_rate_on_acted": 75.0,
    "total_potential_return": 18.5,
    "best_pick": {
      "symbol": "INFY",
      "return": 32.5
    },
    "by_sector": [
      {"sector": "IT", "acted": 5, "hit_rate": 80.0}
    ]
  }
}
```

---

### 7.5 Admin Service APIs

#### 7.5.1 Admin Authentication
```yaml
POST /v1/admin/auth/login
Rate Limit: 3/min per IP
IP Whitelist: Required

Request:
{
  "email": "admin@investalert.in",
  "password": "********",
  "mfa_code": "123456"
}

Response (200):
{
  "success": true,
  "data": {
    "access_token": "eyJhbG...",
    "expires_in": 3600,
    "admin": {
      "id": "admin-uuid",
      "email": "admin@investalert.in",
      "role": "ADMIN",
      "permissions": ["*"]
    }
  }
}

Note: MFA required for all admin accounts
```

#### 7.5.2 List Data Sources
```yaml
GET /v1/admin/sources
Auth: Admin Token

Response (200):
{
  "success": true,
  "data": [
    {
      "id": "source-uuid",
      "name": "NSE India",
      "url": "https://api.nse...",
      "is_active": true,
      "poll_interval_min": 15,
      "health": {
        "status": "healthy",
        "last_success": "2024-11-01T09:45:00Z",
        "success_rate_24h": 98.5,
        "avg_latency_ms": 450
      },
      "stats": {
        "total_ingested": 12500,
        "last_24h": 45
      }
    }
  ]
}
```

#### 7.5.3 Add/Update Data Source
```yaml
POST /v1/admin/sources
PUT /v1/admin/sources/{id}
Auth: Admin Token

Request:
{
  "name": "New Source",
  "url": "https://api.newsource.com/recommendations",
  "auth_type": "api_key",
  "auth_config": {
    "header_name": "X-API-Key",
    "key_vault_secret": "newsource-api-key"
  },
  "poll_interval_min": 30,
  "is_active": true,
  "mapping_config": {
    "symbol_field": "ticker",
    "target_field": "target_price"
  }
}

Response (201):
{
  "success": true,
  "data": {
    "id": "source-uuid",
    "name": "New Source",
    "created_at": "2024-11-01T10:00:00Z",
    "created_by": "admin-uuid"
  }
}

Audit Log: Created automatically
```

#### 7.5.4 Get AI Scoring Configuration
```yaml
GET /v1/admin/ai-config
Auth: Admin Token

Response (200):
{
  "success": true,
  "data": {
    "weights": {
      "fundamental": {
        "pe_ratio": 15,
        "debt_equity": 10,
        "promoter_holding": 15,
        "earnings_growth": 10
      },
      "technical": {
        "rsi": 15,
        "moving_average": 20,
        "volume": 15
      }
    },
    "thresholds": {
      "approval_min": 60,
      "high_confidence": 80
    },
    "version": 5,
    "last_updated": "2024-10-15T10:00:00Z",
    "updated_by": "admin-uuid"
  }
}
```

#### 7.5.5 Update AI Scoring Configuration
```yaml
PUT /v1/admin/ai-config
Auth: Admin Token

Request:
{
  "weights": {
    "fundamental": {
      "pe_ratio": 15,
      "debt_equity": 12,
      "promoter_holding": 13,
      "earnings_growth": 10
    },
    "technical": {
      "rsi": 15,
      "moving_average": 20,
      "volume": 15
    }
  },
  "thresholds": {
    "approval_min": 65,
    "high_confidence": 80
  },
  "change_reason": "Increased debt_equity weight based on market analysis"
}

Response (200):
{
  "success": true,
  "data": {
    "version": 6,
    "updated_at": "2024-11-01T10:00:00Z",
    "updated_by": "admin-uuid",
    "changes": {
      "weights.fundamental.debt_equity": {"old": 10, "new": 12},
      "weights.fundamental.promoter_holding": {"old": 15, "new": 13},
      "thresholds.approval_min": {"old": 60, "new": 65}
    }
  }
}

Validation:
- Sum of fundamental weights = 50
- Sum of technical weights = 50
- approval_min between 50-90
```

#### 7.5.6 Get Rejected Recommendations
```yaml
GET /v1/admin/rejected
Auth: Admin Token

Query Parameters:
- from_date: ISO date
- to_date: ISO date
- min_score: 0-59
- source_id: filter by source
- limit: 1-100

Response (200):
{
  "success": true,
  "data": [
    {
      "id": "rec-uuid",
      "symbol": "XYZ",
      "source": "MoneyControl",
      "composite_score": 52,
      "rejection_reason": "Low fundamental score (PE ratio > 50)",
      "scoring_breakdown": {...},
      "ingested_at": "2024-11-01T09:30:00Z",
      "can_override": true
    }
  ]
}
```

#### 7.5.7 Override Rejected Recommendation
```yaml
POST /v1/admin/override/{recommendation_id}
Auth: Admin Token

Request:
{
  "reason": "Manual analysis confirms strong fundamentals despite high PE",
  "override_score": 75
}

Response (200):
{
  "success": true,
  "data": {
    "id": "rec-uuid",
    "status": "approved_override",
    "override_by": "admin-uuid",
    "override_at": "2024-11-01T10:00:00Z",
    "alert_created": true,
    "alert_id": "alert-uuid"
  }
}

Audit Log: Override reason mandatory, full trail created
```

#### 7.5.8 Get Audit Logs
```yaml
GET /v1/admin/audit-logs
Auth: Admin Token
Roles: ADMIN with audit:read permission

Query Parameters:
- admin_id: filter by admin
- action: filter by action type
- entity_type: user|source|config|override|...
- from_date: ISO date
- to_date: ISO date
- limit: 1-500

Response (200):
{
  "success": true,
  "data": [
    {
      "id": "log-uuid",
      "timestamp": "2024-11-01T10:00:00Z",
      "admin": {
        "id": "admin-uuid",
        "email": "admin@investalert.in"
      },
      "action": "CONFIG_UPDATE",
      "entity_type": "ai_config",
      "entity_id": "config-uuid",
      "changes": {
        "approval_threshold": {"old": 60, "new": 65}
      },
      "reason": "Market adjustment",
      "ip_address": "hash(ip)",
      "user_agent": "Mozilla/5.0..."
    }
  ],
  "pagination": {...}
}
```

#### 7.5.9 User Management
```yaml
GET /v1/admin/users
GET /v1/admin/users/{user_id}
Auth: Admin Token

Query Parameters (list):
- role: FREE|BASIC|PREMIUM
- status: active|suspended|deleted
- search: email or phone (partial)
- limit: 1-100

Response (200):
{
  "success": true,
  "data": [
    {
      "id": "user-uuid",
      "email": "u***r@example.com",
      "phone": "XXXXXX7890",
      "name": "John Doe",
      "role": "PREMIUM",
      "status": "active",
      "subscription": {
        "plan": "Premium",
        "expires_at": "2024-12-01"
      },
      "created_at": "2024-01-15",
      "last_login": "2024-11-01"
    }
  ]
}

Note: PII masked even for admin view
Full PII access requires elevated permission + audit log
```

#### 7.5.10 Modify User Access
```yaml
PUT /v1/admin/users/{user_id}/access
Auth: Admin Token

Request:
{
  "action": "extend_subscription",
  "days": 30,
  "reason": "Compensation for service outage on Oct 15"
}

OR

{
  "action": "suspend",
  "reason": "Terms violation - spam reports"
}

Response (200):
{
  "success": true,
  "data": {
    "user_id": "user-uuid",
    "action": "extend_subscription",
    "new_expiry": "2024-12-31T00:00:00Z",
    "performed_by": "admin-uuid",
    "performed_at": "2024-11-01T10:00:00Z"
  }
}
```

#### 7.5.11 System Health Dashboard
```yaml
GET /v1/admin/health
Auth: Admin Token

Response (200):
{
  "success": true,
  "data": {
    "status": "healthy",
    "services": {
      "user_service": {"status": "up", "latency_ms": 45},
      "subscription_service": {"status": "up", "latency_ms": 52},
      "ingestion_service": {"status": "up", "latency_ms": 120},
      "ai_scoring_service": {"status": "up", "latency_ms": 850},
      "alert_service": {"status": "up", "latency_ms": 65},
      "notification_service": {"status": "up", "latency_ms": 78}
    },
    "dependencies": {
      "postgresql": {"status": "up", "connections": 45},
      "redis": {"status": "up", "memory_used": "256MB"},
      "service_bus": {"status": "up", "queue_depth": 12},
      "vertex_ai": {"status": "up", "latency_ms": 320}
    },
    "metrics": {
      "requests_per_min": 450,
      "error_rate": 0.02,
      "avg_latency_ms": 125
    },
    "last_check": "2024-11-01T10:00:00Z"
  }
}
```

---

## 8. Error Handling

### 8.1 Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable message",
    "details": {
      "field": "Additional context"
    },
    "request_id": "req-uuid",
    "timestamp": "2024-11-01T10:00:00Z",
    "documentation_url": "https://docs.investalert.in/errors/ERROR_CODE"
  }
}
```

### 8.2 Standard Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | VALIDATION_ERROR | Request validation failed |
| 400 | INVALID_REQUEST | Malformed request body |
| 401 | UNAUTHORIZED | Missing or invalid token |
| 401 | TOKEN_EXPIRED | JWT token has expired |
| 401 | INVALID_OTP | OTP verification failed |
| 403 | FORBIDDEN | Insufficient permissions |
| 403 | SUBSCRIPTION_REQUIRED | Feature requires subscription |
| 403 | RATE_LIMIT_ALERT | Daily alert limit reached |
| 404 | NOT_FOUND | Resource not found |
| 409 | CONFLICT | Resource already exists |
| 422 | UNPROCESSABLE_ENTITY | Business rule violation |
| 429 | RATE_LIMIT_EXCEEDED | Too many requests |
| 500 | INTERNAL_ERROR | Server error (logged) |
| 502 | SERVICE_UNAVAILABLE | Downstream service error |
| 503 | MAINTENANCE | Planned maintenance |

### 8.3 Validation Error Example

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": {
      "errors": [
        {
          "field": "email",
          "message": "Invalid email format",
          "value": "invalid-email"
        },
        {
          "field": "pan",
          "message": "PAN must be 10 characters",
          "value": "ABC123"
        }
      ]
    }
  }
}
```

---

## 9. Audit & Compliance

### 9.1 Audit Events

| Event Type | Trigger | Retention |
|------------|---------|-----------|
| AUTH_LOGIN | User/Admin login | 1 year |
| AUTH_LOGOUT | Explicit logout | 1 year |
| AUTH_FAILED | Failed login attempt | 1 year |
| USER_CREATED | New registration | 7 years |
| USER_UPDATED | Profile changes | 7 years |
| USER_DELETED | Account deletion | 7 years |
| SUBSCRIPTION_CREATED | New subscription | 7 years |
| SUBSCRIPTION_CANCELLED | Cancellation | 7 years |
| PAYMENT_SUCCESS | Successful payment | 7 years |
| PAYMENT_FAILED | Failed payment | 7 years |
| PII_ACCESS | Admin views PII | 7 years |
| CONFIG_CHANGE | AI/Source config | 7 years |
| OVERRIDE_CREATED | Manual approval | 7 years |
| DATA_EXPORT | GDPR export request | 7 years |

### 9.2 Audit Log Entry Structure

```json
{
  "id": "audit-uuid",
  "timestamp": "2024-11-01T10:00:00Z",
  "event_type": "CONFIG_CHANGE",
  "actor": {
    "type": "admin",
    "id": "admin-uuid",
    "email": "admin@investalert.in"
  },
  "target": {
    "type": "ai_config",
    "id": "config-uuid"
  },
  "action": "UPDATE",
  "changes": {
    "before": {"approval_threshold": 60},
    "after": {"approval_threshold": 65}
  },
  "reason": "Market adjustment",
  "context": {
    "ip_address": "hash(192.168.1.1)",
    "user_agent": "Mozilla/5.0...",
    "request_id": "req-uuid",
    "session_id": "sess-uuid"
  },
  "compliance": {
    "data_classification": "internal",
    "retention_until": "2031-11-01"
  }
}
```

### 9.3 Compliance Requirements

| Regulation | Requirement | Implementation |
|------------|-------------|----------------|
| SEBI | Disclaimer on alerts | Mandatory field in all alerts |
| DPDP Act | Data subject rights | Export, delete APIs |
| DPDP Act | Consent management | Terms tracking |
| DPDP Act | Data localization | India region only |
| PCI-DSS | Card data handling | No storage, gateway only |

---

## 10. API Versioning Strategy

### 10.1 Versioning Approach

- **URL Path Versioning**: `/v1/`, `/v2/`
- **Major versions only**: Breaking changes
- **Deprecation period**: 6 months minimum

### 10.2 Version Lifecycle

| Phase | Duration | Description |
|-------|----------|-------------|
| Current | Indefinite | Full support |
| Deprecated | 6 months | Sunset header, docs warning |
| Retired | - | Returns 410 Gone |

### 10.3 Deprecation Headers

```http
Deprecation: true
Sunset: Sat, 01 May 2025 00:00:00 GMT
Link: <https://api.investalert.in/v2/users>; rel="successor-version"
```

### 10.4 Breaking vs Non-Breaking Changes

**Non-Breaking (No version bump):**
- Adding new optional fields
- Adding new endpoints
- Adding new optional parameters
- Extending enums (additive)

**Breaking (Version bump required):**
- Removing fields
- Changing field types
- Renaming fields
- Changing URL structure
- Removing endpoints
- Changing authentication

---

## Appendix A: API Quick Reference

| Service | Method | Endpoint | Auth | Rate Limit |
|---------|--------|----------|------|------------|
| Auth | POST | /v1/auth/register | None | 3/hr/IP |
| Auth | POST | /v1/auth/verify-otp | None | 5/min/phone |
| Auth | POST | /v1/auth/login | None | 5/min/IP |
| Auth | POST | /v1/auth/refresh | Refresh | 10/hr |
| User | GET | /v1/users/me/profile | Bearer | Tier |
| User | PUT | /v1/users/me/profile | Bearer | Tier |
| User | POST | /v1/users/me/data-export | Bearer | 1/day |
| User | DELETE | /v1/users/me | Bearer | 1/month |
| Plans | GET | /v1/plans | Optional | 100/hr |
| Subscription | POST | /v1/subscriptions/create-order | Bearer | 10/hr |
| Subscription | GET | /v1/subscriptions/me | Bearer | Tier |
| Subscription | POST | /v1/subscriptions/me/cancel | Bearer | 3/day |
| Alerts | GET | /v1/alerts | Bearer | Tier |
| Alerts | GET | /v1/alerts/{id} | Bearer | Tier |
| Alerts | PUT | /v1/alerts/{id}/read | Bearer | Tier |
| Alerts | GET | /v1/alerts/preferences | Bearer | Tier |
| Alerts | PUT | /v1/alerts/preferences | Bearer | 10/hr |
| Performance | GET | /v1/performance/stats | Optional | 100/hr |
| Performance | GET | /v1/performance/me | Bearer | Tier |
| Admin | POST | /v1/admin/auth/login | None+IP | 3/min |
| Admin | GET | /v1/admin/sources | Admin | 100/min |
| Admin | POST | /v1/admin/sources | Admin | 20/hr |
| Admin | GET | /v1/admin/ai-config | Admin | 100/min |
| Admin | PUT | /v1/admin/ai-config | Admin | 10/hr |
| Admin | GET | /v1/admin/rejected | Admin | 100/min |
| Admin | POST | /v1/admin/override/{id} | Admin | 50/hr |
| Admin | GET | /v1/admin/audit-logs | Admin | 100/min |
| Admin | GET | /v1/admin/users | Admin | 100/min |
| Admin | PUT | /v1/admin/users/{id}/access | Admin | 50/hr |
| Admin | GET | /v1/admin/health | Admin | 300/min |

---

*Document Version: 1.0*  
*API Version: v1*  
*Last Updated: November 2024*
