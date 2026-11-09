# Task 1: Web App QA & Debug Report

**Role:** Automation & QA Developer  
**Candidate:** Bapi Baidya  
**Target Application:** Conduit RealWorld ([demo.realworld.show](https://demo.realworld.show))  
**Backend REST API:** `https://api.realworld.show/api`  
**Date:** November 2026  

---

## 1. Executive Summary & Scope

An end-to-end quality assurance and diagnostic audit was executed across the **Conduit RealWorld** application (Angular 18 SPA + REST API). Testing systematically exercised core user journeys:
- User registration and duplicate conflict handling
- Session authentication, token issuance, and storage
- Content authoring (Article CRUD: Create, Read, Update, Delete)
- Favoriting, un-favoriting, and comment moderation
- Tag ingestion, boundary testing, and client/server validation
- Edge resilience and security posture

The audit surfaced critical functional bugs, input validation bypasses, and security smells detailed below.

---

## 2. Identified Issues (QA Bug Table)

| # | Title / Summary | Steps to Reproduce | Expected vs Actual | Severity | Suspected Cause |
|---|---|---|---|---|---|
| **1** | **Duplicate Account Registration Overwrites Identity & Issues New Session** | 1. Send `POST /api/users` with new user credentials (returns `201 Created` + Token A).<br>2. Send an identical `POST /api/users` with the exact same email and username.<br>3. Inspect response status and payload. | **Expected:** `422 Unprocessable Entity` ("email/username has already been taken").<br>**Actual:** Server returns `201 Created` with a newly minted JWT token, overwriting or duplicating user record without conflict rejection. | **Critical** | Missing unique constraint validation on email and username columns in backend registration controller and database schema. |
| **2** | **Whitespace-Only Article Title Accepted, Generating Empty Slug & Breaking Routing** | 1. Log in and navigate to `/editor`.<br>2. Fill Article Title with spaces (`"   "`).<br>3. Enter valid description and body; submit.<br>4. Observe response payload and browser URL. | **Expected:** Inline error "Title cannot be blank" or `422 Unprocessable Entity`.<br>**Actual:** Accepted with `201 Created` and `"slug": ""`. Client routes to `/article/` (colliding with base path; article cannot be viewed, updated, or deleted). | **High** | Validation checks truthiness without trimming (`if (title)`); slugify regex strips all whitespace to an empty string with no fallback ID. |
| **3** | **Production Build Exposes Global Debug Object Leaking Active JWT Token** | 1. Navigate to `https://demo.realworld.show` and log into account.<br>2. Open DevTools (`F12`) &rarr; Console.<br>3. Execute `window.__conduit_debug__.getToken()`. | **Expected:** Production builds must strip development hooks; `window.__conduit_debug__` should be `undefined`.<br>**Actual:** Plaintext JWT token and internal auth state methods are exposed globally on `window`. | **High** | Developer debug helper in main bundle was not gated behind `!environment.production` compiler flags. |
| **4** | **Absence of Essential HTTP Security Headers Across Edge Gateways** | 1. Send HEAD / GET request to web app or REST API.<br>2. Inspect response headers in DevTools Network tab or `curl -I`.<br>3. Check for CSP, HSTS, X-Frame-Options, X-Content-Type-Options. | **Expected:** Modern OWASP security headers enforced (`X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Strict-Transport-Security`).<br>**Actual:** All major security headers are completely missing from response headers. | **Medium** | Reverse proxy (Nginx/Cloudflare) or backend server lacks security middleware (e.g., Helmet.js). |
| **5** | **Tag Input Permits Unsanitized Raw HTML / Script Payloads** | 1. In `/editor`, enter `<script>alert(1)</script>` into "Enter tags" and press Enter.<br>2. Publish article.<br>3. Inspect JSON response payload and home feed tags. | **Expected:** Tag input sanitizes characters or enforces an alphanumeric whitelist.<br>**Actual:** Raw HTML string accepted, persisted in database, and returned verbatim in `tagList` array. | **Medium** | Tag handler only checks for non-empty string; lacks regex whitelist validation and HTML entity escaping. |

---

## 3. Root-Cause Analysis (RCA)
### Targeted Issue: Whitespace-Only Article Title Accepted, Generating Empty Slug & Breaking Routing (Issue #2)

1. **What is happening:**  
   When a user creates an article with a title composed solely of blank spaces (e.g., `"   "`), the backend accepts the payload with an HTTP `201 Created` response and commits the article with an empty string slug (`""`). Consequently, the frontend router routes the user to `https://demo.realworld.show/article/`, which collides with parent route paths or defaults to home, leaving the newly created article inaccessible, un-editable, and un-deletable (`PUT` and `DELETE /api/articles/` fail with 404/405 errors because REST calls hit the collection root instead of an individual entity).

2. **Why it occurs (Technical Root Cause):**  
   In the backend article service, title validation relies on loose existence checking (e.g. `if (!req.body.article.title)`) rather than sanitizing and asserting non-empty content after whitespace normalization (`if (!title || title.trim().length === 0)`). Following validation, the slug generator applies a standard regex replacement (`title.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '')`), which reduces a string of pure whitespace characters into an empty string `""`. Because neither the database schema nor the ORM model enforces a minimum slug length constraint, the record is inserted successfully with `slug = ""`.

3. **How to fix it (Remediation Plan):**  
   - **Client & Server Schema Validation:** Implement strict schema validation at the boundary using Zod/Joi: `title: z.string().trim().min(3, "Title must be at least 3 characters").max(120)`.
   - **Defensive Slug Generation:** Update the slugification utility to guarantee a non-empty fallback. If the regex output has length 0, fall back to an automated slug such as `untitled-${crypto.randomUUID().slice(0, 8)}`.
   - **Database Integrity Guard:** Add a database check constraint `CHECK (length(trim(slug)) > 0)` and a unique index on slug to guarantee database-level integrity against invalid slugs.

---

## 4. Summary & Verification

All issues were tested and verified against the live environment. The accompanying PDF report `Task1_QA_Report_Bapi_Baidya.pdf` contains the formatted submission document.

