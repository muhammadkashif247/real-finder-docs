# RealFinder – Trust Level (TL1, TL2, TL3) & AI Verification Framework

## Table of Contents

1. [Purpose of This Document](#1-purpose-of-this-document)
2. [Trust Levels – High-Level Summary](#2-trust-levels--high-level-summary)
3. [TL1 – Listing Verification](#3-tl1--listing-verification)
4. [TL2 – Broker Verification](#4-tl2--broker-verification)
5. [TL3 – Property / HouseBook Verification](#5-tl3--property--housebook-verification)
6. [HouseBook & Listing Linking (AI Suggestions)](#6-housebook--listing-linking-ai-suggestions)
7. [Conflict Resolution (AI-First, Admin-Final)](#7-conflict-resolution-ai-first-admin-final)

---

## 1. Purpose of This Document

This document explains:

- How real-world property verification works today
- How we translate real-world verification into AI-driven workflows
- What TL1, TL2, and TL3 exactly mean in our system
- Which user actions trigger which Trust Level
- How AI assists verification, linking, and conflict resolution
- Platform rules around broker geography & listings

This is intended to align **Product**, **AI**, **Backend**, and **Admin** workflows.

---

## 2. Trust Levels – High-Level Summary

| Trust Level | What it Verifies              | Real-World Meaning                                    |
| ----------- | ----------------------------- | ----------------------------------------------------- |
| **TL1**     | Listing Verification          | "This listing looks real and non-fake"                |
| **TL2**     | Broker Verification           | "The person listing is a verified broker/agent"       |
| **TL3**     | Property / HouseBook Verification | "The property itself is fully verified"           |

### Important Rule

> A property **must have at least TL1** to be publicly listed.
> **No TL = No public listing.**

---

## 3. TL1 – Listing Verification

### 3.1 What TL1 Is

TL1 verifies **the listing itself**, not the broker and not full ownership.

It answers one simple question:

> "Is this listing real enough to be shown publicly?"

### 3.2 Real-World Process (How Humans Do It Today)

In the real world, a human reviewer usually:

1. Checks if the address exists
2. Looks at photos to see if they are realistic
3. Checks if the description is reasonable (not spam/fake)
4. Ensures the same property isn't already listed multiple times
5. Confirms minimum required fields are present

This is **basic sanity validation**, not legal verification.

### 3.3 AI Translation (How AI Will Do TL1)

AI performs automated listing sanity checks, such as:

| Check                  | Description                                              |
| ---------------------- | -------------------------------------------------------- |
| Address validation     | Map/location consistency                                 |
| Image analysis         | Duplicates, stock images, mismatches                     |
| Text analysis          | Spam patterns, fake pricing, missing data                |
| Duplicate detection    | Same address, same images, same metadata                 |
| Confidence scoring     | Low / medium / high risk                                 |

**Output:**
- **TL1 Approved** — listing goes public
- **Flagged for Admin review** — listing stays in review queue

### 3.4 When TL1 Is Triggered

TL1 runs when:

- A **new listing is created** (manual or CRM)
- A listing is **edited significantly**
- A listing is **imported from an external source** (CRM)

### 3.5 Who Is Affected

| Stakeholder    | Impact                                      |
| -------------- | ------------------------------------------- |
| Public users   | Can now see the listing                     |
| Admin          | Sees AI confidence + flags                  |
| Search engine  | Listing becomes indexable                   |
| Platform       | Listing becomes eligible for ranking        |

---

## 4. TL2 – Broker Verification

### 4.1 What TL2 Is

TL2 verifies **the person or entity** listing the property.

It answers:

> "Is this broker/agent a legitimate, verified professional?"

### 4.2 Real-World Process

In the real world:

1. Broker signs up
2. Provides license / registration
3. Documents are checked with local authority
4. Broker is approved to operate in a specific country/region

### 4.3 AI Translation

AI assists by:

| Check                        | Description                                     |
| ---------------------------- | ----------------------------------------------- |
| Document validation          | OCR + pattern checks on broker documents        |
| License format matching      | Match license formats to country rules           |
| Duplicate / fraud detection  | Check for duplicates and fraud patterns          |
| Inconsistency flagging       | Flag inconsistencies for Admin review            |

> **Final approval is still Admin-controlled in MVP.**

### 4.4 When TL2 Is Triggered

TL2 happens:

- At **broker signup**
- OR when **broker submits verification documents**

Once broker is approved:

> All listings created by that broker **inherit TL2 eligibility**.

### 4.5 Key Platform Rule (Important for AI Team)

**Broker verification is geography-based.**

Example:

> A broker verified for **Dubai (UAE)**
> - ✅ Can list properties in UAE
> - ❌ Cannot automatically list properties in other countries

**Cross-country listing support = future phase, not MVP.**

### 4.6 Who Is Affected

| Stakeholder    | Impact                                      |
| -------------- | ------------------------------------------- |
| Broker         | Gains trust badge + listing privileges      |
| Listings       | Become more trustworthy                     |
| Search ranking | TL2 listings rank higher                    |
| Admin          | Fewer fake actors on platform               |

---

## 5. TL3 – Property / HouseBook Verification

### 5.1 What TL3 Is

TL3 verifies **the property itself**, not just the listing or broker.

It answers:

> "Is this property legally and physically verified?"

This is the **highest trust level**.

### 5.2 Real-World Process

Typically involves:

1. Ownership documents (title deed, registry extract)
2. On-ground inspection
3. Video walkthroughs
4. Third-party verification
5. Utility/tax records

### 5.3 AI Translation

AI supports TL3 by:

| Check                          | Description                                        |
| ------------------------------ | -------------------------------------------------- |
| Ownership document analysis    | Analyze title deeds, registry extracts via OCR     |
| HouseBook metadata matching    | Match property metadata with HouseBook records     |
| Media authenticity verification| Verify photos/videos are genuine and consistent    |
| Cross-claim consistency        | Check consistency across all claims                |
| Conflict detection             | Detect multiple owners, multiple listings          |

> **Admin always has final authority.**

### 5.4 When TL3 Is Triggered

TL3 runs when:

- A **HouseBook is created or verified**
- **Ownership claims are submitted**
- **Admin initiates full verification**

### 5.5 Who Is Affected

| Stakeholder    | Impact                                      |
| -------------- | ------------------------------------------- |
| Public users   | Maximum trust visibility                    |
| Platform       | Premium/featured eligibility                |
| Disputes       | Reduced risk                                |
| AI systems     | Use TL3 as ground truth                     |

---

## 6. HouseBook & Listing Linking (AI Suggestions)

### 6.1 Real-World Problem

Admins manually identify:

- Which listings belong to which property
- Which records represent the same house/unit

### 6.2 AI Translation

AI automatically:

1. **Matches** address, geo-coordinates, layout, images
2. **Suggests** possible HouseBook ↔ Listing links
3. **Provides** confidence scores

> **Admin confirms the final link.**

### Linking Flow

```
Listing Created
    │
    ▼
AI analyzes address, geo, images, layout
    │
    ▼
AI searches existing HouseBooks for matches
    │
    ├── High confidence match found
    │       → Suggest link to Admin
    │
    ├── Multiple possible matches
    │       → Rank by confidence, present to Admin
    │
    └── No match found
            → Flag as new property candidate
```

---

## 7. Conflict Resolution (AI-First, Admin-Final)

### 7.1 Types of Conflicts

| Conflict Type                | Description                                          |
| ---------------------------- | ---------------------------------------------------- |
| Multiple ownership claims    | Two or more parties claim the same property          |
| Duplicate listings           | Same property listed multiple times                  |
| Incorrect broker listing     | Property listed by wrong / unauthorized broker       |
| HouseBook mismatches         | Data in HouseBook contradicts listing or documents   |

### 7.2 Real-World Process

Admin manually:

1. Reviews documents
2. Compares claims
3. Decides outcome

### 7.3 AI Translation

**AI:**

1. Analyzes all claims and documents
2. Detects contradictions
3. Ranks claims by credibility
4. Produces a recommendation report

**Admin:**

1. Reviews AI output
2. Makes final decision

### Conflict Resolution Flow

```
Conflict Detected (by AI or Admin)
    │
    ▼
AI gathers all related claims, documents, listings
    │
    ▼
AI analyzes & cross-references
    │
    ▼
AI produces recommendation report
    ├── Credibility ranking of each claim
    ├── Contradictions found
    └── Suggested resolution
    │
    ▼
Admin reviews AI report
    │
    ▼
Admin makes final decision
    │
    ├── Resolve in favor of claim X
    ├── Request more documents
    └── Escalate / flag for legal review
```

---

## Appendix: Trust Level Progression

```
No TL (Draft)
    │
    ▼  [Listing created / imported]
TL1 — Listing Verified
    │    "This listing looks real"
    │
    ▼  [Broker submits documents + Admin approves]
TL2 — Broker Verified
    │    "The broker is legitimate"
    │
    ▼  [HouseBook verified + ownership confirmed]
TL3 — Property Verified
         "The property is fully verified"
```

### Key Takeaways

- **TL1** is automated (AI-driven) with Admin override
- **TL2** is AI-assisted but **Admin-approved** in MVP
- **TL3** is the highest bar — AI supports, **Admin decides**
- Each trust level builds on the previous one
- AI provides confidence scores and recommendations at every level
- Admin always retains final authority for TL2 and TL3 decisions
