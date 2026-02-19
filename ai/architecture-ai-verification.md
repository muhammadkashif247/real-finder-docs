# RealFinder AI Verification -- System Architecture

> **Author**: Solution Architect
> **Created**: 2026-02-12
> **Status**: Design
> **Covers**: TL1 (Listing), TL2 (Broker), TL3 (Property) AI Verification

---

## 1. Executive Summary

The AI verification system is a **separate Python FastAPI microservice** (`realfinder-ai`) that the existing NestJS backend calls via REST API. It uses Google Gemini for text and image analysis to verify listings, brokers, and properties across three trust levels.

**Key design principles:**
- **Stateless AI service** -- receives data, processes, returns results. No database access.
- **Backend owns all state** -- NestJS handles database writes, status updates, audit trail.
- **Single responsibility** -- Python service does AI only, backend does everything else.
- **Horizontal scalability** -- AI service scales independently from the backend.
- **Graceful degradation** -- if AI service is down, listings queue for manual admin review instead of blocking.

---

## 2. Current System (Before AI)

```
                    +------------------+
                    |   Next.js 16     |
                    |   Frontend       |
                    +--------+---------+
                             |
                             | REST API
                             v
                    +------------------+
                    |   NestJS Backend |
                    |   (api/v1)       |
                    +----+-------+-----+
                         |       |
                    +----+   +---+----+
                    |        |        |
                    v        v        v
               +------+ +------+ +------+
               |Postgres| |Redis | |  S3  |
               |  DB   | |Cache | |Files |
               +------+ +------+ +------+
```

**What happens today when a broker submits a listing:**
1. Broker fills out listing form in Next.js frontend
2. Frontend calls NestJS `POST /api/v1/listings`
3. NestJS saves listing (status: PAUSED)
4. Broker clicks "Submit for Approval"
5. NestJS sets status to PENDING
6. **Admin manually reviews every listing** (bottleneck)
7. Admin approves --> ACTIVE + VISIBLE, or rejects --> stays PAUSED

**The problem:** Every listing requires manual admin review. This doesn't scale.

---

## 3. Target System (With AI)

```
                    +------------------+
                    |   Next.js 16     |
                    |   Frontend       |
                    +--------+---------+
                             |
                             | REST API
                             v
                    +------------------+        +------------------+
                    |   NestJS Backend |  REST  | Python FastAPI   |
                    |   (api/v1)       +------->+  AI Service      |
                    |                  |<-------+  (realfinder-ai) |
                    +----+-------+-----+        +--------+---------+
                         |       |                       |
                    +----+   +---+----+                  |
                    |        |        |                  |
                    v        v        v                  v
               +------+ +------+ +------+        +----------+
               |Postgres| |Redis | |  S3  |        | Gemini   |
               |  DB   | |Cache | |Files |        |   API    |
               +------+ +------+ +------+        +----------+
```

**What changes:**
- A new Python FastAPI service sits alongside the backend
- NestJS calls it via HTTP when verification is needed
- Python service calls Gemini API, processes results, returns decisions
- NestJS saves results and updates statuses
- Admins only review flagged cases (not every listing)

---

## 4. Detailed Architecture

### 4.1 Component Diagram

```
+===========================================================================+
|                          REALFINDER PLATFORM                              |
|                                                                           |
|  +---------------------------+      +----------------------------------+  |
|  |      NestJS Backend       |      |     Python AI Service            |  |
|  |      (Port 3001)          |      |     (Port 8000)                  |  |
|  |                           |      |                                  |  |
|  |  +---------------------+ |      |  +----------------------------+  |  |
|  |  | Listing Module      | |      |  | /api/v1/verify/listing     |  |  |
|  |  | - submitForApproval | |----->|  | - Text rules engine        |  |  |
|  |  | - changeStatus      | |      |  | - Gemini text analysis     |  |  |
|  |  | - admin endpoints   | |      |  | - Gemini image analysis    |  |  |
|  |  +---------------------+ |      |  | - Score combiner           |  |  |
|  |                           |      |  +----------------------------+  |  |
|  |  +---------------------+ |      |                                  |  |
|  |  | Broker Module       | |      |  +----------------------------+  |  |
|  |  | - submitVerification| |----->|  | /api/v1/verify/broker      |  |  |
|  |  | - admin review      | |      |  | - Document OCR             |  |  |
|  |  +---------------------+ |      |  | - License validation       |  |  |
|  |                           |      |  | - Fraud detection          |  |  |
|  |  +---------------------+ |      |  +----------------------------+  |  |
|  |  | Housebook Module    | |      |                                  |  |
|  |  | - verifyProperty    | |----->|  +----------------------------+  |  |
|  |  | - resolveConflicts  | |      |  | /api/v1/verify/property    |  |  |
|  |  +---------------------+ |      |  | - Ownership doc analysis   |  |  |
|  |                           |      |  | - Metadata consistency     |  |  |
|  |  +---------------------+ |      |  | - Media authenticity       |  |  |
|  |  | Audit Module        | |      |  | - Conflict detection       |  |  |
|  |  | - AI event logging  | |      |  +----------------------------+  |  |
|  |  +---------------------+ |      |                                  |  |
|  |                           |      |  +----------------------------+  |  |
|  |  +---------------------+ |      |  | Shared Components          |  |  |
|  |  | Prisma ORM          | |      |  | - Gemini client wrapper    |  |  |
|  |  | - All DB writes     | |      |  | - Rate limiter             |  |  |
|  |  | - All DB reads      | |      |  | - Retry logic              |  |  |
|  |  +---------------------+ |      |  | - Health check             |  |  |
|  +---------------------------+      |  +----------------------------+  |  |
|                                     +----------------------------------+  |
+===========================================================================+
```

### 4.2 Why This Architecture?

| Decision | Reasoning |
|----------|-----------|
| **Separate Python service** | Python has the best AI/ML ecosystem. Gemini SDK, image processing, NLP libraries are all Python-first. Keeping AI in Python and business logic in NestJS plays to each language's strengths. |
| **REST over message queue** | Simpler to build, debug, and deploy for MVP. No extra infrastructure (RabbitMQ/Kafka). NestJS can call Python async (non-blocking) without a queue. Can add queue later if volume demands it. |
| **Stateless Python service** | No database access means: easy horizontal scaling (just add more instances), no connection pool issues, no migration conflicts, no shared state bugs. The backend is the single source of truth. |
| **Same Gemini key for all TLs** | One API key covers text + vision for all three trust levels. Simplifies key management. Free tier (60 req/min) is enough for MVP. |
| **Python reads S3 directly** | Images are stored in S3 with accessible URLs. Python fetches them via HTTP -- no need to proxy through NestJS. Reduces backend load and latency. |
| **NestJS owns all DB writes** | Prevents race conditions, keeps audit trail consistent, simplifies transaction management. Python never touches the DB. |

---

## 5. Communication Design

### 5.1 Service-to-Service Authentication

```
NestJS                                  Python AI Service
  |                                          |
  |  POST /api/v1/verify/listing             |
  |  Headers:                                |
  |    X-Service-Key: <shared-secret>        |
  |    Content-Type: application/json        |
  |----------------------------------------->|
  |                                          |  Validate X-Service-Key
  |                                          |  Process request
  |  200 OK                                  |
  |  { decision, score, details }            |
  |<-----------------------------------------|
```

**Why a shared secret (not JWT)?**
- This is internal service-to-service communication, not user-facing
- Both services are in the same network / cluster
- A simple API key in the header is sufficient and avoids token rotation complexity
- The key is stored in environment variables on both sides

### 5.2 Request/Response Contract

**NestJS sends:**
- All data the Python service needs (listing text, image URLs, attributes, etc.)
- No database IDs that Python would need to look up -- everything is inline

**Python returns:**
- Decision (APPROVE / FLAG / REJECT)
- Combined confidence score (0.00 - 1.00)
- Per-check breakdown (text, image, OCR, etc.)
- Specific issues found (human-readable)
- Execution time per check

**NestJS then:**
- Saves the full response as JSON in the `details` column
- Creates `AiEvent` + `AiDecision` records for audit trail
- Updates entity status based on the decision

### 5.3 Error Handling Contract

| HTTP Status | Meaning | NestJS Action |
|-------------|---------|---------------|
| 200 | Success | Process result normally |
| 400 | Bad request (missing fields) | Log error, mark verification as ERROR |
| 408 | Timeout | Retry once, then queue for manual review |
| 429 | Rate limited (Gemini quota) | Retry after delay from Retry-After header |
| 500 | Python service error | Retry once, then queue for manual review |
| Connection refused | Service down | Queue for manual review, alert ops team |

**Critical rule:** AI service failure must NEVER block listing submission. If AI is unavailable, the listing stays PENDING for manual admin review (same as today).

---

## 6. Python AI Service -- Internal Architecture

```
realfinder-ai/
|
+-- app/
|   +-- main.py                  # FastAPI app, middleware, startup
|   +-- config.py                # Environment config (pydantic-settings)
|   +-- dependencies.py          # Dependency injection
|   |
|   +-- api/
|   |   +-- v1/
|   |       +-- router.py        # All route registrations
|   |       +-- verify.py        # /verify/listing, /verify/broker, /verify/property
|   |       +-- analyze.py       # /analyze/text, /analyze/images, /analyze/document
|   |       +-- detect.py        # /detect/conflicts
|   |       +-- health.py        # /health
|   |
|   +-- services/
|   |   +-- gemini_client.py     # Gemini SDK wrapper (text + vision)
|   |   +-- text_analyzer.py     # Rules engine + Gemini text analysis
|   |   +-- image_analyzer.py    # Image validation + Gemini vision analysis
|   |   +-- document_ocr.py      # Document OCR + data extraction
|   |   +-- license_validator.py # Country-specific license format rules
|   |   +-- fraud_detector.py    # Duplicate + tampering detection
|   |   +-- consistency_checker.py # Metadata comparison logic
|   |   +-- conflict_detector.py # Cross-record conflict detection
|   |   +-- scoring.py           # Score combination + decision logic
|   |
|   +-- models/
|   |   +-- requests.py          # Pydantic request models
|   |   +-- responses.py         # Pydantic response models
|   |   +-- enums.py             # Decision, Status enums
|   |
|   +-- core/
|       +-- rate_limiter.py      # Gemini API rate limiting
|       +-- retry.py             # Retry with exponential backoff
|       +-- exceptions.py        # Custom exceptions
|       +-- middleware.py        # Auth middleware, logging, timing
|
+-- tests/
|   +-- test_text_analyzer.py
|   +-- test_image_analyzer.py
|   +-- test_document_ocr.py
|   +-- test_verify_listing.py
|   +-- test_verify_broker.py
|   +-- test_verify_property.py
|
+-- docker/
|   +-- Dockerfile
|   +-- Dockerfile.dev
|
+-- requirements.txt
+-- .env.example
+-- README.md
```

### 6.1 Gemini Client Wrapper

The Gemini client is the most critical shared component. It wraps the `google-generativeai` SDK and provides:

```
+-------------------------------------------+
|           GeminiClient                     |
|                                            |
|  - analyze_text(prompt) -> GeminiResponse  |
|  - analyze_image(prompt, images) -> ...    |
|  - analyze_document(prompt, doc) -> ...    |
|                                            |
|  Built-in:                                 |
|  - Rate limiting (token bucket, 60/min)    |
|  - Retry (3 attempts, exponential backoff) |
|  - Timeout (30s per request)               |
|  - Error classification                    |
|  - Response parsing + validation           |
|  - Execution time tracking                 |
+-------------------------------------------+
```

**Why wrap the SDK?**
- Every service (text, image, OCR) uses Gemini differently but shares rate limiting and retry logic
- Centralizes error handling -- if Gemini changes their API, we fix one file
- Makes testing easy -- mock the wrapper, not the SDK

### 6.2 Rate Limiting Strategy

Gemini free tier: 60 requests/minute, 1500 requests/day.

```
Request comes in
  --> Check token bucket (60 tokens, refill 1/sec)
      --> Tokens available? Process immediately.
      --> No tokens? Wait up to 5 seconds.
          --> Still no tokens? Return 429 to NestJS.
              --> NestJS retries after delay or queues for manual review.
```

**Why token bucket?**
- Simple, predictable, no external dependencies
- Works per-instance (each Python instance gets its own bucket)
- For multiple instances: divide quota (e.g. 3 instances = 20 req/min each)
- In production with paid tier, limits are much higher and this becomes a safety net

### 6.3 Retry Strategy

```
Call Gemini API
  --> Success? Return result.
  --> Transient error (500, timeout, rate limit)?
      --> Retry 1: wait 1 second
      --> Retry 2: wait 3 seconds
      --> Retry 3: wait 9 seconds
      --> All failed? Return error with details.
  --> Permanent error (400, invalid request)?
      --> Don't retry, return error immediately.
```

**Why exponential backoff?**
- Gemini rate limits recover quickly -- waiting a few seconds usually works
- Prevents thundering herd if multiple requests fail at once
- 3 retries with backoff covers most transient issues without excessive delay

---

## 7. Data Flow -- Detailed per Trust Level

### 7.1 TL1: Listing Verification

```
Broker clicks "Submit for Approval"
        |
        v
+------------------+
| NestJS Backend   |
|                  |
| 1. Validate      |  (listing must be PAUSED)
| 2. Set PENDING   |  (marketStatus = PENDING, visibility = SYSTEM_HIDDEN)
| 3. Load listing  |  (title, description, price, images, attributes)
| 4. HTTP POST     |-----> Python AI Service
|                  |       |
|                  |       | POST /api/v1/verify/listing
|                  |       |   { title, description, price, image_urls, ... }
|                  |       |
|                  |       v
|                  |  +--------------------------+
|                  |  | Text Analysis            |
|                  |  | 1. Rules engine (regex)  |  <-- free, instant
|                  |  | 2. Gemini Pro call       |  <-- ~1-2 sec
|                  |  | 3. Combine scores        |
|                  |  +--------------------------+
|                  |       |  (runs in parallel)
|                  |  +--------------------------+
|                  |  | Image Analysis           |
|                  |  | 1. URL validation        |  <-- free, instant
|                  |  | 2. Fetch images from S3  |  <-- ~1 sec
|                  |  | 3. Gemini Vision call    |  <-- ~2-5 sec
|                  |  | 4. Per-image scoring     |
|                  |  +--------------------------+
|                  |       |
|                  |       v
|                  |  +--------------------------+
|                  |  | Score Combiner           |
|                  |  | text: 50%, image: 50%    |
|                  |  | decision = APPROVE/FLAG/ |
|                  |  |            REJECT        |
|                  |  +--------------------------+
|                  |       |
|                  |<------+  { decision, score, details }
|                  |
| 5. Save results  |  (ListingVerification records)
| 6. Save audit    |  (AiEvent + AiDecision)
| 7. Update status |
|    APPROVE:      |  verificationStatus=VERIFIED, ACTIVE, VISIBLE
|    FLAG:         |  verificationStatus=FLAGGED, stays PENDING
|    REJECT:       |  verificationStatus=REJECTED, stays PAUSED
+------------------+
```

**Total latency: ~3-8 seconds** (text + image in parallel)

### 7.2 TL2: Broker Verification

```
Broker uploads license document
        |
        v
+------------------+
| NestJS Backend   |
|                  |
| 1. Save doc to S3|
| 2. Create        |  BrokerVerification (status: PENDING)
| 3. HTTP POST     |-----> Python AI Service
|                  |       |
|                  |       | POST /api/v1/verify/broker
|                  |       |   { broker_name, license_number, country, doc_urls }
|                  |       |
|                  |       v
|                  |  +--------------------------+
|                  |  | Document OCR             |
|                  |  | Gemini Vision reads doc  |  <-- ~3-5 sec
|                  |  | Extracts: license#,      |
|                  |  |   name, authority, expiry |
|                  |  | Compares to submitted    |
|                  |  +--------------------------+
|                  |       |
|                  |  +--------------------------+
|                  |  | License Format Validation|
|                  |  | Country-specific regex   |  <-- instant
|                  |  | Pattern matching         |
|                  |  +--------------------------+
|                  |       |
|                  |  +--------------------------+
|                  |  | Fraud Detection          |
|                  |  | Duplicate license check  |  <-- instant
|                  |  | Gemini: doc tampering    |  <-- ~2-3 sec
|                  |  +--------------------------+
|                  |       |
|                  |<------+  { decision, score, ocr_data, issues }
|                  |
| 4. Save results  |  (AI fields on BrokerVerification)
| 5. Save audit    |  (AiEvent + AiDecision)
| 6. Update status |
|    APPROVE:      |  BrokerVerification.status = VERIFIED
|    FLAG:         |  stays PENDING for admin
|    REJECT:       |  BrokerVerification.status = REVOKED
+------------------+
```

**Total latency: ~5-10 seconds**

### 7.3 TL3: Property Verification

```
Admin triggers verification / User submits ownership claim
        |
        v
+------------------+
| NestJS Backend   |
|                  |
| 1. Collect data  |  (HouseBook, listings, claims, documents)
| 2. HTTP POST     |-----> Python AI Service
|                  |       |
|                  |       | POST /api/v1/verify/property
|                  |       |   { housebook, listings[], claims[], doc_urls[] }
|                  |       |
|                  |       v
|                  |  +--------------------------+
|                  |  | Document Analysis        |  <-- ~3-5 sec
|                  |  | (title deed OCR)         |
|                  |  +--------------------------+
|                  |  | Metadata Consistency     |  <-- ~1-2 sec
|                  |  | (listing vs HouseBook)   |
|                  |  +--------------------------+
|                  |  | Media Authenticity       |  <-- ~3-5 sec
|                  |  | (photo verification)     |
|                  |  +--------------------------+
|                  |  | Conflict Detection       |  <-- ~1-2 sec
|                  |  | (duplicate claims/list.) |
|                  |  +--------------------------+
|                  |       |
|                  |<------+  { decision, score, per_check_details }
|                  |
| 3. Save results  |
| 4. Save audit    |
| 5. Update status |
|    APPROVE:      |  Property gets TL3 badge
|    FLAG:         |  Queued for admin
|    REJECT:       |  Verification denied
+------------------+
```

**Total latency: ~8-15 seconds** (most checks run in parallel)

---

## 8. Scalability Design

### 8.1 Current Scale (MVP)

```
        +-------------+         +-------------+
        |  NestJS x1  |-------->| Python x1   |
        +------+------+         +------+------+
               |                       |
        +------+------+         +------+------+
        | Postgres x1 |         | Gemini API  |
        +-------------+         | (free tier) |
                                +-------------+
```

- Single NestJS instance
- Single Python instance
- Free Gemini tier (60 req/min)
- Good for: < 100 listings/day

### 8.2 Growth Scale

```
                    +------------------+
                    |  Load Balancer   |
                    +--------+---------+
                             |
                +------------+------------+
                |                         |
        +-------+------+         +-------+------+
        |  NestJS x2   |         | Python x3    |
        +-------+------+         +-------+------+
                |                         |
        +-------+------+         +-------+------+
        | Postgres     |         | Gemini API   |
        | (read replicas)|       | (paid tier)  |
        +-------------+         +-------------+
```

- Multiple Python instances behind load balancer
- Python scales independently (CPU/memory bound by image processing)
- Paid Gemini tier (1000+ req/min)
- Good for: 100-1000 listings/day

### 8.3 Why This Scales

| Component | Scaling Strategy | Reasoning |
|-----------|-----------------|-----------|
| Python AI Service | Horizontal (add instances) | Stateless, no shared state, each instance handles requests independently |
| Gemini API | Upgrade tier | Free -> Pay-as-you-go. Linear cost scaling. No code changes needed. |
| NestJS Backend | Horizontal (add instances) | Already stateless (session in Redis, DB in Postgres) |
| PostgreSQL | Read replicas + connection pooling | AI writes go through NestJS (single writer), reads can be distributed |
| S3 | Already scalable | AWS handles this. Python reads images via HTTP -- no bottleneck. |
| Redis | Already scalable | Used for caching. AI doesn't touch Redis directly. |

### 8.4 Future Scale (if needed)

If request volume exceeds what synchronous REST can handle:

```
NestJS --> Redis Queue --> Python Workers (consume from queue)
                              |
                              +--> Process verification
                              +--> POST result back to NestJS webhook
```

**When to add a queue:**
- More than 1000 verifications/hour sustained
- Gemini response times exceed 30 seconds regularly
- Need to batch/schedule verifications (e.g. nightly re-verification)

**This is NOT needed for MVP.** The sync REST approach handles the expected volume easily.

---

## 9. Reliability & Fault Tolerance

### 9.1 Circuit Breaker Pattern

```
NestJS calls Python AI Service
  |
  +--> Circuit CLOSED (normal):
  |      Send request, get response.
  |      Track failures.
  |
  +--> 5 consecutive failures:
  |      Circuit OPEN.
  |      Don't call Python service for 60 seconds.
  |      All listings go to manual admin review (PENDING).
  |      Alert ops team.
  |
  +--> After 60 seconds:
         Circuit HALF-OPEN.
         Send one test request.
         Success? Close circuit, resume normal.
         Failure? Stay open, wait another 60 seconds.
```

**Why:** Prevents cascading failures. If Python service is down or Gemini is broken, we don't keep hammering it. Listings still work -- they just go to manual review (same as pre-AI behavior).

### 9.2 Timeout Strategy

| Call | Timeout | Reasoning |
|------|---------|-----------|
| NestJS --> Python | 30 seconds | Image analysis can take time. 30s covers worst case. |
| Python --> Gemini (text) | 15 seconds | Text analysis is fast. 15s is generous. |
| Python --> Gemini (vision) | 20 seconds | Vision analysis with multiple images needs more time. |
| Python --> S3 (image fetch) | 10 seconds | S3 is fast. If 10s isn't enough, the URL is probably broken. |

### 9.3 Graceful Degradation

```
Full AI available:
  --> Text + Image analysis --> Auto-decision

Gemini Vision down, Gemini Text works:
  --> Text analysis only --> Partial score
  --> Flag for admin (can't fully verify without images)

Gemini completely down:
  --> Rules engine only (free, no API call)
  --> Flag for admin (rules catch obvious fakes, rest goes to manual)

Python service completely down:
  --> Listing goes to PENDING
  --> Admin reviews manually (pre-AI behavior)
  --> System is degraded but NOT broken
```

**Critical principle:** The AI system must NEVER block the core listing workflow. If anything fails, the fallback is always manual admin review.

---

## 10. Security

### 10.1 Service Authentication

```
NestJS --> Python:
  Header: X-Service-Key: <AI_SERVICE_API_KEY>
  
Python validates:
  if request.headers['X-Service-Key'] != config.SERVICE_API_KEY:
      return 401 Unauthorized
```

- Shared secret stored in environment variables on both services
- Rotated periodically via deployment configuration
- NOT the same as the public `API_KEY` used by the frontend

### 10.2 Network Security

| Rule | Implementation |
|------|---------------|
| Python service NOT exposed to internet | Only accessible from NestJS (internal network / Docker network) |
| No direct database access from Python | Python has no DATABASE_URL. Cannot read or write the DB. |
| S3 images accessed via pre-signed URLs or public read | Python fetches images over HTTPS. No AWS credentials needed if URLs are pre-signed. |
| Gemini API key only on Python service | NestJS never touches Gemini. Key rotation only affects one service. |
| No user data logged | AI service logs decisions but never logs full listing text or image content |

### 10.3 Data Flow Security

```
User data flows:  Frontend --> NestJS --> Python --> Gemini
                                    (HTTPS)   (HTTPS)

What Python receives:  Listing text, image URLs, prices, attributes
What Python stores:    Nothing (stateless)
What Python returns:   Scores, decisions, issues (no user PII)
What Gemini receives:  Text content, image content (per Google's data policy)
```

---

## 11. Monitoring & Observability

### 11.1 Health Checks

**Python service:**
```
GET /api/v1/health

Response:
{
  "status": "healthy",
  "gemini": "connected",        // Last successful Gemini call < 5 min ago
  "uptime_seconds": 3600,
  "requests_processed": 142,
  "avg_response_time_ms": 4200
}
```

**NestJS health check extension:**
```
GET /api/v1/health

Response includes:
{
  "ai_service": "healthy",     // Can reach Python service
  ...existing health checks...
}
```

### 11.2 Key Metrics to Track

| Metric | Where | Alert Threshold |
|--------|-------|----------------|
| AI service response time | Python | > 30 seconds average |
| AI service error rate | Python | > 10% of requests |
| Gemini API error rate | Python | > 5% of calls |
| Gemini API latency | Python | > 15 seconds average |
| Verification throughput | NestJS | < expected volume (service may be down) |
| Auto-approve rate | NestJS | < 50% or > 95% (model may be miscalibrated) |
| Manual review queue size | NestJS | > 100 pending (admin backlog growing) |

### 11.3 Logging

**Python service logs (structured JSON):**
```json
{
  "timestamp": "2026-02-12T10:30:00Z",
  "level": "INFO",
  "event": "verification_complete",
  "listing_id": "uuid",
  "decision": "APPROVE",
  "score": 0.92,
  "text_score": 0.95,
  "image_score": 0.89,
  "duration_ms": 4200,
  "gemini_calls": 2,
  "gemini_tokens_used": 1500
}
```

---

## 12. Deployment

### 12.1 Docker Setup

**Python service Dockerfile:**
```
Python 3.11-slim
  --> Install dependencies (requirements.txt)
  --> Copy app code
  --> Expose port 8000
  --> CMD: uvicorn app.main:app --host 0.0.0.0 --port 8000
```

**Docker Compose (development):**
```
services:
  backend:     NestJS on port 3001
  ai-service:  Python on port 8000
  postgres:    Port 5433
  redis:       Port 6379
```

### 12.2 Environment Variables

**Python AI Service (.env):**

| Variable | Description |
|----------|-------------|
| `GEMINI_API_KEY` | Google Gemini API key |
| `SERVICE_API_KEY` | Shared secret for NestJS authentication |
| `PORT` | Service port (default: 8000) |
| `LOG_LEVEL` | Logging level (default: INFO) |
| `GEMINI_TEXT_MODEL` | Model name (default: gemini-pro) |
| `GEMINI_VISION_MODEL` | Model name (default: gemini-pro-vision) |
| `GEMINI_RATE_LIMIT` | Requests per minute (default: 60) |
| `REQUEST_TIMEOUT` | Max seconds per request (default: 30) |

**NestJS Backend additions (.env):**

| Variable | Description |
|----------|-------------|
| `AI_SERVICE_URL` | URL of Python service (e.g. http://ai-service:8000) |
| `AI_SERVICE_API_KEY` | Shared secret (must match Python's SERVICE_API_KEY) |
| `AI_SERVICE_TIMEOUT_MS` | HTTP timeout (default: 30000) |
| `AI_SERVICE_ENABLED` | Feature flag to enable/disable AI (default: true) |

### 12.3 CI/CD

Follows the same pattern as existing backend/frontend:

```
Push to branch
  --> GitHub Actions
      --> Build Docker image
      --> Run tests
      --> Push to container registry
      --> Deploy to environment (dev/uat/prod)
```

The AI service gets its own GitHub Actions workflow in the `realfinder-ai` repo, using the same `Phaedra-Solutions/reuseable-templates` for Docker build/scan.

---

## 13. Cost Estimate

### Gemini API Pricing (as of 2026)

| Model | Input | Output |
|-------|-------|--------|
| Gemini Pro (text) | $0.50 / 1M tokens | $1.50 / 1M tokens |
| Gemini Pro Vision | $0.50 / 1M tokens | $1.50 / 1M tokens |

### Per-Verification Cost

| Trust Level | Gemini Calls | Estimated Tokens | Cost |
|-------------|-------------|-----------------|------|
| TL1 (Listing) | 2 (text + vision) | ~2,000 | ~$0.01-0.02 |
| TL2 (Broker) | 3 (OCR + validation + fraud) | ~3,000 | ~$0.02-0.05 |
| TL3 (Property) | 4+ (docs + consistency + media + conflicts) | ~5,000 | ~$0.05-0.10 |

### Monthly Cost at Scale

| Volume | TL1 Cost | TL2 Cost | TL3 Cost | Total |
|--------|----------|----------|----------|-------|
| 100 listings/mo | $1-2 | $2-5 | $5-10 | ~$10-15 |
| 1,000 listings/mo | $10-20 | $20-50 | $50-100 | ~$80-170 |
| 10,000 listings/mo | $100-200 | $200-500 | $500-1000 | ~$800-1700 |

Free tier (60 req/min, 1500/day) covers MVP volume at zero cost.

---

## 14. Implementation Roadmap

```
Phase 1: TL1 (Listing Verification)          ~37h / ~5 days
  - Python project setup + Gemini client
  - Text analysis + Image analysis
  - Combined endpoint + NestJS integration

Phase 2: TL2 (Broker Verification)           ~39h / ~5 days
  - Document OCR + License validation
  - Fraud detection
  - Combined endpoint + NestJS integration

Phase 3: TL3 (Property Verification)         ~48h / ~6 days
  - Ownership doc analysis + Metadata consistency
  - Media authenticity + Conflict detection
  - Combined endpoint + NestJS integration

Total: ~124h / ~16 working days
```

Each phase is independently deployable and valuable. TL1 alone solves the biggest pain point (manual listing review). TL2 and TL3 build on the same infrastructure.
