# Bangladesh Asset Recovery Platform - System Design Document

## Executive Summary

A civic tech platform enabling citizen-driven asset recovery and illicit wealth tracking in Bangladesh, focusing on the estimated $200-234 billion allegedly plundered during recent authoritarian rule.

**Tech Stack**: Go (Backend) + Next.js (Frontend) + PostgreSQL + Redis + Object Storage

---

## 1. System Architecture Overview

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Web App    │  │  Mobile Web  │  │  Admin Panel │          │
│  │  (Next.js)   │  │  (Next.js)   │  │  (Next.js)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │   NGINX / Traefik (Load Balancer + Rate Limiting)        │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Application Layer (Go)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Report     │  │ Verification │  │  Analytics   │          │
│  │   Service    │  │   Service    │  │   Service    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │    User      │  │   Search     │  │  Blockchain  │          │
│  │   Service    │  │   Service    │  │   Service    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Data Layer                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  PostgreSQL  │  │    Redis     │  │ Elasticsearch│          │
│  │  (Primary)   │  │   (Cache)    │  │ (Search)     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │    MinIO     │  │   Message    │  │  Blockchain  │          │
│  │  (Storage)   │  │  Queue(NATS) │  │   Ledger     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                   External Integration Layer                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ NID Verify   │  │ Land Records │  │  RJSC API    │          │
│  │   API        │  │  (e-Porcha)  │  │  (Company)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   SMS/OTP    │  │  ACC Public  │  │  Media APIs  │          │
│  │   Gateway    │  │   Database   │  │ (Forensics)  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Core Modules & Services

### 2.1 Citizen Reporting Module

**Purpose**: Secure, multi-tiered submission system for asset recovery leads

**Key Features**:
- Anonymous & verified submission modes
- Structured data collection with evidence upload
- Real-time validation and duplicate detection
- Multi-language support (Bengali/English)

**API Endpoints**:
```
POST   /api/v1/reports                    # Submit new report
GET    /api/v1/reports/:id                # Get report details
PUT    /api/v1/reports/:id/evidence       # Add evidence
GET    /api/v1/reports                    # List reports (filtered)
POST   /api/v1/reports/:id/corroborate    # Community corroboration
```

**Database Schema** (PostgreSQL):
```sql
-- Reports table
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    submission_type VARCHAR(20) CHECK (submission_type IN ('anonymous', 'verified')),
    status VARCHAR(20) DEFAULT 'pending',
    credibility_score INTEGER DEFAULT 0,
    
    -- Report content
    title VARCHAR(500) NOT NULL,
    description TEXT NOT NULL,
    asset_type VARCHAR(50),
    involved_parties JSONB,
    location JSONB, -- district, upazila, coordinates
    estimated_value DECIMAL(20,2),
    timeline JSONB,
    
    -- Verification metadata
    auto_check_results JSONB,
    verification_notes TEXT,
    assigned_to UUID,
    
    -- User tracking
    submitter_id UUID,
    submitter_ip_hash VARCHAR(64),
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    verified_at TIMESTAMP,
    
    -- Full-text search
    search_vector tsvector
);

CREATE INDEX idx_reports_status ON reports(status);
CREATE INDEX idx_reports_credibility ON reports(credibility_score DESC);
CREATE INDEX idx_reports_search ON reports USING gin(search_vector);

-- Evidence table
CREATE TABLE evidence (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID REFERENCES reports(id),
    evidence_type VARCHAR(50), -- photo, document, audio, video, link
    file_path VARCHAR(500),
    file_hash VARCHAR(64),
    metadata JSONB, -- EXIF, GPS, timestamps
    forensics_results JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Corroboration table
CREATE TABLE corroborations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID REFERENCES reports(id),
    user_id UUID,
    corroboration_type VARCHAR(20), -- upvote, additional_info, witness
    content TEXT,
    credibility_boost INTEGER,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(report_id, user_id)
);
```

---

### 2.2 Verification Service

**Purpose**: Multi-layered automated and human verification pipeline

**Components**:

1. **Automated Technical Checks** (Go services):
   - Duplicate detection (text similarity using TF-IDF)
   - Consistency scoring (LLM-based or rule-based)
   - Media forensics (EXIF extraction, reverse image search)
   - Geolocation validation

2. **Human Moderation Queue**:
   - Tiered priority system
   - Community review interface
   - Fact-checker assignment workflow

3. **Cross-Verification Engine**:
   - Integration with public databases
   - Blockchain hash logging
   - External data corroboration

**Verification Workflow**:
```
Report Submitted
    ↓
Auto Checks (30s)
    ↓
├─ High Credibility (Score > 70) → Priority Queue
├─ Medium (Score 40-70) → Community Review
└─ Low (Score < 40) → Extended Review / Flag
    ↓
Human Review
    ↓
Cross-Verification with Official Data
    ↓
├─ Verified → Forward to Task Force
├─ Partially Verified → Request More Info
└─ Rejected → Archive with Reason
```

**API Endpoints**:
```
POST   /api/v1/verify/auto-check/:reportId    # Run automated checks
GET    /api/v1/verify/queue                   # Get moderation queue
PUT    /api/v1/verify/:reportId/assign        # Assign to moderator
POST   /api/v1/verify/:reportId/cross-check   # External data verification
PUT    /api/v1/verify/:reportId/status        # Update verification status
```

---

### 2.3 Beneficial Ownership Tracker

**Purpose**: AI-powered network analysis for exposing hidden asset ownership

**Features**:
- Company registry scraping (RJSC integration)
- Network graph visualization of ownership chains
- Panama Papers / leaked data integration
- Risk scoring for entities

**Database Schema**:
```sql
CREATE TABLE entities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(20), -- person, company, trust
    name VARCHAR(500),
    nid VARCHAR(20),
    tin VARCHAR(20),
    registration_number VARCHAR(100),
    metadata JSONB,
    risk_score INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE ownership_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_entity_id UUID REFERENCES entities(id),
    owned_entity_id UUID REFERENCES entities(id),
    ownership_percentage DECIMAL(5,2),
    relationship_type VARCHAR(50),
    source VARCHAR(200),
    valid_from DATE,
    valid_to DATE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE assets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id UUID REFERENCES entities(id),
    asset_type VARCHAR(50), -- property, company_share, bank_account
    description TEXT,
    value DECIMAL(20,2),
    location JSONB,
    acquisition_date DATE,
    source VARCHAR(200),
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

### 2.4 Recovery Progress Tracker

**Purpose**: Public dashboard showing real-time asset recovery status

**Features**:
- Live statistics (seized vs outstanding)
- Interactive maps and charts
- Case timeline tracking
- Sector-wise breakdown

**API Endpoints**:
```
GET    /api/v1/recovery/stats              # Overall statistics
GET    /api/v1/recovery/timeline           # Recovery timeline
GET    /api/v1/recovery/by-sector          # Sector breakdown
GET    /api/v1/recovery/cases              # List recovery cases
GET    /api/v1/recovery/map                # Geographic distribution
```

**Database Schema**:
```sql
CREATE TABLE recovery_cases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_number VARCHAR(100),
    title VARCHAR(500),
    description TEXT,
    related_entities UUID[],
    total_value DECIMAL(20,2),
    recovered_value DECIMAL(20,2),
    status VARCHAR(50), -- ongoing, completed, appealed
    sector VARCHAR(100),
    location JSONB,
    filed_date DATE,
    updated_at TIMESTAMP DEFAULT NOW()
);
```

---

## 3. Technical Implementation Details

### 3.1 Backend Architecture (Go)

**Project Structure**:
```
/backend
├── cmd/
│   ├── api/              # Main API server
│   └── workers/          # Background workers
├── internal/
│   ├── domain/           # Business logic & models
│   │   ├── report/
│   │   ├── verification/
│   │   ├── user/
│   │   └── recovery/
│   ├── repository/       # Data access layer
│   ├── service/          # Business services
│   ├── handler/          # HTTP handlers
│   ├── middleware/       # Auth, logging, rate limiting
│   └── worker/           # Background jobs
├── pkg/
│   ├── auth/             # JWT, OAuth
│   ├── blockchain/       # Hash logging
│   ├── forensics/        # Media analysis
│   ├── similarity/       # Duplicate detection
│   └── validator/        # Input validation
├── migrations/           # Database migrations
├── config/               # Configuration
└── go.mod
```

**Key Go Libraries**:
- `gorilla/mux` or `chi` - HTTP routing
- `sqlx` - Database access
- `go-redis` - Caching
- `jwt-go` - Authentication
- `nats.go` - Message queue
- `testify` - Testing

**Configuration Management**:
```go
type Config struct {
    Server struct {
        Port string
        ReadTimeout time.Duration
        WriteTimeout time.Duration
    }
    Database struct {
        Host string
        Port int
        User string
        Password string
        DBName string
    }
    Redis struct {
        Host string
        Port int
    }
    Storage struct {
        Endpoint string
        Bucket string
    }
    Blockchain struct {
        Enabled bool
        NodeURL string
    }
    External struct {
        NIDVerifyURL string
        RJSCApiURL string
    }
}
```

---

### 3.2 Frontend Architecture (Next.js)

**Project Structure**:
```
/frontend
├── src/
│   ├── app/
│   │   ├── (public)/
│   │   │   ├── page.tsx           # Landing page
│   │   │   ├── reports/
│   │   │   │   ├── page.tsx       # Browse reports
│   │   │   │   ├── [id]/page.tsx  # Report details
│   │   │   │   └── submit/page.tsx # Submit report
│   │   │   ├── tracker/page.tsx   # Recovery tracker
│   │   │   └── ownership/page.tsx # BO network viewer
│   │   ├── (admin)/
│   │   │   ├── dashboard/
│   │   │   ├── verify/
│   │   │   └── analytics/
│   │   └── api/                   # API routes (if needed)
│   ├── components/
│   │   ├── ui/                    # shadcn/ui components
│   │   ├── forms/
│   │   │   ├── ReportSubmission.tsx
│   │   │   └── EvidenceUpload.tsx
│   │   ├── visualizations/
│   │   │   ├── RecoveryChart.tsx
│   │   │   ├── NetworkGraph.tsx
│   │   │   └── GeographicMap.tsx
│   │   └── verification/
│   │       ├── CredibilityScore.tsx
│   │       └── ModerationQueue.tsx
│   ├── lib/
│   │   ├── api.ts                 # API client
│   │   ├── auth.ts
│   │   └── utils.ts
│   ├── hooks/
│   ├── types/
│   └── i18n/                      # Bengali/English
├── public/
└── package.json
```

**Key Features**:
- Server-side rendering for public pages
- Client-side state management (Zustand or Context)
- Form validation (React Hook Form + Zod)
- Data visualization (Recharts, D3.js for network graphs)
- Map integration (Leaflet or Mapbox)
- i18n support (next-intl)

---

### 3.3 Authentication & Authorization

**Multi-tier Access System**:

1. **Anonymous** - Submit reports only
2. **Verified Citizen** - Submit + corroborate + higher credibility
3. **Moderator** - Review queue access
4. **Task Force** - Full verification + case management
5. **Admin** - System configuration

**Implementation**:
- JWT tokens with refresh mechanism
- Optional NID-linked verification (via government API)
- SMS OTP for phone verification
- Role-based access control (RBAC)

```go
type User struct {
    ID           uuid.UUID
    Email        string
    Phone        string
    NID          string // encrypted
    Role         string
    IsVerified   bool
    ReputationScore int
    CreatedAt    time.Time
}
```

---

### 3.4 Security Measures

**Data Protection**:
- End-to-end encryption for sensitive evidence
- PII encryption at rest (NID, phone, submitter info)
- Hashed IP addresses (for anonymity)
- Secure file upload with virus scanning

**Rate Limiting**:
- Per IP: 100 requests/hour (public)
- Per authenticated user: 1000 requests/hour
- Report submission: 10/day (anonymous), 50/day (verified)

**Audit Trail**:
```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID,
    action VARCHAR(100),
    resource_type VARCHAR(50),
    resource_id UUID,
    changes JSONB,
    ip_address VARCHAR(45),
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

### 3.5 Verification Algorithms

**1. Duplicate Detection**:
```go
func CalculateSimilarity(text1, text2 string) float64 {
    // TF-IDF based cosine similarity
    // Return score 0.0 - 1.0
    // Flag if > 0.85 as potential duplicate
}
```

**2. Credibility Scoring**:
```go
type CredibilityFactors struct {
    HasEvidence float64        // 0-30 points
    EvidenceQuality float64    // 0-20 points
    SubmitterVerified float64  // 0-15 points
    GeoValidated float64       // 0-10 points
    Corroborations int         // 2 points each
    CrossVerified bool         // 15 points
    StructuredData float64     // 0-10 points
}

func CalculateCredibility(report Report, factors CredibilityFactors) int {
    score := 0
    score += int(factors.HasEvidence)
    score += int(factors.EvidenceQuality)
    // ... aggregate all factors
    return min(score, 100)
}
```

**3. Media Forensics**:
- EXIF metadata extraction
- GPS coordinate validation
- Timestamp consistency check
- Reverse image search (via TinEye API or similar)
- Basic deepfake detection (error level analysis)

---

### 3.6 Data Integration Strategy

**Government APIs** (simulated for MVP, real integration for production):
- NID Verification API
- Land Records (e-Porcha)
- RJSC Company Registry
- ACC Public Database

**Mock Data for Demo**:
Create realistic synthetic datasets:
- 1000+ sample reports
- 500+ entity network
- 100+ recovery cases
- Geographic data for all districts

---

## 4. Deployment Architecture

### 4.1 Infrastructure

**Cloud Provider**: DigitalOcean / AWS / Local Hosting

**Services**:
```
┌─────────────────────────────────────┐
│  CDN (Cloudflare)                   │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│  Load Balancer (NGINX)              │
└─────────────────────────────────────┘
              ↓
┌──────────┬──────────┬──────────────┐
│  API     │  API     │  API         │
│  Server  │  Server  │  Server      │
│  (Go)    │  (Go)    │  (Go)        │
└──────────┴──────────┴──────────────┘
              ↓
┌──────────┬──────────┬──────────────┐
│PostgreSQL│  Redis   │  MinIO       │
│ Primary  │  Cache   │  Storage     │
└──────────┴──────────┴──────────────┘
```

**Docker Compose Setup**:
```yaml
version: '3.8'
services:
  api:
    build: ./backend
    ports: ["8080:8080"]
    depends_on: [postgres, redis]
    
  frontend:
    build: ./frontend
    ports: ["3000:3000"]
    
  postgres:
    image: postgres:16
    volumes: [./data/postgres:/var/lib/postgresql/data]
    
  redis:
    image: redis:7-alpine
    
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
```

---

### 4.2 Scaling Strategy

**Phase 1 (MVP)**: Single server, 10K users
**Phase 2**: Horizontal scaling, 100K users
**Phase 3**: Microservices, 1M+ users

**Bottleneck Management**:
- Read replicas for PostgreSQL
- Redis cluster for caching
- CDN for static assets
- Background workers for heavy tasks (forensics, similarity)

---

## 5. Bangladesh-Specific Customizations

### 5.1 Localization
- Bengali UI (primary)
- Bengali NLP for text analysis
- Local date/number formats
- District/upazila dropdowns

### 5.2 Asset Categories
```go
var AssetTypes = []string{
    "land_property",      // জমি সম্পত্তি
    "building",           // ভবন
    "company_shares",     // কোম্পানি শেয়ার
    "bank_accounts",      // ব্যাংক হিসাব
    "vehicles",           // যানবাহন
    "jewelry_gold",       // স্বর্ণালংকার
    "offshore_accounts",  // অফশোর একাউন্ট
    "cash",               // নগদ অর্থ
}
```

### 5.3 Geographic Integration
- Bangladesh district/upazila database
- Kushtia & Khulna specific modules
- Land plot DID number system
- GPS coordinates for BD boundaries

---

## 6. Success Metrics & Analytics

**Key Metrics**:
- Reports submitted (daily/weekly)
- Verification completion rate
- Average credibility score
- Corroboration participation
- Recovery case progress
- User engagement (return visits)

**Analytics Dashboard** (Admin):
- Real-time submission trends
- Heatmap of report origins
- Sector-wise distribution
- Top contributing users
- Verification pipeline status

---

## 7. Roadmap

### Phase 1 (Weeks 1-4): MVP
- [ ] Basic report submission (anonymous + verified)
- [ ] Auto-check pipeline (duplicate, basic validation)
- [ ] Recovery progress tracker (with mock data)
- [ ] Simple admin moderation queue

### Phase 2 (Weeks 5-8): Enhanced Verification
- [ ] Community corroboration system
- [ ] Media forensics integration
- [ ] NID verification API
- [ ] Credibility scoring v2

### Phase 3 (Weeks 9-12): Advanced Features
- [ ] Beneficial ownership network graph
- [ ] RJSC data integration
- [ ] Blockchain hash logging
- [ ] Mobile app (React Native)

### Phase 4 (Weeks 13+): Scale & Partnerships
- [ ] StAR Initiative collaboration
- [ ] ACC integration
- [ ] Multi-language expansion
- [ ] AI-powered risk assessment

---

## 8. Risk Mitigation

**Technical Risks**:
- Data breach → Encryption + security audits
- Server overload → Auto-scaling + CDN
- False reports → Multi-layer verification

**Political/Legal Risks**:
- Government resistance → Open-source + international hosting
- Whistleblower safety → Tor support + anonymous mode
- Legal challenges → Clear disclaimers + legal partnership

**Operational Risks**:
- Low adoption → Community outreach + media campaigns
- Funding → Grant applications (World Bank, Open Society)
- Misinformation → Strict verification + transparency reports

---

## 9. Cost Estimation (Monthly)

**MVP Phase**:
- Server (4GB RAM): $24/month
- Database (2GB): $15/month
- Storage (100GB): $10/month
- Domain + SSL: $5/month
- **Total**: ~$54/month

**Production Phase** (100K users):
- Servers (load balanced): $200/month
- Database (managed): $100/month
- Storage (1TB): $50/month
- CDN: $50/month
- **Total**: ~$400/month

---

## 10. Next Steps for Implementation

1. **Set up development environment**
   - Initialize Go project with clean architecture
   - Bootstrap Next.js with TypeScript + shadcn/ui
   - Docker compose for local dev

2. **Database schema migration**
   - PostgreSQL setup
   - Run initial migrations
   - Seed with sample data

3. **Build core services**
   - Report submission API
   - User authentication
   - File upload service

4. **Frontend foundation**
   - Landing page
   - Report submission form
   - Recovery tracker dashboard

5. **Testing & deployment**
   - Unit tests (Go)
   - E2E tests (Playwright)
   - Deploy to staging

---

## Appendix: API Specification Summary

**Authentication**:
```
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/verify-otp
POST   /api/v1/auth/refresh
```

**Reports**:
```
POST   /api/v1/reports
GET    /api/v1/reports
GET    /api/v1/reports/:id
PUT    /api/v1/reports/:id
DELETE /api/v1/reports/:id
POST   /api/v1/reports/:id/evidence
POST   /api/v1/reports/:id/corroborate
```

**Verification**:
```
POST   /api/v1/verify/auto-check/:id
GET    /api/v1/verify/queue
PUT    /api/v1/verify/:id/assign
PUT    /api/v1/verify/:id/status
```

**Recovery Tracker**:
```
GET    /api/v1/recovery/stats
GET    /api/v1/recovery/cases
GET    /api/v1/recovery/timeline
```

**Ownership**:
```
GET    /api/v1/ownership/entities
GET    /api/v1/ownership/network/:entityId
POST   /api/v1/ownership/search
```

---

**Document Version**: 1.0  
**Last Updated**: January 30, 2026  
**Author**: System Architect
