# HealthVillage – System Architecture Document
**Project ID:** FSD119  
**Date:** January 2026  
**Company:** Civora Nexus Pvt. Ltd.

---

## 1. Executive Summary

HealthVillage is a rural telemedicine platform designed to connect patients in remote areas with healthcare providers through secure, low-bandwidth video/audio consultations. The system implements role-based access control (Patient, Doctor, Admin), encrypted Electronic Health Records (EHR), appointment scheduling, and e-prescription management.

### Key Architectural Principles
- **Security-First Design**: End-to-end encryption, HIPAA-compliant data handling
- **Low-Bandwidth Optimization**: Adaptive bitrate video, compressed assets
- **Role-Based Access Control**: Strict separation of user permissions
- **Audit Trail**: Complete logging of all sensitive data access
- **Scalability**: Horizontally scalable microservices architecture

---

## 2. System Architecture Overview

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLIENT LAYER                                │
├────────────────┬────────────────┬────────────────┬───────────────┤
│  Patient Web   │  Doctor Web    │  Admin Web     │  Mobile PWA   │
│  React SPA     │  React SPA     │  React SPA     │  (Future)     │
└────────┬───────┴────────┬───────┴────────┬───────┴───────┬───────┘
         │                │                │               │
         └────────────────┴────────────────┴───────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LOAD BALANCER / NGINX                         │
│         - SSL Termination (TLS 1.3)                              │
│         - Rate Limiting (100 req/min per IP)                     │
│         - DDoS Protection                                        │
│         - Static Asset Caching                                   │
└──────────────────────────────┬──────────────────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  Auth Service    │  │  REST API Server │  │  WebRTC Signaling│
│                  │  │                  │  │  Server          │
│  - JWT Issuance  │  │  - Business Logic│  │                  │
│  - RBAC          │  │  - Data Access   │  │  - Socket.io     │
│  - Session Mgmt  │  │  - Validation    │  │  - STUN/TURN     │
│  - Rate Limiting │  │  - Logging       │  │  - Connection    │
│                  │  │                  │  │    Management    │
└────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE LAYER                                 │
│  - Appointment Service                                           │
│  - Consultation Service                                          │
│  - EHR Service                                                   │
│  - Prescription Service                                          │
│  - Notification Service                                          │
│  - Audit Service                                                 │
└──────────────────────────────┬──────────────────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  PostgreSQL 15   │  │  Redis 7.x       │  │  MinIO / S3      │
│                  │  │                  │  │                  │
│  - User Data     │  │  - Session Store │  │  - Prescriptions │
│  - EHR (Encrypted│  │  - Rate Limiting │  │  - Lab Reports   │
│  - Appointments  │  │  - WebRTC State  │  │  - Documents     │
│  - Prescriptions │  │  - Cache Layer   │  │  (Encrypted)     │
│  - Audit Logs    │  │                  │  │                  │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

### 2.2 Component Descriptions

#### **Frontend Layer**
- **Technology**: React 18 with Vite
- **State Management**: Context API + useReducer
- **Routing**: React Router v6 with protected routes
- **WebRTC**: simple-peer for video/audio
- **Bundle Strategy**: Code-splitting by role (Patient/Doctor/Admin)

#### **API Gateway / Load Balancer**
- **Technology**: Nginx
- **Responsibilities**:
  - SSL/TLS termination
  - Request routing
  - Rate limiting (per IP and per user)
  - Static asset serving with caching
  - WebSocket proxy for Socket.io

#### **Authentication Service**
- **Technology**: Node.js + Express + Passport.js
- **Authentication Strategy**: JWT with refresh tokens
- **Session Storage**: Redis (1-hour TTL)
- **Features**:
  - Email/password authentication
  - Account lockout after 5 failed attempts
  - Email verification
  - Password reset flow

#### **REST API Server**
- **Technology**: Node.js + Express
- **ORM**: Sequelize with PostgreSQL
- **Validation**: Joi schemas
- **Error Handling**: Centralized error middleware
- **Logging**: Winston with daily rotation

#### **WebRTC Signaling Server**
- **Technology**: Socket.io
- **STUN/TURN**: Coturn server for NAT traversal
- **Features**:
  - Peer-to-peer connection establishment
  - Room management
  - Connection quality monitoring
  - Automatic reconnection

#### **Database Layer**
- **Primary Database**: PostgreSQL 15
  - ACID compliance
  - Row-level security
  - Connection pooling (max 20 connections)
- **Cache Layer**: Redis 7.x
  - Session storage
  - Rate limiting counters
  - Temporary consultation state
- **Object Storage**: MinIO (S3-compatible)
  - Encrypted file storage
  - Versioning enabled
  - 7-year retention policy

---

## 3. Security Architecture

### 3.1 Defense in Depth

```
Layer 1: Network Security
├── Firewall rules (allow only 80, 443, 22)
├── DDoS protection (Cloudflare/AWS Shield)
└── VPC isolation (database not publicly accessible)

Layer 2: Application Security
├── Input validation (Joi schemas)
├── SQL injection prevention (parameterized queries)
├── XSS prevention (React auto-escaping, CSP headers)
├── CSRF protection (SameSite cookies)
└── Rate limiting (100 req/min per endpoint)

Layer 3: Authentication & Authorization
├── JWT with short expiry (1 hour)
├── Refresh tokens (7 days, httpOnly cookies)
├── Role-Based Access Control (RBAC)
├── Session invalidation on logout
└── Account lockout after failed attempts

Layer 4: Data Security
├── Encryption at rest (AES-256-GCM)
├── Encryption in transit (TLS 1.3)
├── Encrypted backups
├── Secure key management (AWS KMS / HashiCorp Vault)
└── Data masking in logs

Layer 5: Audit & Monitoring
├── Complete audit trail (who, what, when)
├── Real-time anomaly detection
├── Security event logging
└── Automated alerts for suspicious activity
```

### 3.2 Encryption Strategy

#### **Data at Rest**
```javascript
// Application-level encryption for sensitive fields
Algorithm: AES-256-GCM
Key Management: Environment variable (32-byte key)
Encrypted Fields:
  - patient_profiles: address, emergency_contact, medical_history
  - health_records: symptoms, diagnosis, treatment_plan, clinical_notes, vitals
  - prescriptions: prescription_data

Format: iv:authTag:encryptedData
```

#### **Data in Transit**
- TLS 1.3 for all HTTP traffic
- DTLS-SRTP for WebRTC media streams
- WSS (WebSocket Secure) for signaling

#### **Password Security**
```javascript
Algorithm: bcrypt
Salt Rounds: 12
Storage: Hashed password only (plaintext never stored)
Minimum Requirements: 8 chars, 1 uppercase, 1 lowercase, 1 number, 1 special char
```

### 3.3 Role-Based Access Control (RBAC)

```javascript
Roles:
  - patient
  - doctor
  - admin

Permissions Matrix:

┌─────────────────────────┬─────────┬────────┬───────┐
│ Resource                │ Patient │ Doctor │ Admin │
├─────────────────────────┼─────────┼────────┼───────┤
│ View own profile        │    ✓    │   ✓    │   ✓   │
│ Update own profile      │    ✓    │   ✓    │   ✓   │
│ View all users          │    ✗    │   ✗    │   ✓   │
│ Book appointment        │    ✓    │   ✗    │   ✗   │
│ View own appointments   │    ✓    │   ✓    │   ✓   │
│ Set availability        │    ✗    │   ✓    │   ✗   │
│ Create EHR              │    ✗    │   ✓    │   ✗   │
│ View own EHR            │    ✓    │   ✗    │   ✓   │
│ View patient EHR        │    ✗    │   ✓*   │   ✓   │
│ Create prescription     │    ✗    │   ✓    │   ✗   │
│ View own prescriptions  │    ✓    │   ✗    │   ✓   │
│ Generate reports        │    ✗    │   ✗    │   ✓   │
│ View audit logs         │    ✗    │   ✗    │   ✓   │
└─────────────────────────┴─────────┴────────┴───────┘

* Doctor can only view EHR of patients they have consulted
```

### 3.4 Audit Logging

Every sensitive operation is logged:
```javascript
Logged Events:
  - User login/logout
  - EHR access (view, create, update)
  - Prescription creation
  - Appointment booking/cancellation
  - Profile updates
  - Admin actions
  - Failed authentication attempts

Log Format:
{
  timestamp: ISO8601,
  user_id: UUID,
  action: 'ehr_access',
  resource_type: 'health_record',
  resource_id: UUID,
  ip_address: '192.168.1.1',
  user_agent: 'Mozilla/5.0...',
  details: { field_accessed: 'diagnosis' }
}

Retention: 7 years (compliance requirement)
Storage: Write-only table (no updates/deletes allowed)
```

---

## 4. Low-Bandwidth Optimization

### 4.1 Video/Audio Optimization

```javascript
WebRTC Configuration:
  Video:
    - Resolution: 640x480 (adaptive up to 1280x720)
    - Frame rate: 15fps (adaptive up to 30fps)
    - Codec: VP9 (better compression than H.264)
    - Bitrate: 300-800 kbps (adaptive)
  
  Audio:
    - Codec: Opus (excellent for voice)
    - Bitrate: 32 kbps
    - Echo cancellation: enabled
    - Noise suppression: enabled

Network Adaptation:
  - Monitor RTT (Round Trip Time)
  - If RTT > 300ms: reduce resolution to 320x240
  - If packet loss > 5%: disable video, audio-only mode
  - Automatic reconnection on connection drop
```

### 4.2 Frontend Optimization

```javascript
Bundle Size Targets:
  - Initial load: < 200 KB (gzipped)
  - Patient dashboard: < 150 KB
  - Doctor dashboard: < 180 KB
  - Admin dashboard: < 200 KB

Optimization Techniques:
  - Code splitting by route and role
  - Lazy loading of WebRTC components
  - Tree shaking (remove unused code)
  - Image compression (WebP format)
  - Service Worker for offline capability
  - Asset caching (1 week for static files)
```

### 4.3 Backend Optimization

```javascript
Database:
  - Connection pooling (max 20 connections)
  - Query optimization (indexed foreign keys)
  - Materialized views for reports
  - Pagination (20 records per page)

API:
  - Response compression (gzip)
  - Partial responses (field filtering)
  - Efficient serialization (avoid N+1 queries)
  - Rate limiting to prevent overload

Caching Strategy:
  - Doctor availability: Redis cache (5 min TTL)
  - User profiles: Redis cache (15 min TTL)
  - Static content: CDN caching (1 week)
```

---

## 5. Scalability Considerations

### 5.1 Horizontal Scaling

```
Current Architecture (Single Server):
  - Handles: ~100 concurrent consultations
  - Database: Single PostgreSQL instance

Future Scaling (When needed):
  - Load balancer with multiple API servers
  - PostgreSQL read replicas
  - Redis cluster (master-slave replication)
  - Separate WebRTC signaling server cluster
  - Microservices architecture (appointment, consultation, EHR as separate services)
```

### 5.2 Database Scaling Strategy

```
Phase 1 (Current): Single PostgreSQL instance with optimized queries
Phase 2 (1000+ users): Read replicas for reporting queries
Phase 3 (10,000+ users): Sharding by region/doctor
Phase 4 (100,000+ users): Separate databases per service (microservices)
```

---

## 6. Disaster Recovery & Backup

```
Backup Strategy:
  - Full database backup: Daily at 2 AM
  - Incremental backup: Every 6 hours
  - Retention: 30 days
  - Offsite storage: AWS S3 Glacier (encrypted)

Recovery Objectives:
  - RTO (Recovery Time Objective): 4 hours
  - RPO (Recovery Point Objective): 6 hours

Disaster Scenarios:
  - Database corruption: Restore from latest backup
  - Server failure: Failover to standby server
  - Data breach: Incident response plan (notify users within 72 hours)
```

---

## 7. Technology Stack Summary

```
Frontend:
├── Framework: React 18.2
├── Build Tool: Vite 5.x
├── State: Context API + useReducer
├── Routing: React Router v6
├── Forms: React Hook Form + Zod
├── HTTP: Axios with interceptors
├── WebRTC: simple-peer
└── Testing: Jest + React Testing Library

Backend:
├── Runtime: Node.js v20 LTS
├── Framework: Express.js 4.x
├── ORM: Sequelize 6.x
├── Authentication: Passport.js + JWT
├── Validation: Joi
├── Real-time: Socket.io
├── Logging: Winston
└── Testing: Jest + Supertest

Database:
├── Primary: PostgreSQL 15
├── Cache: Redis 7.x
└── Object Storage: MinIO (S3-compatible)

Infrastructure:
├── Web Server: Nginx
├── TURN Server: Coturn
├── Container: Docker (optional)
└── CI/CD: GitHub Actions
```

---

## 8. Non-Functional Requirements

### 8.1 Performance
- API response time: < 500ms (95th percentile)
- Page load time: < 3 seconds on 3G connection
- Video connection establishment: < 5 seconds
- Database query time: < 100ms

### 8.2 Availability
- Uptime target: 99.5% (43.8 hours downtime/year)
- Scheduled maintenance: Weekly, 2-4 AM

### 8.3 Compliance
- HIPAA-equivalent data handling
- GDPR-compliant (data portability, right to deletion)
- 7-year data retention for medical records

---

## 9. Risks & Mitigations

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Data breach | High | Medium | Encryption, audit logs, penetration testing |
| WebRTC connection failure | High | Medium | Fallback to audio-only, TURN server |
| Database overload | Medium | Low | Connection pooling, read replicas |
| Key compromise | High | Low | Key rotation policy, secure key storage |
| Denial of Service | Medium | Medium | Rate limiting, DDoS protection |

---

## 10. Deployment Architecture

```
Production Environment:
├── Web Server: Nginx on Ubuntu 22.04 LTS
├── Application Server: Node.js (PM2 process manager)
├── Database Server: PostgreSQL 15 (dedicated server)
├── Cache Server: Redis 7.x
├── Object Storage: MinIO cluster
└── TURN Server: Coturn

Development Environment:
├── Docker Compose setup
├── Hot reload enabled
└── Seed data included

Staging Environment:
├── Identical to production
├── Used for final testing
└── Anonymized production data
```

---

## Appendix A: Glossary

- **EHR**: Electronic Health Record
- **RBAC**: Role-Based Access Control
- **JWT**: JSON Web Token
- **TURN**: Traversal Using Relays around NAT
- **STUN**: Session Traversal Utilities for NAT
- **WebRTC**: Web Real-Time Communication
- **TLS**: Transport Layer Security
- **AES-GCM**: Advanced Encryption Standard - Galois/Counter Mode


---

**End of Architecture Document**