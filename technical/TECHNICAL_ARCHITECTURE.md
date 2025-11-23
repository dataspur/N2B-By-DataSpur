# BullshID Technical Architecture
## Building a Mobile-First Fake ID Detection System with Open-Source Components

---

## Architecture Overview

BullshID is a **mobile-first, hybrid cloud** system that combines open-source components (40-50% code reuse) with proprietary algorithms (50-60% unique IP) to deliver enterprise-grade fake ID detection on consumer devices (iPhones, iPads, Android phones/tablets).

### System Components

```
┌─────────────────────────────────────────────────────────────┐
│                    MOBILE APP LAYER                         │
│  (iOS/Android Native or Flutter Cross-Platform)             │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Camera   │  │ Document │  │ Face     │  │ Offline  │   │
│  │ Capture  │  │ Scanning │  │ Capture  │  │ Storage  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                          ↕ HTTPS/REST API
┌─────────────────────────────────────────────────────────────┐
│                   CLOUD SERVICES LAYER                       │
│  (Self-Hosted or AWS/Azure/GCP)                             │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ CompreFace│  │ Document │  │ Barcode  │  │ Analytics│   │
│  │ (Faces)  │  │ OCR      │  │ Validator│  │ Dashboard│   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                          ↕ Database Queries
┌─────────────────────────────────────────────────────────────┐
│                    DATA STORAGE LAYER                        │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ PostgreSQL│ │ Redis    │  │ S3/Blob  │  │ Training │   │
│  │ (Metadata)│ │ (Cache)  │  │ (Logs)   │  │ Datasets │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Design Principles

1. **Offline-First**: Core scanning works without internet for 14 days (essential for bars with poor WiFi)
2. **Privacy-by-Design**: Zero PII retention, encrypted transit, GDPR/CCPA compliant
3. **Sub-3-Second Scans**: Optimized for bouncer workflow (1-2 sec target)
4. **Multi-State Support**: 50 US states + 200 countries, 10,000+ ID templates
5. **Hyperspectral Ready**: Modular architecture to add UV/IR analysis in Pro tier
6. **Scalable**: Support 1 bar or 10,000 bars on same infrastructure

---

## Technology Stack

### Mobile App (Frontend)

#### Option 1: Flutter (Cross-Platform) - **RECOMMENDED**
**Pros**:
- Single codebase for iOS + Android (50% faster development)
- Native performance via Dart AOT compilation
- Excellent camera/ML plugins (Google ML Kit, TensorFlow Lite)
- Strong offline support (SQLite, Hive local databases)
- Fast iteration (hot reload)

**Cons**:
- Larger app size (~50 MB vs. native 20 MB)
- Limited access to cutting-edge iOS features (but not needed for our use case)

**Tech Stack**:
- **Language**: Dart
- **Framework**: Flutter 3.x
- **Camera**: `camera` plugin + `google_ml_kit` for document detection
- **OCR**: `google_mlkit_text_recognition` (on-device)
- **Face Detection**: `google_mlkit_face_detection` (on-device)
- **Barcode Scanning**: `mobile_scanner` (fast, on-device)
- **Local Storage**: `hive` (NoSQL) or `sqflite` (SQLite)
- **API Client**: `dio` (HTTP client with retry logic)
- **State Management**: Riverpod or Bloc
- **Offline Queue**: `queue` package for background sync

**Development Timeline**:
- MVP (Basic tier features): 8-10 weeks
- Pro tier (hyperspectral integration): +4 weeks
- Enterprise (multi-location sync): +2 weeks

---

#### Option 2: Native iOS (Swift) + Native Android (Kotlin)
**Pros**:
- Best performance and platform integration
- Access to latest iOS Vision Framework, Android ML Kit
- Smaller app size

**Cons**:
- 2x development time (separate codebases)
- Higher maintenance cost

**Tech Stack**:
- **iOS**: Swift + SwiftUI, Vision Framework (document detection), Core ML (face recognition)
- **Android**: Kotlin + Jetpack Compose, CameraX, ML Kit, TensorFlow Lite

**Only choose if**: Extreme performance or iOS-specific features (e.g., Face ID integration) are critical.

---

### Backend (Cloud Services)

#### Core Services

**1. Face Recognition: CompreFace (Open-Source, MIT License)**
- **GitHub**: https://github.com/exadel-inc/CompreFace
- **Purpose**: Biometric face matching (ID photo → selfie), age detection
- **Deployment**: Docker container (CPU or GPU)
- **API Endpoints**:
  - `POST /api/v1/recognition/recognize` - 1:1 face verification
  - `POST /api/v1/detection/detect` - Face detection + age estimation
  - `POST /api/v1/recognition/subjects/{subject}/delete` - Privacy compliance (delete faces)

**Setup**:
```bash
# Quick start (CPU mode)
docker run -d -p 8000:8000 exadel/compreface:latest

# Production (GPU mode for speed)
docker run -d -p 8000:8000 --gpus all exadel/compreface-gpu:latest
```

**Configuration for BullshID**:
- **Model**: FaceNet or InsightFace (99.5% accuracy on LFW benchmark)
- **Threshold**: 0.85 similarity score (tune based on pilot false positive rate)
- **Age Estimation**: Enable for 21+ verification
- **Liveness Detection**: Add anti-spoofing model (proprietary or open-source like Silent-Face-Anti-Spoofing)

**Scaling**:
- Single server: 100 requests/second (CPU), 500 req/sec (GPU)
- Load balancer + 3 instances: 1,500 req/sec (handles 5,000 bars at peak)

---

**2. Document OCR: MinerU (Open-Source, MIT License)**
- **GitHub**: https://github.com/opendatalab/MinerU
- **Purpose**: Extract text from ID front/back (name, DOB, address, expiration)
- **Deployment**: Python API service (FastAPI + PyTorch)

**Alternative**: **Google Cloud Vision API** (paid, $1.50 per 1,000 requests) for production-grade accuracy

**Hybrid Approach** (Recommended):
- **On-Device OCR** (free, fast): Google ML Kit Text Recognition for simple IDs (clear prints)
- **Cloud OCR** (fallback): MinerU or Azure Document Intelligence for complex/damaged IDs

**Setup**:
```python
# MinerU API wrapper (FastAPI)
from fastapi import FastAPI, File, UploadFile
from mineru import DocumentExtractor

app = FastAPI()
extractor = DocumentExtractor()

@app.post("/ocr")
async def extract_text(file: UploadFile):
    text = extractor.extract(file)
    return {"text": text, "confidence": 0.95}
```

**Integration**:
- Mobile app sends ID images (front + back) to `/ocr` endpoint
- Response: Structured JSON (name, DOB, license number, state, expiration)
- Fallback: If on-device OCR confidence <90%, use cloud OCR

---

**3. Barcode/MRZ Validation: Custom + Open-Source Libraries**
- **Purpose**: Decode barcode (PDF417, Code 39) and MRZ (Machine-Readable Zone) from IDs, validate data consistency

**Open-Source Libraries**:
- **PDF417 Decoder**: `pypdf417` (Python) or `zxing` (Java/Android)
- **MRZ Parser**: `mrz` (Python, supports passports + ID cards)

**Proprietary Layer** (BullshID's IP):
- **Barcode Fraud Detection**:
  - Compare barcode data vs. OCR text (mismatch = fake)
  - Validate state-specific barcode formats (Texas DL has 8-digit license #, encoded in specific positions)
  - Check digit validation (mod-10 algorithm for checksum)
- **AAMVA Database Lookup** (Pro/Enterprise only):
  - Query state DMV databases for license validity (requires AAMVA membership, $10K+ annual fee)
  - Fallback: Compare against known fake barcode databases

**Setup**:
```python
# Barcode validation service
from pypdf417 import decode_pdf417
import json

def validate_texas_dl(barcode_data, ocr_data):
    parsed = decode_pdf417(barcode_data)

    # Compare key fields
    if parsed['license_number'] != ocr_data['license_number']:
        return {"valid": False, "reason": "barcode_mismatch"}

    # Texas-specific rules
    if len(parsed['license_number']) != 8:
        return {"valid": False, "reason": "invalid_format"}

    # Checksum validation
    if not validate_checksum(parsed['license_number']):
        return {"valid": False, "reason": "checksum_fail"}

    return {"valid": True}
```

---

**4. Hyperspectral Analysis: Proprietary + Custom Hardware**
**Purpose**: Analyze ID using UV/IR light to detect hologram authenticity, ink composition, hidden security features

**Hardware**:
- **iPad Pro with Custom Camera Filters**:
  - **UV Filter** (365nm): Detect UV-reactive inks, holograms
  - **IR Filter** (850nm): Detect near-infrared security features
  - **Cost**: $800/device (iPad + custom filters)

**Software** (Proprietary Algorithm):
```python
# Hyperspectral fraud detection
import cv2
import numpy as np

def analyze_hyperspectral(uv_image, ir_image, visible_image):
    # Step 1: Detect hologram presence (UV)
    uv_hologram_score = detect_hologram_pattern(uv_image)

    # Step 2: Ink spectral analysis (IR vs. visible)
    ink_authenticity = compare_ink_spectrum(ir_image, visible_image)

    # Step 3: Microprint analysis (high-res visible)
    microprint_valid = detect_microprint(visible_image)

    # Combine scores
    fraud_score = (
        0.4 * uv_hologram_score +
        0.4 * ink_authenticity +
        0.2 * microprint_valid
    )

    return {
        "fraud_probability": 1 - fraud_score,
        "confidence": 0.92,
        "details": {
            "hologram": uv_hologram_score > 0.7,
            "ink": ink_authenticity > 0.8,
            "microprint": microprint_valid
        }
    }

def detect_hologram_pattern(uv_image):
    # Edge detection + pattern matching
    edges = cv2.Canny(uv_image, 100, 200)
    templates = load_state_hologram_templates()  # TX, CA, NY, etc.

    max_match = 0
    for template in templates:
        match = cv2.matchTemplate(edges, template, cv2.TM_CCOEFF_NORMED)
        max_match = max(max_match, np.max(match))

    return max_match  # 0.0 (no match) to 1.0 (perfect match)
```

**Training Data**:
- Collect 100+ real IDs per state (UV/IR scans)
- Collect 50+ known fake IDs (UV/IR scans)
- Train CNN classifier: Real vs. Fake based on spectral signatures

**MVP Approach** (No Custom Hardware Initially):
- Use standard iPad camera + software-based fraud detection
- Pilot hyperspectral with 5 devices at Billy Bob's (test ROI)
- Scale if detection rate improves >5%

---

**5. Analytics Dashboard: Metabase (Open-Source)**
- **Purpose**: Venue manager dashboards (scan volume, detection trends, ROI reports)
- **GitHub**: https://github.com/metabase/metabase
- **Deployment**: Docker container + PostgreSQL

**Setup**:
```bash
docker run -d -p 3000:3000 \
  -e MB_DB_TYPE=postgres \
  -e MB_DB_HOST=db.bullshid.com \
  metabase/metabase
```

**Dashboards**:
1. **Daily Operations**: Scans per hour, fake IDs caught today, device status
2. **Weekly Trends**: Detection rate by day, peak hours, ID state distribution
3. **ROI Calculator**: Fines avoided, insurance savings, subscription cost comparison

---

### Database & Storage

**1. PostgreSQL (Metadata & Audit Logs)**
- **Schema**:
  ```sql
  CREATE TABLE scans (
    scan_id UUID PRIMARY KEY,
    venue_id VARCHAR(50),
    device_id VARCHAR(50),
    timestamp TIMESTAMP,
    id_type VARCHAR(20),  -- TX_DL, CA_DL, US_PASSPORT, etc.
    scan_result VARCHAR(20),  -- PASS, FAIL_FAKE, FAIL_EXPIRED, FAIL_UNDERAGE
    alert_reason TEXT,
    scan_duration_ms INT,
    bouncer_override BOOLEAN,
    override_reason TEXT
  );

  CREATE INDEX idx_venue_timestamp ON scans(venue_id, timestamp);
  CREATE INDEX idx_result ON scans(scan_result);
  ```

- **Data Retention**: 90 days for pilot, then delete (GDPR compliance)

**2. Redis (Caching & Rate Limiting)**
- **Use Cases**:
  - Cache state ID templates (reduce repeated downloads)
  - Rate limiting API requests (prevent abuse)
  - Session management for multi-device venues

**3. S3/Azure Blob Storage (Logs & Backups)**
- **Use Cases**:
  - Encrypted backups of scan metadata (compliance audits)
  - ML training datasets (anonymized ID images for model improvement)

---

### Security & Privacy

**1. Data Privacy Architecture**
**Zero PII Retention**:
- ❌ No names, addresses, photos stored in cloud
- ✅ Only metadata: ID type, scan result, timestamp, venue ID
- ✅ Face embeddings (mathematical vectors, not photos) stored temporarily for 1 scan only, then deleted

**Encryption**:
- **In Transit**: TLS 1.3 for all API calls (HTTPS)
- **At Rest**: AES-256 encryption for database (PostgreSQL with pgcrypto)
- **Face Data**: Ephemeral - processed in memory, never written to disk

**GDPR/CCPA Compliance**:
- **Right to Delete**: API endpoint to purge all venue data on request
- **Data Minimization**: Only collect what's needed for fraud detection
- **Consent**: Venues post signage: "IDs scanned for age verification"

**2. Authentication & Authorization**
- **Venue API Keys**: Each venue gets unique key (JWT tokens)
- **Device Registration**: iPads enrolled via QR code (prevent unauthorized devices)
- **Role-Based Access**: Bouncers (scan only), Managers (view analytics), Admins (full access)

**3. Fraud Prevention (Anti-Abuse)**
- **Rate Limiting**: 100 scans/minute per device (prevent bulk scraping)
- **Anomaly Detection**: Flag venues with >1,000 scans/hour (investigate)
- **Watermarking**: Invisible markers on scan results (prevent screenshot sharing)

---

## Open-Source Components (Summary)

| Component | License | Purpose | Integration Effort | Cost Savings |
|-----------|---------|---------|-------------------|-------------|
| **CompreFace** | MIT | Face recognition, age detection | Low (Docker deploy) | $50K+ (vs. AWS Rekognition) |
| **MinerU** | MIT | Document OCR | Medium (Python API) | $20K+ (vs. Google Vision) |
| **Flutter** | BSD | Mobile app framework | Medium (learn Dart) | $100K+ (vs. native iOS + Android) |
| **Metabase** | AGPL | Analytics dashboards | Low (Docker deploy) | $30K+ (vs. Tableau) |
| **PostgreSQL** | PostgreSQL | Metadata storage | Low (managed DB) | $10K+ (vs. Oracle) |
| **Redis** | BSD | Caching, rate limiting | Low (managed Redis) | $5K+ |
| **MIDV-2020** | Academic | ID training datasets | High (data prep) | $50K+ (vs. buying datasets) |
| **DocFace+** | GitHub | ID-to-selfie matching | High (TensorFlow) | $30K+ (vs. custom ML) |
| **Total Savings** | | | | **$295K+** in Year 1 |

---

## Proprietary Innovation (BullshID IP)

### 1. Hyperspectral Fraud Detection
- **Algorithm**: CNN trained on UV/IR spectral signatures of real vs. fake IDs
- **Patent Potential**: "Method for mobile hyperspectral ID verification" (file provisional patent)
- **Competitive Moat**: No existing mobile app does hyperspectral analysis

### 2. Multi-State Barcode Validation
- **Database**: 50-state barcode format rules (Texas 8-digit, California 7-alpha, etc.)
- **Checksum Algorithms**: State-specific validation logic (proprietary reverse-engineering)
- **AAMVA Integration**: Official database queries (requires licensing, high barrier to entry)

### 3. Bouncer UX Optimization
- **One-Handed Scanning**: Custom camera workflow (optimized for iPad held in one hand)
- **Offline-First Sync**: Intelligent queueing (prioritize alerts over routine passes when syncing)
- **Dark Mode**: Optimized for nighttime use (reduce screen glare in dark bars)

### 4. Fraud Pattern Database
- **Crowdsourced Intelligence**: Aggregate fake ID patterns across all venues (anonymized)
- **Real-Time Alerts**: "New fake Texas DL template detected at Billy Bob's" → alert all Texas bars
- **Network Effects**: Value increases with more venues (shared fraud database)

---

## Development Roadmap

### Phase 1: MVP (Months 1-3) - Basic Tier Features
**Goal**: Functional pilot for 4 Texas venues

**Deliverables**:
1. **Mobile App (Flutter)**:
   - Camera capture (front + back ID, selfie)
   - On-device OCR (Google ML Kit)
   - Barcode scanning (mobile_scanner)
   - Offline storage (Hive, 14-day cache)
   - Pass/Fail/Alert UI
   - iOS + Android builds

2. **Backend Services**:
   - CompreFace deployment (Docker, AWS EC2)
   - Barcode validation API (Python FastAPI)
   - PostgreSQL database setup
   - Basic analytics (Metabase)

3. **Integration**:
   - API client in Flutter (Dio)
   - Background sync worker (queue)
   - Device registration flow

**Testing**:
- Unit tests (80% code coverage)
- Integration tests (API endpoints)
- User acceptance testing with 5 beta bouncers

**Timeline**: 12 weeks (3 months)
**Team**: 2 mobile devs, 1 backend dev, 1 ML engineer, 1 QA

---

### Phase 2: Pro Tier (Months 4-6) - Hyperspectral Beta
**Goal**: Add advanced fraud detection for premium customers

**Deliverables**:
1. **Hyperspectral Analysis**:
   - UV/IR camera filter integration (iPad Pro)
   - Spectral analysis algorithm (Python + OpenCV)
   - Hologram detection model (CNN)
   - Train on 500 real + 100 fake IDs

2. **Liveness Detection**:
   - Anti-spoofing model (prevent photo-of-photo attacks)
   - Integrate Silent-Face-Anti-Spoofing (open-source)

3. **AAMVA Integration** (if budget allows):
   - Partner with AAMVA for database access ($10K+ annual)
   - Query API for license validation (Texas, California, etc.)

**Timeline**: 12 weeks
**Team**: +1 computer vision engineer, +1 data scientist

---

### Phase 3: Enterprise Tier (Months 7-9) - Multi-Location & API
**Goal**: Enable venue chains and custom integrations

**Deliverables**:
1. **Multi-Location Sync**:
   - Centralized admin dashboard (React web app)
   - 86-list sharing across venues (Redis Pub/Sub)
   - Unified analytics (Metabase multi-tenant)

2. **API Platform**:
   - RESTful API for POS integration (Toast, Square)
   - Webhooks for real-time alerts (Slack, email)
   - GraphQL for custom queries

3. **White-Label**:
   - Custom branding (venue logos, colors)
   - Domain mapping (riotroom.bullshid.com)

**Timeline**: 12 weeks
**Team**: +1 backend dev, +1 frontend dev

---

### Phase 4: National Scale (Months 10-12) - Infrastructure & Ops
**Goal**: Support 100+ venues, optimize costs

**Deliverables**:
1. **Infrastructure**:
   - Kubernetes deployment (auto-scaling)
   - Multi-region (US-East, US-West for low latency)
   - CDN for ID templates (CloudFront)

2. **Cost Optimization**:
   - On-device ML (reduce cloud API calls by 80%)
   - Spot instances for batch processing
   - Aggressive caching (Redis)

3. **Monitoring**:
   - Prometheus + Grafana (uptime, latency)
   - Sentry (error tracking)
   - PagerDuty (on-call alerts)

**Timeline**: 12 weeks
**Team**: +1 DevOps engineer, +1 SRE

---

## Cost Estimation

### Development Costs (Year 1)

| Phase | Duration | Team | Estimated Cost |
|-------|----------|------|---------------|
| Phase 1 (MVP) | 3 months | 5 FTEs | $150K (salaries + infra) |
| Phase 2 (Pro) | 3 months | 7 FTEs | $180K |
| Phase 3 (Enterprise) | 3 months | 9 FTEs | $200K |
| Phase 4 (Scale) | 3 months | 10 FTEs | $220K |
| **Total** | **12 months** | | **$750K** |

**Funding Sources**:
- Bootstrapping: $200K (founders)
- Angel Round: $300K (pitch with pilot results)
- Revenue (Year 1): $250K (assume 50 bars @ $5K average ARR)

---

### Infrastructure Costs (Year 1)

| Service | Provider | Monthly Cost | Annual Cost |
|---------|----------|-------------|-------------|
| Cloud Hosting (Compute) | AWS EC2 (3 instances) | $500 | $6,000 |
| CompreFace (GPU) | AWS EC2 g4dn.xlarge | $400 | $4,800 |
| Database | AWS RDS (PostgreSQL) | $200 | $2,400 |
| Redis | AWS ElastiCache | $100 | $1,200 |
| Storage (S3) | AWS S3 (1 TB) | $25 | $300 |
| CDN | CloudFront | $50 | $600 |
| Monitoring | Datadog | $100 | $1,200 |
| **Total** | | **$1,375/month** | **$16,500** |

**At Scale (1,000 bars)**:
- Cloud costs: ~$5K/month ($60K/year)
- Gross margin: 70% (typical SaaS)

---

## Deployment Architecture

### Production Stack (AWS)
```
┌─────────────────────────────────────────────────────────────┐
│                     CloudFront CDN                          │
│              (ID templates, static assets)                  │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                   Application Load Balancer                 │
│              (HTTPS termination, auto-scaling)              │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────┬──────────────────┬──────────────────────┐
│   EC2 Instance   │   EC2 Instance   │   EC2 Instance      │
│   (API Server)   │   (API Server)   │   (API Server)      │
│   Flask/FastAPI  │   Flask/FastAPI  │   Flask/FastAPI     │
└──────────────────┴──────────────────┴──────────────────────┘
         ↓                   ↓                     ↓
┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐
│  CompreFace GPU  │  │ Barcode Service  │  │ Metabase    │
│  (Face recog.)   │  │ (Validation)     │  │ (Analytics) │
└──────────────────┘  └──────────────────┘  └──────────────┘
         ↓                   ↓                     ↓
┌─────────────────────────────────────────────────────────────┐
│                    RDS PostgreSQL                           │
│              (Metadata, user accounts)                      │
└─────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────┐
│                    ElastiCache Redis                        │
│              (Caching, rate limiting)                       │
└─────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────┐
│                    S3 Buckets                               │
│         (Logs, backups, training datasets)                  │
└─────────────────────────────────────────────────────────────┘
```

### Kubernetes Option (Future Scale)
- **EKS** (Elastic Kubernetes Service) for auto-scaling
- **Helm charts** for deployment automation
- **Horizontal Pod Autoscaler** (scale based on CPU/requests)

---

## Testing Strategy

### Unit Tests
- **Coverage**: 80%+
- **Tools**: Jest (Flutter), Pytest (Python)
- **Focus**: Business logic (barcode validation, age calculation)

### Integration Tests
- **API Tests**: Postman collections (test all endpoints)
- **Database Tests**: Test data integrity, migrations

### End-to-End Tests
- **Mobile**: Appium or Maestro (automated UI testing)
- **Scenarios**: Complete scan flow (camera → OCR → face match → result)

### Load Testing
- **Tool**: k6 or Locust
- **Target**: 500 req/sec sustained (peak 9 PM - 2 AM on Saturdays)

### Security Testing
- **Penetration Testing**: Hire ethical hackers (Year 1, post-launch)
- **Dependency Scanning**: Snyk for vulnerabilities
- **OWASP Top 10**: Manual checklist (SQL injection, XSS, etc.)

---

## MVP Technical Checklist

### Mobile App (Flutter)
- [ ] Camera integration (front + back ID, selfie)
- [ ] On-device OCR (Google ML Kit)
- [ ] Barcode scanning (PDF417, Code 39)
- [ ] Face detection (bounding box)
- [ ] API client (POST scan data to backend)
- [ ] Offline storage (Hive, 1,000 scans)
- [ ] Background sync (queue worker)
- [ ] Pass/Fail/Alert UI (green/red/yellow screens)
- [ ] Device registration (QR code)
- [ ] iOS build (TestFlight)
- [ ] Android build (Google Play beta)

### Backend Services
- [ ] CompreFace deployment (Docker on AWS EC2)
- [ ] Barcode validation API (FastAPI)
- [ ] OCR fallback service (MinerU or Azure)
- [ ] PostgreSQL database (RDS, schema v1)
- [ ] Redis cache (ElastiCache)
- [ ] API authentication (JWT)
- [ ] Rate limiting (100 req/min per device)
- [ ] Logging (CloudWatch)
- [ ] Metabase dashboard (Docker)

### DevOps
- [ ] GitHub repo (private)
- [ ] CI/CD pipeline (GitHub Actions → AWS)
- [ ] Staging environment (test before prod)
- [ ] Monitoring (Datadog or Prometheus)
- [ ] Backups (automated PostgreSQL snapshots)
- [ ] Secrets management (AWS Secrets Manager)

### Documentation
- [ ] API docs (Swagger/OpenAPI)
- [ ] Mobile app developer guide
- [ ] Deployment runbook
- [ ] Incident response plan

---

## Open Questions & Decisions

### 1. Mobile Framework
**Decision**: Flutter (cross-platform) vs. Native (iOS + Android)
**Recommendation**: **Flutter** (faster MVP, 90% of features supported)
**Rationale**: Save 3 months development time, acceptable performance trade-off

### 2. Face Recognition
**Decision**: Self-hosted (CompreFace) vs. Cloud (AWS Rekognition)
**Recommendation**: **Self-hosted CompreFace** (cost savings, data privacy)
**Rationale**: $50K+ savings/year, better GDPR compliance

### 3. OCR Provider
**Decision**: On-device (ML Kit) vs. Cloud (Azure Document Intelligence)
**Recommendation**: **Hybrid** (on-device primary, cloud fallback)
**Rationale**: 80% of IDs work with free ML Kit, 20% complex cases use Azure ($200/month)

### 4. Hyperspectral Hardware
**Decision**: Launch with or delay until Pro tier?
**Recommendation**: **Delay until Phase 2** (validate Basic tier demand first)
**Rationale**: $6,500 hardware investment for pilot; test ROI before scaling

### 5. AAMVA Database
**Decision**: Integrate now ($10K+ annual fee) or later?
**Recommendation**: **Later** (post-pilot, if Pro tier adoption >30%)
**Rationale**: Expensive, not needed for MVP fraud detection

---

## Risk Assessment

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| CompreFace accuracy <95% | Medium | High | Test thoroughly, fallback to AWS Rekognition if needed |
| Flutter app performance issues | Low | Medium | Optimize with profiling, switch to native if critical |
| GDPR compliance violation | Low | Critical | Legal review, zero PII retention, encryption |
| Hyperspectral hardware too expensive | Medium | Medium | Start with software-only, add hardware in Phase 2 |
| AAMVA denies database access | Medium | Medium | Build proprietary barcode database instead |
| Cloud costs exceed budget | Medium | Medium | Optimize with caching, on-device ML, spot instances |

---

## Next Steps

1. **Week 1**: Finalize tech stack decisions (Flutter vs. native, CompreFace config)
2. **Week 2**: Set up development environment (GitHub, AWS accounts, Docker)
3. **Week 3-4**: Build mobile app skeleton (camera, basic UI)
4. **Week 5-6**: Integrate CompreFace (face matching, age detection)
5. **Week 7-8**: Barcode validation + OCR integration
6. **Week 9-10**: Offline mode + background sync
7. **Week 11-12**: Testing, bug fixes, pilot deployment

**First Commit Target**: Week 3 (basic Flutter app with camera)

---

## Contact

**Technical Questions**: dev@bullshid.com
**Architecture Review**: CTO@bullshid.com (schedule 1-hour deep dive)

---

**Build fast. Ship faster. Iterate always.**

**© 2025 BullshID. Open-source powered, proprietary protected.**
