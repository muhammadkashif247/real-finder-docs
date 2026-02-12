# TL3 -- Property / HouseBook Verification (AI Implementation Plan)

> **AI Provider**: Google Gemini (Gemini Pro + Gemini Pro Vision)
> **AI Service**: Separate Python FastAPI microservice (same service as TL1 and TL2)
> **Communication**: REST API (NestJS backend calls Python service via HTTP)
> **Backend Repo**: `realfinder-backend` (NestJS)
> **AI Repo**: `realfinder-ai` (Python / FastAPI)
> **Created**: 2026-02-12

---

## Resources Needed

| What | Details |
|------|---------|
| **Gemini API Key** | Same key as TL1/TL2 -- covers text and vision |
| **Python 3.11+** | Same AI service (new endpoints added) |
| **Additional packages** | None beyond what TL1/TL2 already install |
| **Cost** | ~$0.05-0.10 per property verification (multiple documents + cross-referencing uses more tokens) |

---

## What Is TL3?

TL3 verifies the property itself -- not just the listing or broker. It answers: **"Is this property legally and physically verified?"**

This is the highest trust level. It involves checking ownership documents (title deeds, registry extracts), matching property metadata between listings and the HouseBook, verifying media authenticity, and detecting conflicts like multiple ownership claims or duplicate listings for the same property. Admin always has final authority, but AI does the heavy lifting of reading documents, spotting inconsistencies, and flagging problems.

### Benefits

- **Users** get maximum trust visibility -- a TL3 badge means the property is fully verified
- **Platform** can offer premium/featured placement for TL3 properties
- **Disputes** are reduced because ownership is pre-verified before problems arise
- **Admins** spend less time on manual document review and conflict resolution
- **Legal risk** is lower because the platform can show due diligence in verification

---

## Scope

### AI team builds (Python service):
- Ownership document analysis endpoint (OCR + data extraction from title deeds)
- Property metadata consistency endpoint (listing vs HouseBook comparison)
- Media authenticity verification endpoint
- Conflict detection endpoint (multiple owners, duplicate listings)
- Combined property verification endpoint

### Backend team builds (NestJS):
- Call the Python service when HouseBook verification is triggered
- Save AI results to database (schema changes needed)
- Update verification status on Housebook / OwnershipClaim
- Admin endpoints to view flagged properties, conflicts, and override

### Out of scope for TL3:
- On-ground physical inspection (manual process)
- Third-party verification API integrations (future phase)
- Utility/tax record lookups (future phase)
- Video walkthrough verification (future phase)

---

## Features

### 1. Ownership Document Analysis

**What it is:**
AI reads uploaded ownership documents (title deeds, registry extracts, sale agreements) and extracts key data: owner name, property address, property type, registration number, dates.

**Why we need it:**
When a user submits an ownership claim, they upload scanned documents as proof. Today an admin reads each document, manually extracts the relevant info, and cross-checks it against the HouseBook data. This is slow and error-prone. AI reads the document in seconds, extracts structured data, and flags mismatches between the document and the claim.

**How it works:**
Document images are sent to Gemini Vision with a structured prompt. Gemini extracts key fields (owner name, address, registration number, dates). The Python service compares extracted data against the ownership claim details and flags any discrepancies -- e.g. the claim says "Villa 5, Palm Jumeirah" but the title deed shows "Villa 7, Palm Jumeirah".

**What the admin sees:**
A structured comparison: what the claimant entered vs. what AI extracted from the document. Mismatches highlighted in red. Admin reviews and decides.

---

### 2. Property Metadata Consistency

**What it is:**
AI compares a listing's data against the HouseBook record to check if they are consistent -- same address, same property type, matching area size, matching number of rooms.

**Why we need it:**
A listing might claim "5-bedroom villa, 5000 sqft" but the HouseBook for that property says "3-bedroom apartment, 1200 sqft". These inconsistencies could mean the listing is linked to the wrong property, or the broker is exaggerating. Catching these mismatches is essential for TL3 trust.

**How it works:**
The Python service receives both the listing data and the HouseBook data. It compares key fields: property type, area, bedrooms, bathrooms, address, coordinates (if within a reasonable radius). For text fields that might be described differently (e.g. "apt" vs "apartment"), Gemini Pro is used to assess semantic similarity. The service returns a consistency score and lists all mismatches found.

**What the admin sees:**
A side-by-side comparison table: Listing value vs. HouseBook value for each field. Mismatches are flagged. Admin can update either record or reject the verification.

---

### 3. Media Authenticity Verification

**What it is:**
AI checks if the property photos and videos submitted for TL3 verification are genuine -- not taken from another property, not digitally altered, and consistent with the property's location and type.

**Why we need it:**
For TL3, the platform is certifying the property itself. If someone submits photos of a luxury villa but the HouseBook is for a studio apartment, that's a problem. Similarly, if the same photos appear on a completely different property, it indicates fraud.

**How it works:**
Gemini Vision analyzes the images and checks: (a) Do the photos match the property type in the HouseBook? (b) Are there signs of digital manipulation? (c) Do the images show metadata consistency (similar lighting, angles suggesting same property)? The service also cross-references images with those already stored for other properties to detect reuse.

**What the admin sees:**
Per-image results showing: authentic/suspicious, reason for flag, comparison to HouseBook property type. For cross-matches: "These images also appear on Property X."

---

### 4. Conflict Detection

**What it is:**
AI detects conflicts across the platform: multiple ownership claims for the same property, duplicate listings for the same property by different brokers, mismatches between HouseBook records.

**Why we need it:**
Conflicts are the hardest problems for admins. Two people both claim to own the same villa. Three brokers list the same apartment. A HouseBook has contradictory data. Today admins discover these manually, often after a user complaint. AI proactively scans for conflicts and presents them to admins before they become problems.

**How it works:**
The Python service receives property data and checks against existing records: (a) Other ownership claims for the same HouseBook, (b) Listings with matching addresses / coordinates / images linked to different HouseBooks, (c) Contradictory data across related records. Gemini Pro is used to assess whether two text descriptions refer to the same property even if worded differently. The service returns a list of detected conflicts with severity scores.

**What the admin sees:**
A conflict report listing each detected issue: type (duplicate claim, duplicate listing, data mismatch), severity (high/medium/low), the records involved, and AI's recommendation. Admin resolves each conflict.

---

### 5. Combined Property Verification

**What it is:**
A single endpoint that runs all checks for a property and returns a comprehensive verification result.

**Why we need it:**
Same pattern as TL1 and TL2 -- the backend makes one call and gets a complete result. The orchestrator runs all relevant checks based on what data is available (not all checks apply to every property).

**How it works:**
The backend sends HouseBook data, listing data, ownership claim data, and document URLs. The service runs applicable checks in parallel, combines results with weighted scoring (documents: 35%, consistency: 25%, media: 15%, conflicts: 25%), and returns:
- Score >= 0.80 --> **APPROVE** -- property should get TL3 status
- Score 0.50-0.79 --> **FLAG** -- needs admin review
- Score < 0.50 --> **REJECT** -- significant issues found

---

## Architecture

```
Admin / System triggers property verification
  --> NestJS collects: HouseBook data, linked listings, ownership claims, documents
  --> NestJS calls Python: POST /api/v1/verify/property
      --> Python: Ownership document analysis (Gemini Vision OCR)
      --> Python: Metadata consistency check (listing vs HouseBook)
      --> Python: Media authenticity verification (Gemini Vision)
      --> Python: Conflict detection (cross-reference existing records)
      --> Python: Combined score returned
  <-- NestJS receives response
  --> NestJS saves AI results
  --> NestJS updates verification status:
        APPROVE --> Property gets TL3 badge
        FLAG    --> Queued for admin review
        REJECT  --> Verification denied, reasons shown
```

---

## What Already Exists (Backend)

| What | Where | Status |
|------|-------|--------|
| Housebook model | `prisma/schema.prisma` --> `Housebook` with address, coordinates, property type, status | Ready |
| OwnershipClaim model | `prisma/schema.prisma` --> `OwnershipClaim` with claimType, claimStatus | Ready |
| OwnershipClaimDocument model | `prisma/schema.prisma` --> stores uploaded document URLs | Ready |
| OwnershipConflict model | `prisma/schema.prisma` --> for tracking detected conflicts | Ready |
| MarketingAuthorization model | `prisma/schema.prisma` --> broker-property authorization | Ready |
| PropertyMedia model | Photos/videos linked to HouseBook or Listing | Ready |
| PropertyDescription model | Descriptions linked to HouseBook or Listing | Ready |
| HousebookAttribute model | Key-value attributes for the property | Ready |
| Listing model with verification fields | `verificationStatus`, `verifiedAt` from TL1 | Ready |
| AI audit tables | `AiEvent`, `AiDecision`, `AiReviewTask`, `AiFeedback` | Ready |
| Admin role + RBAC guards | `src/modules/rbac/` | Ready |
| Audit events | `src/modules/audit/` | Ready |

---

## What's Missing (Backend Schema Changes)

The `Housebook` and `OwnershipClaim` models currently have no AI verification fields. We need to add:

**On Housebook:**

| Field | Type | Purpose |
|-------|------|---------|
| `verificationLevel` | `Enum?` | Track which TL level the property has achieved (NONE, TL1, TL2, TL3) |
| `aiConfidenceScore` | `Decimal(3,2)?` | Overall AI confidence for the latest verification |
| `aiDetails` | `Json?` | Structured AI results |
| `aiEventId` | `Uuid?` | Link to `AiEvent` for audit trail |

**On OwnershipClaim:**

| Field | Type | Purpose |
|-------|------|---------|
| `aiConfidenceScore` | `Decimal(3,2)?` | AI confidence in this claim's legitimacy |
| `aiDetails` | `Json?` | Structured AI results (OCR output, consistency check) |
| `aiEventId` | `Uuid?` | Link to `AiEvent` for audit trail |

---

## Python Service Endpoints

### `POST /api/v1/verify/property`

Combined property verification -- runs all applicable checks.

**Request body:**
```json
{
  "housebook_id": "uuid",
  "housebook": {
    "address_normalized": "Villa 5, Palm Jumeirah, Dubai",
    "property_type": "VILLA",
    "latitude": 25.1124,
    "longitude": 55.1390,
    "attributes": {
      "bedrooms": "5",
      "bathrooms": "6",
      "area_sqft": "8000"
    }
  },
  "listings": [
    {
      "listing_id": "uuid",
      "title": "Luxury 5BR Villa on Palm Jumeirah",
      "description": "...",
      "price": 15000000,
      "currency": "AED",
      "image_urls": ["https://s3.../img1.jpg"]
    }
  ],
  "ownership_claims": [
    {
      "claim_id": "uuid",
      "claimant_name": "Mohammed Al Rashid",
      "claim_type": "OWNER",
      "document_urls": ["https://s3.../title-deed.jpg"]
    }
  ],
  "existing_claims_for_property": [],
  "existing_listings_nearby": []
}
```

**Response:**
```json
{
  "decision": "APPROVE",
  "combined_score": 0.85,
  "document_analysis": {
    "status": "PASS",
    "confidence_score": 0.88,
    "extracted_data": {
      "owner_name": "Mohammed Al Rashid",
      "property_address": "Villa 5, Palm Jumeirah, Dubai",
      "registration_number": "DLD-2024-98765",
      "registration_date": "2024-03-15"
    },
    "mismatches": [],
    "execution_time_ms": 3500
  },
  "metadata_consistency": {
    "status": "PASS",
    "confidence_score": 0.92,
    "comparisons": [
      {"field": "property_type", "housebook": "VILLA", "listing": "VILLA", "match": true},
      {"field": "bedrooms", "housebook": "5", "listing": "5", "match": true},
      {"field": "address", "housebook": "Villa 5, Palm Jumeirah", "listing": "Palm Jumeirah", "match": true}
    ],
    "mismatches": [],
    "execution_time_ms": 1200
  },
  "media_authenticity": {
    "status": "PASS",
    "confidence_score": 0.80,
    "per_image_results": [
      {
        "url": "https://s3.../img1.jpg",
        "is_authentic": true,
        "matches_property_type": true,
        "manipulation_detected": false
      }
    ],
    "cross_matches": [],
    "execution_time_ms": 3800
  },
  "conflict_detection": {
    "status": "PASS",
    "confidence_score": 0.95,
    "conflicts_found": [],
    "execution_time_ms": 800
  }
}
```

### `POST /api/v1/analyze/ownership-document`

Ownership document OCR only (can be called independently).

### `POST /api/v1/analyze/property-consistency`

Metadata consistency check only (can be called independently).

### `POST /api/v1/detect/conflicts`

Conflict detection only (can be called independently).

### `GET /api/v1/health`

Health check (shared with TL1/TL2).

---

## NestJS Integration Points

### 1. HouseBook verification trigger

When an admin initiates full property verification, or when an ownership claim is submitted, the backend collects all relevant data and calls the Python service.

### 2. Ownership claim flow

When a user submits an ownership claim with documents, the backend calls the document analysis endpoint to pre-screen the documents before admin review.

### 3. Admin endpoints (new or extended)

- **List flagged properties** -- query properties where AI flagged issues
- **View verification details** -- show document OCR, consistency results, conflicts
- **View conflicts** -- dedicated conflict resolution dashboard
- **Override decision** -- admin resolves conflicts, approves/rejects, logs to `AiFeedback`

---

## Tasks

| # | Task | Hours | Why | Status |
|---|------|-------|-----|--------|
| 1 | Backend schema changes -- add AI fields to Housebook + OwnershipClaim + migration | 4h | Need somewhere to store AI results for property-level checks | Not started |
| 2 | Ownership document analysis endpoint (Gemini Vision OCR, structured extraction) | 10h | Core feature -- reads title deeds and extracts ownership data automatically | Not started |
| 3 | Property metadata consistency endpoint (listing vs HouseBook comparison) | 8h | Catches mismatches between listings and property records before they confuse users | Not started |
| 4 | Media authenticity verification endpoint (Gemini Vision + cross-reference) | 6h | Ensures property photos are genuine and match the actual property type | Not started |
| 5 | Conflict detection endpoint (duplicate claims, duplicate listings, data mismatches) | 8h | Proactively finds conflicts before they become user complaints or legal issues | Not started |
| 6 | Combined property verification endpoint (orchestrator) | 4h | Single endpoint for backend -- runs all checks and returns one decision | Not started |
| 7 | NestJS integration (HTTP calls, save results, admin conflict dashboard) | 4h | Connects AI to the real property verification and conflict resolution workflow | Not started |
| 8 | Tests (unit + integration) | 4h | Ensures property verification works correctly before going live | Not started |
| | **Total** | **48h** | | |

---

## Flow

```
Admin initiates property verification / User submits ownership claim
  --> NestJS collects HouseBook data, linked listings, claims, documents
  --> NestJS calls Python: POST /api/v1/verify/property
      --> Python: Document OCR (extracts owner, address, registration from title deed)
      --> Python: Metadata consistency (compares listing data vs HouseBook data)
      --> Python: Media authenticity (checks property photos are genuine)
      --> Python: Conflict detection (scans for duplicate claims/listings)
      --> Python: Combined score returned
  --> NestJS saves AI results to Housebook / OwnershipClaim
  --> NestJS updates status:
        >= 0.80 --> Property gets TL3 badge, maximum trust
        0.50-0.79 --> Flagged for admin review
        < 0.50 --> Verification denied, issues shown to admin
```
