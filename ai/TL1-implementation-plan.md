# TL1 -- Listing Verification (AI Implementation Plan)

> **AI Provider**: Google Gemini (Gemini Pro + Gemini Pro Vision)
> **AI Service**: Separate Python FastAPI microservice
> **Communication**: REST API (NestJS backend calls Python service via HTTP)
> **Backend Repo**: `realfinder-backend` (NestJS)
> **AI Repo**: `realfinder-ai` (Python / FastAPI)
> **Created**: 2026-02-12

---

## Resources Needed

| What | Details |
|------|---------|
| **Gemini API Key** | Get from https://aistudio.google.com/apikey -- one key covers both text and vision. Free tier = 60 req/min. Add as `GEMINI_API_KEY` in `.env` |
| **Python 3.11+** | Runtime for the AI service |
| **Key Python packages** | `fastapi`, `uvicorn`, `google-generativeai`, `httpx`, `pydantic`, `python-dotenv` |
| **Docker** | To containerize the Python service for deployment |
| **Cost** | ~$0.01-0.02 per listing verification (practically free at low volume with free tier) |

Everything else (PostgreSQL, Redis, S3, NestJS, Prisma) already exists in the backend.

---

## What Is TL1?

TL1 is the first trust level a listing can earn. It answers one question: **"Is this listing real enough to show publicly?"**

Right now when a broker submits a listing, it goes to PENDING and an admin has to manually review every single one. That doesn't scale. TL1 replaces that manual review with AI -- listings that look legitimate get auto-approved, suspicious ones get flagged for admin, and obvious fakes get rejected immediately.

### Benefits

- **Brokers** get their legitimate listings live faster (seconds instead of hours/days waiting for admin)
- **Admins** only review the edge cases instead of every listing -- saves massive time
- **Users** see fewer fake/spam listings on the platform
- **Platform** can scale listing volume without scaling admin headcount
- **Search/SEO** only indexes verified listings, improving quality

---

## Scope

### AI team builds (Python service):
- Text & Spam Analysis endpoint
- Image Analysis endpoint
- Combined verification endpoint (orchestrator)
- Gemini client wrapper

### Backend team builds (NestJS):
- Call the Python service from `submitForApproval()`
- Save verification results to `ListingVerification` table
- Update listing `verificationStatus` based on AI response
- Admin endpoints to view flagged listings, details, and override

### Out of scope for TL1:
- Address validation (future enhancement)
- Duplicate detection across listings (future enhancement)
- Completeness checks (backend team handles separately)

---

## Features

### 1. Text & Spam Analysis

**What it is:**
AI reads the listing title, description, price, and location and determines if the content looks like a real property listing or if it's spam, fake, or misleading.

**Why we need it:**
Fake listings are the #1 trust killer on property platforms. Common problems include: brokers putting phone numbers in descriptions to bypass the platform, spam listings with nonsense text, unrealistic pricing ($1 for a villa), copy-paste descriptions, and misleading claims like "guaranteed 10% ROI". Without this check, all of these go live and users lose trust.

**How it works:**
There are two layers. First, a fast rules layer catches obvious red flags instantly -- phone numbers in text, email addresses, $0 prices, excessive caps lock, very short descriptions. This costs nothing and runs in milliseconds. Second, the listing text gets sent to Gemini Pro which reads it like a human reviewer would -- checking if the description is coherent, if the price makes sense for the location, if there are any scam indicators. Gemini returns a legitimacy score. Both scores combine into a single confidence number.

**What the admin sees:**
If flagged, the admin sees exactly which rules triggered (e.g. "phone number detected in description") and what Gemini found (e.g. "price seems unrealistically low for Dubai Marina"). They can then approve or reject with one click.

---

### 2. Image Analysis

**What it is:**
AI looks at the listing photos and determines if they are real property photos or if they are fake, stolen from another listing, stock images, or watermarked from another platform.

**Why we need it:**
Images are the first thing a buyer/renter looks at. If a listing has stolen photos, stock images, or photos from a completely different property, the user gets deceived. Worse, brokers sometimes scrape photos from competitor listings and repost them. This erodes trust across the entire platform. Also, listings with zero images or broken image links shouldn't go live at all.

**How it works:**
First, basic validation runs for free -- checks if the listing has at least one image, checks if image URLs actually work (not broken links), and checks for duplicate images within the same listing. Then up to 5 images get sent to Gemini Vision which looks at each one and answers: Is this a real property photo? Is there a watermark from another platform (like Bayut, Property Finder, etc.)? What's the overall quality? All of this rolls up into a single image confidence score.

**What the admin sees:**
If flagged, the admin sees per-image results -- which images passed, which ones had watermarks, which ones looked like stock photos. They can look at the images themselves and decide.

---

### 3. Combined Verification (Orchestrator)

**What it is:**
A single endpoint that runs both checks (text + image) when called by the backend, combines the scores, and returns a final decision.

**Why we need it:**
The individual checks are useless on their own -- something needs to run them together, weigh the results, and make a final decision. The orchestrator is that glue. It means the backend only needs to make one HTTP call to get a complete verification result.

**How it works:**
The backend sends listing data (text, images, price, location) to the Python service's `/verify/listing` endpoint. The Python service runs both checks in parallel for speed, combines the scores with a 50/50 weight, and maps the combined score to a decision:
- Score >= 0.80 --> **APPROVE** -- listing should get VERIFIED status
- Score 0.50-0.79 --> **FLAG** -- listing should stay for admin review
- Score < 0.50 --> **REJECT** -- listing should be rejected

The response includes the decision, combined score, and detailed results from each check.

---

## Architecture

```
Broker submits listing
  --> NestJS backend sets status to PENDING
  --> NestJS calls Python AI service: POST /api/v1/verify/listing
      --> Python runs Text Analysis (rules + Gemini Pro)
      --> Python runs Image Analysis (validation + Gemini Vision)
      --> Python combines scores, returns decision
  <-- NestJS receives response
  --> NestJS saves results to ListingVerification table
  --> NestJS updates Listing.verificationStatus:
        APPROVE --> VERIFIED + ACTIVE + VISIBLE
        FLAG    --> FLAGGED + stays PENDING
        REJECT  --> REJECTED + stays PAUSED
```

---

## What Already Exists (Backend)

| What | Where | Status |
|------|-------|--------|
| Listing model with verification fields | `prisma/schema.prisma` --> `Listing.verificationStatus`, `verifiedAt`, `verificationConfidence` | Ready |
| ListingVerification table | `prisma/schema.prisma` --> `ListingVerification` model | Ready |
| Verification enums | `ListingVerificationStatus`, `VerificationCheckType`, `VerificationCheckStatus` | Ready |
| Listing text (title + description) | `PropertyDescription` model, linked to Listing | Ready |
| Listing images (URLs on S3) | `PropertyMedia` model, `fileUrl` field | Ready |
| Listing attributes (bedrooms, area, etc.) | `ListingAttribute` model | Ready |
| AI audit tables | `AiEvent`, `AiDecision`, `AiReviewTask`, `AiFeedback` | Ready (unused) |
| AI decision enum | `AiDecisionResult`: APPROVE / FLAG / REJECT | Ready |
| `submitForApproval()` method | `listing.service.ts` -- sets listing to PENDING | Ready (hook point) |
| Admin role + RBAC guards | `src/modules/rbac/` | Ready |
| Audit events | `src/modules/audit/` | Ready |
| Migration applied | `20260211100510_add_listing_verification` | Done |

---

## Python Service Endpoints

### `POST /api/v1/verify/listing`

Combined verification -- runs text + image analysis and returns a single decision.

**Request body:**
```json
{
  "listing_id": "uuid",
  "title": "3BR Apartment in Dubai Marina",
  "description": "Spacious apartment with sea view...",
  "price": 1500000.00,
  "currency": "AED",
  "listing_type": "SALE",
  "location": "Dubai Marina, Dubai, UAE",
  "latitude": 25.0800,
  "longitude": 55.1400,
  "image_urls": [
    "https://s3.../image1.jpg",
    "https://s3.../image2.jpg"
  ],
  "attributes": {
    "bedrooms": "3",
    "bathrooms": "2",
    "area_sqft": "1500"
  }
}
```

**Response:**
```json
{
  "decision": "APPROVE",
  "combined_score": 0.92,
  "text_analysis": {
    "status": "PASS",
    "confidence_score": 0.95,
    "rules_triggered": [],
    "gemini_analysis": {
      "legitimacy_score": 0.95,
      "issues": [],
      "summary": "Listing appears legitimate..."
    },
    "execution_time_ms": 1200
  },
  "image_analysis": {
    "status": "PASS",
    "confidence_score": 0.89,
    "images_checked": 2,
    "validation_issues": [],
    "per_image_results": [
      {
        "url": "https://s3.../image1.jpg",
        "is_real_property": true,
        "has_watermark": false,
        "quality": "good"
      }
    ],
    "execution_time_ms": 3400
  }
}
```

### `POST /api/v1/analyze/text`

Text-only analysis (can be called independently).

### `POST /api/v1/analyze/images`

Image-only analysis (can be called independently).

### `GET /api/v1/health`

Health check endpoint.

---

## NestJS Integration Points

### 1. `listing.service.ts` --> `submitForApproval()`

**Currently:** Sets listing to PENDING and stops.

**After:** Sets listing to PENDING, then calls the Python AI service asynchronously. Based on AI result, updates the listing verification status and saves check results to `ListingVerification` table.

### 2. Admin endpoints (new or extended)

- **List flagged listings** -- query listings where `verificationStatus = FLAGGED`
- **View verification details** -- read `ListingVerification` records for a listing
- **Override decision** -- admin approves/rejects, updates status, logs to `AiFeedback`

---

## Tasks

| # | Task | Hours | Why | Status |
|---|------|-------|-----|--------|
| 1 | Python project setup (FastAPI, structure, config, Docker, health endpoint) | 4h | Foundation for the entire AI service -- without this, nothing else can be built | Not started |
| 2 | Gemini client wrapper (text + vision methods, error handling, retry logic) | 3h | Shared component used by both text and image analysis -- avoids duplicate Gemini code | Not started |
| 3 | Text analysis endpoint (spam rules layer + Gemini Pro integration) | 8h | Catches fake/spam listings before they go public -- protects platform trust | Not started |
| 4 | Image analysis endpoint (validation + Gemini Vision integration) | 8h | Catches stolen/fake/watermarked images -- prevents deception of buyers/renters | Not started |
| 5 | Combined verification endpoint (orchestrator, runs both, combines scores) | 4h | Single endpoint for the backend to call -- simplifies integration | Not started |
| 6 | NestJS integration (HTTP call from submitForApproval, save results, admin endpoints) | 6h | Connects the AI service to the real workflow -- without this, AI results go nowhere | Not started |
| 7 | Tests (unit + integration for Python endpoints) | 4h | Ensures AI checks work correctly before going live | Not started |
| | **Total** | **37h** | | |

---

## Flow

```
Broker completes listing --> clicks Submit
  --> submitForApproval() sets PENDING
  --> NestJS calls Python: POST /api/v1/verify/listing
      --> Python: Text check (rules + Gemini Pro)
      --> Python: Image check (validation + Gemini Vision)
      --> Python: Combined score returned
  --> NestJS saves ListingVerification records
  --> NestJS updates Listing based on score:
        >= 0.80 --> verificationStatus=VERIFIED, marketStatus=ACTIVE, visibility=VISIBLE
        0.50-0.79 --> verificationStatus=FLAGGED, stays PENDING for admin
        < 0.50 --> verificationStatus=REJECTED, stays PAUSED, broker notified
```
