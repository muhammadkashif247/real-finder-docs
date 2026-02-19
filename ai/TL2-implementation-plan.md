# TL2 -- Broker Verification (AI Implementation Plan)

> **AI Provider**: Google Gemini (Gemini Pro + Gemini Pro Vision)
> **AI Service**: Separate Python FastAPI microservice (same service as TL1)
> **Communication**: REST API (NestJS backend calls Python service via HTTP)
> **Backend Repo**: `realfinder-backend` (NestJS)
> **AI Repo**: `realfinder-ai` (Python / FastAPI)
> **Created**: 2026-02-12

---

## Resources Needed

| What | Details |
|------|---------|
| **Gemini API Key** | Same key as TL1 -- covers text and vision |
| **Python 3.11+** | Same AI service as TL1 (new endpoints added) |
| **Additional packages** | None beyond what TL1 already installs |
| **Cost** | ~$0.02-0.05 per broker verification (document OCR uses more tokens than text analysis) |

---

## What Is TL2?

TL2 verifies the person or entity listing the property. It answers: **"Is this broker/agent a legitimate, verified professional?"**

Right now, when a broker submits verification documents (license, registration), an admin manually reviews them -- checking if the license number format is correct, if the document looks real, if the broker already exists under a different account. TL2 uses AI to automate the initial review so admins only handle edge cases.

### Benefits

- **Brokers** get verified faster -- AI pre-screens documents instead of waiting days for admin
- **Admins** skip obvious approvals and focus on suspicious submissions
- **Platform** blocks fake brokers earlier, before they post listings
- **Users** can trust that brokers with a TL2 badge are real professionals
- **Search ranking** -- TL2 listings rank higher, incentivizing brokers to verify

---

## Scope

### AI team builds (Python service):
- Document OCR + data extraction endpoint
- License format validation by country
- Fraud / duplicate pattern detection endpoint
- Combined broker verification endpoint

### Backend team builds (NestJS):
- Call the Python service when broker submits verification documents
- Save AI results to database (schema changes needed)
- Update `BrokerVerification.status` based on AI response
- Admin endpoints to view flagged broker verifications and override

### Out of scope for TL2:
- Real-time license authority API lookups (future phase)
- Cross-country verification (future phase)
- Automated re-verification on expiry (future phase)

---

## Features

### 1. Document OCR + Data Extraction

**What it is:**
AI reads the uploaded broker document image (license, registration certificate) and extracts key data: license number, broker name, issuing authority, country, expiry date.

**Why we need it:**
Brokers upload a photo or scan of their license. Today an admin has to read each document manually, find the license number, check the expiry, and compare it to what the broker entered. AI does this in seconds -- it reads the document, extracts the fields, and flags mismatches (e.g. broker entered license "12345" but the document shows "12346").

**How it works:**
The document image is sent to Gemini Vision with a structured prompt asking it to extract specific fields. Gemini returns the extracted data in a structured format. The Python service compares extracted data against what the broker submitted (license number, name) and flags any discrepancies.

**What the admin sees:**
Side-by-side comparison: what the broker entered vs. what AI extracted from the document. Mismatches are highlighted. Admin can confirm or correct.

---

### 2. License Format Validation

**What it is:**
AI validates that the license number follows the correct format for the broker's country. Different countries have different license formats (e.g. UAE RERA numbers follow a specific pattern, Saudi REDF licenses have a different format).

**Why we need it:**
A fake broker might enter a random string as their license number. Format validation catches obviously invalid license numbers without needing to call any external authority. It's a fast, free first check.

**How it works:**
The Python service maintains a set of regex patterns and validation rules per country. When a broker submits a license for a specific country, the service checks if the format matches. If the format is unknown for that country, it falls back to Gemini to assess whether the license number looks plausible given the document.

**What the admin sees:**
If the format doesn't match, the admin sees "License number format does not match expected pattern for [Country]" with the expected format shown.

---

### 3. Fraud / Duplicate Detection

**What it is:**
AI checks if this broker or document has been seen before under a different account, or if the document shows signs of tampering.

**Why we need it:**
A banned broker might re-register with a new email and re-upload the same license. Or someone might photoshop a license document. Detecting these patterns early prevents repeat offenders and forged documents from polluting the platform.

**How it works:**
The Python service checks for: (a) same license number already verified for a different broker, (b) Gemini Vision analysis of document authenticity -- looking for signs of digital editing, inconsistent fonts, blurred regions that suggest tampering. The service returns a fraud risk score.

**What the admin sees:**
If flagged, the admin sees the specific concern: "This license number is already verified for another broker" or "Document may have been digitally altered -- inconsistent font detected in license number region."

---

### 4. Combined Broker Verification

**What it is:**
A single endpoint that runs all checks (OCR, format validation, fraud detection) and returns a combined decision.

**Why we need it:**
Same as TL1 -- the backend needs one call to get a complete verification result. The orchestrator runs all checks, combines the scores, and returns APPROVE / FLAG / REJECT.

**How it works:**
The backend sends broker data + document URLs to the Python service. The service runs all checks, weights the results (OCR accuracy: 40%, format: 20%, fraud: 40%), and returns:
- Score >= 0.80 --> **APPROVE** -- broker should be verified
- Score 0.50-0.79 --> **FLAG** -- needs admin review
- Score < 0.50 --> **REJECT** -- document is likely fake or invalid

---

## Architecture

```
Broker submits verification documents
  --> NestJS backend saves documents to S3
  --> NestJS creates BrokerVerification record (status: PENDING)
  --> NestJS calls Python: POST /api/v1/verify/broker
      --> Python: Document OCR (Gemini Vision)
      --> Python: License format validation (rules per country)
      --> Python: Fraud/duplicate detection (Gemini Vision + DB check)
      --> Python: Combined score returned
  <-- NestJS receives response
  --> NestJS saves AI results to BrokerVerification
  --> NestJS updates BrokerVerification.status:
        APPROVE --> VERIFIED
        FLAG    --> stays PENDING for admin
        REJECT  --> REVOKED
```

---

## What Already Exists (Backend)

| What | Where | Status |
|------|-------|--------|
| Broker model | `prisma/schema.prisma` --> `Broker` with `status`, `countryId` | Ready |
| BrokerVerification model | `prisma/schema.prisma` --> `BrokerVerification` with `licenseNumber`, `status`, `countryId` | Ready |
| BrokerVerificationStatus enum | PENDING, VERIFIED, REVOKED, EXPIRED | Ready |
| Geography-based verification | `BrokerVerification` is unique per `[brokerId, countryId]` | Ready |
| AI audit tables | `AiEvent`, `AiDecision` | Ready (unused) |
| Admin role + RBAC guards | `src/modules/rbac/` | Ready |
| Audit events | `src/modules/audit/` | Ready |

---

## What's Missing (Backend Schema Changes)

The `BrokerVerification` model currently has no AI-related fields. We need to add:

| Field | Type | Purpose |
|-------|------|---------|
| `aiConfidenceScore` | `Decimal(3,2)?` | Overall AI confidence for this verification |
| `aiDetails` | `Json?` | Structured AI results (OCR output, format check, fraud check) |
| `aiEventId` | `Uuid?` | Link to `AiEvent` for audit trail |

This is a small migration -- 3 columns added to an existing table.

---

## Python Service Endpoints

### `POST /api/v1/verify/broker`

Combined broker verification -- runs OCR, format validation, and fraud detection.

**Request body:**
```json
{
  "broker_id": "uuid",
  "broker_name": "Ahmed Al Maktoum",
  "license_number": "BRN-12345",
  "country_code": "AE",
  "document_urls": [
    "https://s3.../license-front.jpg",
    "https://s3.../license-back.jpg"
  ],
  "existing_license_numbers": ["BRN-11111", "BRN-22222"]
}
```

**Response:**
```json
{
  "decision": "APPROVE",
  "combined_score": 0.88,
  "ocr_analysis": {
    "status": "PASS",
    "confidence_score": 0.90,
    "extracted_data": {
      "license_number": "BRN-12345",
      "broker_name": "Ahmed Al Maktoum",
      "issuing_authority": "RERA Dubai",
      "expiry_date": "2027-06-15",
      "country": "UAE"
    },
    "mismatches": [],
    "execution_time_ms": 2800
  },
  "format_validation": {
    "status": "PASS",
    "confidence_score": 0.95,
    "expected_format": "BRN-XXXXX",
    "issues": [],
    "execution_time_ms": 50
  },
  "fraud_detection": {
    "status": "PASS",
    "confidence_score": 0.82,
    "duplicate_found": false,
    "tampering_indicators": [],
    "execution_time_ms": 3200
  }
}
```

### `POST /api/v1/analyze/document`

Document OCR only (can be called independently).

### `GET /api/v1/health`

Health check (shared with TL1).

---

## NestJS Integration Points

### 1. Broker verification flow

When a broker submits documents, the backend calls the Python service. Based on the response, it updates `BrokerVerification.status` and saves AI details.

### 2. Admin endpoints (new or extended)

- **List flagged broker verifications** -- query where AI flagged
- **View verification details** -- show OCR results, format check, fraud flags
- **Override decision** -- admin approves/rejects, logs to `AiFeedback`

---

## Tasks

| # | Task | Hours | Why | Status |
|---|------|-------|-----|--------|
| 1 | Backend schema changes -- add AI fields to BrokerVerification + migration | 3h | Need somewhere to store AI results for broker checks | Not started |
| 2 | Document OCR endpoint (Gemini Vision, structured extraction, mismatch detection) | 10h | Core feature -- reads broker documents and extracts license data automatically | Not started |
| 3 | License format validation (country-specific regex rules + fallback to Gemini) | 8h | Catches obviously invalid license numbers without external API calls | Not started |
| 4 | Fraud / duplicate detection endpoint (Gemini Vision + duplicate check) | 6h | Prevents banned brokers from re-registering and catches forged documents | Not started |
| 5 | Combined broker verification endpoint (orchestrator) | 4h | Single endpoint for backend -- runs all checks and returns one decision | Not started |
| 6 | NestJS integration (HTTP call on document submit, save results, admin endpoints) | 4h | Connects AI to the real broker verification workflow | Not started |
| 7 | Tests (unit + integration) | 4h | Ensures broker verification works correctly before going live | Not started |
| | **Total** | **39h** | | |

---

## Flow

```
Broker signs up --> uploads license document
  --> NestJS saves document to S3
  --> NestJS creates BrokerVerification (status: PENDING)
  --> NestJS calls Python: POST /api/v1/verify/broker
      --> Python: OCR extracts license number, name, expiry from document
      --> Python: Format validation checks license number pattern for country
      --> Python: Fraud detection checks for duplicates + document tampering
      --> Python: Combined score returned
  --> NestJS saves AI results to BrokerVerification
  --> NestJS updates status:
        >= 0.80 --> status=VERIFIED, broker gains trust badge
        0.50-0.79 --> stays PENDING, admin reviews
        < 0.50 --> status=REVOKED, broker notified why
```
