# DigiLocker Integration — Developer Guide

> **Who is this for?**  
> Anyone new to this project — interns, new team members, or developers from other teams who need to understand how this service works. No prior knowledge of DigiLocker is assumed.

---

## Table of Contents

1. [What Does This Service Do?](#1-what-does-this-service-do)
2. [The Big Picture — How It All Works](#2-the-big-picture--how-it-all-works)
3. [Project Structure](#3-project-structure)
4. [Key Concepts You Must Understand First](#4-key-concepts-you-must-understand-first)
5. [API Endpoints — What They Do](#5-api-endpoints--what-they-do)
6. [Walking Through Each Flow Step by Step](#6-walking-through-each-flow-step-by-step)
7. [The Database — Tables and What They Store](#7-the-database--tables-and-what-they-store)
8. [Configuration System — How Clients Customize Behavior](#8-configuration-system--how-clients-customize-behavior)
9. [Service Classes — What Each One Does](#9-service-classes--what-each-one-does)
10. [Error Codes — The Full List](#10-error-codes--the-full-list)
11. [Security — How Authentication Works](#11-security--how-authentication-works)
12. [Webhooks — Notifying Clients](#12-webhooks--notifying-clients)
13. [S3 File Storage — How Files Are Organized](#13-s3-file-storage--how-files-are-organized)
14. [How to Debug a Problem](#14-how-to-debug-a-problem)
15. [Common Mistakes New Developers Make](#15-common-mistakes-new-developers-make)
16. [Adding a New Feature — Step-by-Step Checklist](#16-adding-a-new-feature--step-by-step-checklist)
17. [Glossary](#17-glossary)

---

## 1. What Does This Service Do?

Imagine a company (let's call them "ABC Bank") wants to verify a customer's Aadhaar card digitally — without the customer having to upload any documents. This service makes that possible.

Here is the simple version:

```
ABC Bank  →  Our Service  →  DigiLocker (Government)  →  User's Documents
```

**Step by step in plain English:**

1. ABC Bank calls our API and says: "I want to verify user XYZ. Here's their mobile number."
2. We create a secure link and SMS it to the user.
3. The user opens the link, logs into DigiLocker (their government account), and gives permission to share their documents.
4. We fetch those documents (Aadhaar XML, PAN, Driving License, etc.) from DigiLocker.
5. We store them securely in S3 (cloud storage).
6. We notify ABC Bank via a webhook: "Done! Here's the data."
7. ABC Bank can also call our API later to get the documents or check status.

That's it. This whole journey is called a **transaction**.

---

## 2. The Big Picture — How It All Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           COMPLETE FLOW                                  │
└─────────────────────────────────────────────────────────────────────────┘

  Enterprise Client (ABC Bank)
         │
         │ POST /ent/v1/kyc/generate-url
         │ (Basic Auth)
         ▼
  ┌─────────────────┐
  │ Our Service     │ → Creates Transaction (status: GENERATED)
  │                 │ → Generates JWT token
  │                 │ → Sends SMS with link to user
  └────────┬────────┘
           │
           │ (User clicks SMS link)
           ▼
  ┌─────────────────┐
  │ Frontend (KYC   │ ← JWT token is embedded in the link URL
  │ Web App)        │
  └────────┬────────┘
           │
           │ POST /digilocker/v1/get-token  (JWT auth)
           ▼
  ┌─────────────────┐
  │ Our Service     │ → Validates token → Returns flow token + tracking ID
  └────────┬────────┘
           │
           │ POST /digilocker/v1/authorize  (JWT auth)
           ▼
  ┌─────────────────┐
  │ Our Service     │ → Builds DigiLocker authorization URL
  └────────┬────────┘
           │
           │ (Frontend redirects user to DigiLocker)
           ▼
  ┌─────────────────┐
  │ DigiLocker      │ ← User logs in and approves document sharing
  │ (Government)    │
  └────────┬────────┘
           │ (Redirects back with authorization code)
           ▼
  ┌─────────────────┐
  │ Frontend sends  │
  │ code to us      │ POST /digilocker/v1/process-request  (JWT auth)
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ Our Service     │ → Exchanges code for access token with DigiLocker
  │                 │ → Fetches document list from DigiLocker
  │                 │ → Downloads Aadhaar XML (and other docs)
  │                 │ → Parses XML, extracts name/DOB/address/photo
  │                 │ → Uploads everything to S3
  │                 │ → Updates transaction status → SUCCESS
  │                 │ → Sends webhook to ABC Bank
  └────────┬────────┘
           │
           │ Webhook POST → ABC Bank's server
           │ (payload: {status: "Success", name, dob, aadhaar...})
           ▼
  ┌─────────────────┐
  │ Enterprise      │
  │ Client gets     │ → Can also call /ent/v1/kyc/get-digilocker-details
  │ notified        │    to fetch full data later
  └─────────────────┘
```

---

## 3. Project Structure

```
src/main/java/com/digitap/digilocker/
│
├── auth/                          # Everything related to WHO can call our APIs
│   ├── configuration/             # Spring Security setup, JWT config, DB connections
│   ├── filters/                   # Intercepts incoming requests for auth checks
│   ├── model/                     # Database entities: Ent (Enterprise), EntClient
│   └── repository/                # DB queries for auth tables
│
├── transactions/                  # The main business logic lives here
│   ├── controller/                # REST API endpoints (the "door" to our service)
│   │   ├── DigilockerController.java          # Main KYC flow endpoints
│   │   ├── DigilockerConfigController.java    # Config management endpoints
│   │   ├── DashboardController.java           # Activity logs, doc download
│   │   └── DigilockerMetaController.java      # Account verification
│   │
│   ├── service/                   # All the business logic
│   │   ├── DigilockerGenerateURLService.java  # Creates transactions and URLs
│   │   ├── DigilockerSourceService.java       # Talks to DigiLocker APIs
│   │   ├── DigilockerInfoService.java         # Retrieves stored KYC data
│   │   ├── DigilockerAadhaarService.java      # Downloads and parses Aadhaar XML
│   │   ├── DigilockerAdditionalFilesService   # Downloads PAN, DL, etc.
│   │   ├── JwtService.java                    # Creates and validates JWT tokens
│   │   ├── S3Service.java                     # Uploads/downloads from S3
│   │   ├── WebhookService.java                # Sends webhooks to clients
│   │   └── WebhookUtil.java                   # Prepares webhook payloads
│   │
│   ├── model/                     # Database entities (Java representation of DB tables)
│   │   ├── DigilockerTransaction.java
│   │   ├── DigilockerConfig.java
│   │   ├── DigilockerErrorCodes.java
│   │   ├── DigilockerEaadhaarXmlDetails.java
│   │   ├── DigilockerDocs.java
│   │   └── DigilockerActivityLog.java
│   │
│   ├── repository/                # Database queries (extends JpaRepository)
│   ├── dto/                       # Request/Response objects (data containers)
│   │   ├── request/               # What clients send TO us
│   │   └── response/              # What we send BACK to clients
│   │
│   └── utils/                     # Helper classes
│       ├── DigilockerConfigUtility.java  # Config reading helpers
│       ├── WebhookUtil.java              # Webhook payload builders
│       ├── PdfGenerationUtil.java        # PDF creation helpers
│       └── FilesDownloader.java          # HTTP file download helpers
│
├── exception/                     # Custom exception classes
│   ├── SingleErrorException.java  # Our main exception type
│   └── ExceptionUtils.java        # Helper for RestClient errors
│
├── log/                           # Logging helpers
│   └── MaskingUtils.java          # Masks Aadhaar numbers, names in logs
│
└── GlobalExceptionHandler.java    # Catches ALL exceptions and formats error responses
```

---

## 4. Key Concepts You Must Understand First

### 4.1 Transaction

A **transaction** represents one KYC verification attempt for one user. It is created when the enterprise calls `generate-url` and dies (marked INACTIVE) when the user completes or the URL expires.

**Transaction lifecycle:**
```
GENERATED → INITIATED → SUCCESS
                      ↘ FAILED
         → EXPIRED (via cron)
```

Every action in the system is tied to a transaction. If you're debugging, always start by finding the transaction record.

### 4.2 Enterprise (Ent) and Client (EntClient)

- **Enterprise (Ent)**: A company using our service. Example: ABC Bank. Has an `eid` (Enterprise ID).
- **EntClient**: A specific application within that enterprise. Example: ABC Bank's mobile app vs web app. Has a `clientId` and `clientSecret`.

When ABC Bank calls our API, they authenticate with `clientId:clientSecret` as Basic Auth. We look up the `EntClient` record to know which enterprise they belong to.

### 4.3 Config System

Enterprises can configure how DigiLocker behaves for them — without us changing any code. Examples:
- How long the URL stays valid
- What documents are required
- What colors the UI should show
- What webhook URL to call

These configs live in the `digilocker_config` table. Each config has a `configName` (what it controls) and `configValue` (the actual setting). A config can also be specific to a `clientId` — if set, it overrides the enterprise-level config.

### 4.4 JWT Tokens

We use JWT tokens for the KYC flow (not for enterprise API calls — those use Basic Auth). There are two types:

- **URL Token** — Embedded in the SMS link. Contains transaction details, theme colors, branding config. Valid for the transaction's expiry duration.
- **Flow Token** — Generated at `get-token`. Valid for 1 hour. Used for `authorize` and `process-request`.

### 4.5 The DigiLocker OAuth Flow (PKCE)

DigiLocker uses standard OAuth2 with PKCE (Proof Key for Code Exchange). This is a security mechanism. Here's what happens:

1. We generate a random `codeVerifier` string
2. We hash it → `codeChallenge`
3. We send `codeChallenge` to DigiLocker in the authorization URL
4. DigiLocker returns an authorization `code` after user login
5. We send BOTH the `code` AND the original `codeVerifier` to get the access token
6. DigiLocker verifies the verifier matches the challenge from step 3 — confirms it's us

This prevents anyone who intercepts the authorization code from using it.

---

## 5. API Endpoints — What They Do

### Enterprise APIs (Basic Auth with clientId:clientSecret)

| Endpoint | What it does |
|---|---|
| `POST /ent/v1/kyc/generate-url` | **Start a new KYC transaction.** Enterprise calls this to get a link to send to the user. |
| `POST /ent/v1/kyc/get-digilocker-details` | **Get full KYC data** for a completed transaction. Returns Aadhaar XML, photo, address, etc. |
| `POST /ent/v1/digilocker/list-docs` | **Get a list of all documents** fetched for a transaction with their download URLs. |
| `POST /ent/v1/kyc/get-digilocker-status` | **Check if a transaction is complete.** Returns INITIATED / SUCCESS / FAILED. |
| `GET /digilocker/v1/config` | **View all configs** for the enterprise. |
| `POST /digilocker/v1/config` | **Create or update a config.** |
| `POST /digilocker/v1/activity-log` | **Get transaction history** with pagination. |
| `POST /digilocker/v1/download-docs` | **Get a presigned URL** to download a specific document from S3. |

### User Flow APIs (JWT Bearer Token)

| Endpoint | Who calls it | What it does |
|---|---|---|
| `POST /digilocker/v1/get-token` | Frontend | First step after user opens link. Validates the URL token, returns a flow token. |
| `POST /digilocker/v1/authorize` | Frontend | Generates the DigiLocker login URL to redirect the user to. |
| `POST /digilocker/v1/process-request` | Frontend | Processes the authorization code after user approves. Fetches all documents. |
| `POST /digilocker/v1/get-kyc-details` | Frontend | Gets cached KYC details (used if user needs to see their data again). |

---

## 6. Walking Through Each Flow Step by Step

### Flow 1: Generate URL

**Who calls it:** Enterprise (ABC Bank)  
**Endpoint:** `POST /ent/v1/kyc/generate-url`

```
Request:
{
  "uid": "user_abc_123",          // Your unique ID for this user
  "firstName": "John",
  "lastName": "Doe",
  "mobile": "9876543210",
  "emailId": "john@example.com",
  "redirectionUrl": "https://yourapp.com/kyc-done",
  "webhookUrl": "https://yourapp.com/webhook",
  "requiredDocs": "ADHAR,PANCR",   // Which documents you need
  "expiry": 259200000              // 3 days in milliseconds
}
```

**What happens inside:**
1. `DigilockerConfigController` → `DigilockerGenerateURLService`
2. Checks if a transaction with the same `uid` already exists (prevents duplicates)
3. Reads enterprise configs (expiry limits, allowed docs, theme, etc.)
4. Creates a `DigilockerTransaction` row in DB (status=GENERATED, url_status=ACTIVE)
5. Generates a JWT token with transaction data + theme config embedded
6. Builds the KYC URL: `{DIGILOCKER_BASE_URL}?token={jwtToken}`
7. Creates a short URL via Kaleyra and sends SMS to user
8. Logs `URL_GENERATED` event in `digilocker_activity_log`

```
Response:
{
  "url": "https://short.url/abc",       // SMS-friendly short link
  "transactionId": "123456789012345678",
  "kycUrl": "https://kyc.digitap.ai?token=eyJ..."  // Full URL
}
```

---

### Flow 2: User Opens Link → get-token

**Who calls it:** KYC Frontend (automatically when user opens link)  
**Endpoint:** `POST /digilocker/v1/get-token`  
**Auth:** JWT from the link URL

**What happens inside:**
1. Validates the URL JWT token (checks expiry, transaction status)
2. Checks transaction is still ACTIVE and not already completed
3. Creates a `trackingId` (UUID) for this user session
4. Generates a new flow token (1-hour validity)
5. Logs `KYC_INITIATED` event
6. Returns `hideExplanationScreen` flag (based on transaction config)

```
Response:
{
  "flowToken": "eyJ...",
  "transactionId": "123456789012345678",
  "trackingId": "uuid-here",
  "redirectionUrl": "https://yourapp.com/kyc-done"
}
```

> **Note:** After this, the frontend switches from the URL token to the flow token for subsequent calls.

---

### Flow 3: Generate DigiLocker Authorization URL → authorize

**Who calls it:** KYC Frontend  
**Endpoint:** `POST /digilocker/v1/authorize`  
**Auth:** Flow JWT

**What happens inside:**
1. Validates flow token + transaction ID match
2. Generates PKCE `codeVerifier` (random 43-char string)
3. Generates `codeChallenge` = SHA256(codeVerifier) encoded as base64url
4. Builds DigiLocker authorization URL:
   ```
   https://digilocker.meripehchaan.gov.in/...?
     client_id=...&
     redirect_uri=...&
     scope=openid+ADHAR+PANCR&
     code_challenge=...&
     code_challenge_method=S256
   ```
5. Logs `REDIRECTED_TO_SOURCE` event

```
Response:
{
  "url": "https://digilocker.meripehchaan.gov.in/...",
  "codeVerifier": "abc123...",    // Frontend must save this for next step
  "isMeriPehchaanFlow": true
}
```

The frontend redirects the user to this URL. The user logs into DigiLocker and approves document sharing.

---

### Flow 4: Process Documents → process-request

This is the **most important and complex** endpoint.

**Who calls it:** KYC Frontend (after DigiLocker redirects back with a code)  
**Endpoint:** `POST /digilocker/v1/process-request`  
**Auth:** Flow JWT

```
Request:
{
  "code": "auth-code-from-digilocker",
  "transactionId": "123456789012345678",
  "codeVerifier": "abc123...",    // The one from authorize step
  "grant_type": "authorization_code"
}
```

**What happens inside (in order):**

```
Step 1: Check if user denied access
   └── If errorCode = "access_denied" → throw DG1001, send error webhook

Step 2: Exchange auth code for access token
   └── POST to DigiLocker token endpoint
   └── Get access_token + refresh_token

Step 3: Fetch document list from DigiLocker
   └── GET /files/issued with Bearer token
   └── Check if e-Aadhaar (ADHAR) is available → else throw DG1002
   └── Check if Aadhaar scope was consented → else throw DG1003
   └── Log consent scopes

Step 4: Download Aadhaar XML (DigilockerAadhaarService)
   └── POST to DigiLocker e-Aadhaar XML endpoint
   └── Validate HMAC signature (if provided)
   └── Parse XML to extract:
       - Name, DOB, Gender
       - Masked Aadhaar number
       - Full address (house, street, city, state, pincode...)
       - Photo (base64)
   └── Upload photo to S3 → save as PNG
   └── Upload full XML to S3
   └── Save DigilockerEaadhaarXmlDetails in DB
   └── Save DigilockerDocs entry for XML

Step 5: Download additional documents in parallel (virtual threads)
   └── For each required document (PANCR, DRVLC, etc.):
       ├── Download PDF from DigiLocker
       ├── Download XML from DigiLocker
       ├── Upload both to S3
       └── Save DigilockerDocs entries

Step 6: Generate PDF summary
   └── Uses typst binary to generate a formatted PDF
   └── Uploads to S3

Step 7: Update transaction
   └── status = SUCCESS
   └── url_status = INACTIVE (link can no longer be used)
   └── completionDate = today

Step 8: Send webhook to enterprise
   └── Prepare payload with KYC data
   └── POST to enterprise's webhook URL (async, retries up to 2x)

Step 9: Log FLOW_COMPLETED event
```

```
Response:
{
  "status": "s",          // "s" = success, "f" = failure
  "uniqueId": "user_abc_123",
  "maskedAdharNumber": "XXXX-XXXX-1234",
  "name": "John Doe",
  "gender": "M",
  "dob": "01-01-1990",
  "careOf": "S/O Father Name",
  "address": {
    "house": "123",
    "street": "Main Street",
    "landmark": "Near Park",
    "loc": "Locality",
    "po": "Post Office",
    "dist": "Delhi",
    "subdist": "Sub District",
    "vtc": "Village",
    "pc": "110001",
    "state": "Delhi",
    "country": "India"
  },
  "image": "base64encodedphoto...",
  "pdfLink": "https://s3.presigned.url/..."
}
```

---

## 7. The Database — Tables and What They Store

### `digilocker_transaction` — The Master Table

Every KYC attempt is one row here. This is the first place to look when debugging.

| Column | Type | What it stores |
|---|---|---|
| `id` | Long | Auto-incremented primary key |
| `eid` | Integer | Which enterprise this belongs to |
| `client_id` | String | Which client application |
| `unique_id` | String | Enterprise's ID for the user |
| `transaction_id` | String | Our 18-digit unique transaction ID |
| `access_token` | String | Internal random token (part of JWT claims) |
| `short_url` | String | Short SMS link |
| `user_info` | JSON | `{firstName, lastName, mobile, email}` |
| `transaction_configs` | JSON | `{expiry, retryCount, requiredDocs, failOnDocNotFound...}` |
| `url_status` | Enum | `ACTIVE` (link works) or `INACTIVE` (link disabled) |
| `status` | Enum | `GENERATED → INITIATED → SUCCESS / FAILED` |
| `error_code` | Integer | FK to `digilocker_error_codes.id` (null if success) |
| `webhook_url` | String | Transaction-specific webhook URL |
| `redirection_url` | String | Where user goes after completion |
| `purge_status` | Enum | GDPR compliance (`NOT_PURGED` or `PURGE_COMPLETED`) |

**Status explanation:**
- `GENERATED`: URL created, user hasn't opened it yet
- `INITIATED`: User opened the link and started the flow
- `SUCCESS`: Documents fetched successfully
- `FAILED`: Something went wrong (see error_code for why)

---

### `digilocker_config` — Enterprise Settings

| Column | What it stores |
|---|---|
| `eid` | Enterprise ID |
| `client_id` | NULL = applies to all clients, set = only for that client |
| `config_name` | Which setting (see config table below) |
| `config_value` | The actual setting value (plain string or JSON) |
| `status` | `ACTIVE` or `INACTIVE` |

> **Sub-client override:** If both a `clientId=NULL` and a `clientId="mobile-app"` config exist for the same config name, the `clientId="mobile-app"` one wins for the mobile app, and the NULL one applies to everything else.

---

### `digilocker_eaadhaar_xml_details` — Parsed Aadhaar Data

This table stores all the information extracted from the Aadhaar XML. If a user's transaction is SUCCESS, there will be a row here.

| Column | What it stores |
|---|---|
| `transaction_id` | Links to the transaction |
| `name` | User's full name |
| `gender` | M / F / T |
| `date_of_birth` | DOB string |
| `masked_adhar_number` | "XXXX-XXXX-1234" |
| `care_of` | "S/O Father Name" or "D/O Father Name" |
| `house_no`, `street`, `landmark` ... | Address fields |
| `image_url` | S3 key for the photo |
| `pdf_link` | S3 key for the Aadhaar PDF |

---

### `digilocker_docs` — All Fetched Documents

One row per document per transaction. Documents can be XML or PDF.

| Column | What it stores |
|---|---|
| `transaction_id` | Links to the transaction |
| `doc_id` | UUID for this specific document |
| `url` | S3 key (path in the bucket) |
| `filename` | Document type code: `ADHAR`, `PANCR`, `DRVLC`, etc. |
| `doc_extension` | `pdf` or `xml` |

---

### `digilocker_activity_log` — Audit Trail

Every important step is logged here. Very useful for debugging.

| Event | When it's logged |
|---|---|
| `URL_GENERATED` | When generate-url is called successfully |
| `KYC_INITIATED` | When user opens the link (get-token) |
| `REDIRECTED_TO_SOURCE` | When user is sent to DigiLocker (authorize) |
| `ACCESS_TOKEN_GENERATED_FROM_SOURCE` | DigiLocker token received |
| `DOC_LIST_FETCHED_FROM_SOURCE` | Document list retrieved from DigiLocker |
| `AADHAAR_XML_FETCHED_FROM_SOURCE` | Aadhaar XML downloaded and parsed |
| `ADDITIONAL_DOCS_FETCHED_FROM_SOURCE` | Other documents fetched |
| `FLOW_COMPLETED` | Transaction marked SUCCESS |

---

### `digilocker_apis_log` — External API Calls

Every call made TO DigiLocker is logged here with the response. Useful when DigiLocker APIs fail.

---

### `digilocker_webhook_log` — Webhook Delivery Tracking

Every webhook attempt is logged. If a webhook fails, you can see the response code and retry attempts here.

---

### `digilocker_error_codes` — Error Catalog

| Error Code | What it means |
|---|---|
| DG1001 | User denied access to DigiLocker |
| DG1002 | E-Aadhaar not available in user's DigiLocker account |
| DG1003 | User did not consent to share Aadhaar |
| DG1006 | Document fetch timed out |
| DG1007 | Transaction not in success state (for detail fetch) |
| DG1009 | URL is inactive (already used or expired) |
| DG1011 | Technical error (additional file fetch failed) |
| DG1013 | Transaction already completed (URL reuse attempt) |
| DG1014 | Transaction failed (URL reuse attempt) |

---

## 8. Configuration System — How Clients Customize Behavior

Clients (enterprises) can change behavior by setting configs in `digilocker_config`. Here's what each config does:

| Config Name | Value Type | What it controls |
|---|---|---|
| `THEME_CONFIG` | JSON | UI colors, font, success/redirection messages |
| `HIDE_POWERED_BY_DIGITAP` | (presence) | Remove "Powered by Digitap" branding |
| `URL_EXPIRY_TIME` | Number (ms) | How long the generated link stays valid (default 3 days, max 7 days) |
| `PULL_ALL_DOCS` | (presence) | Allow client to request specific document types |
| `FAIL_ON_FETCHING_FILE` | (presence) | If enabled, allows `failOnDocNotFound` flag in generate-url |
| `ENTERPRISE_WEBHOOK_URL` | JSON | Global webhook config: `{"webhookUrl":"...","authKey":"Authorization","authValue":"Bearer xyz"}` |
| `DETAILED_WEBHOOK` | (presence) | Include full Aadhaar data in webhook payload |
| `ENTERPRISE_DIGILOCKER_CREDENTIALS` | JSON | Client's own DigiLocker API credentials |
| `CLOUD_STORAGE_CONFIG` | JSON | Client's own S3/GCP bucket credentials |
| `DIGILOCKER_OLD_LOGIN_FLOW` | (presence) | Use old DigiLocker login instead of MeriPehchaan |
| `DOMAIN_WHITELABELING` | String | Custom domain for the KYC web app |
| `DIGILOCKER_ID_IN_RESPONSE` | (presence) | Include the user's DigiLockerId in API responses |
| `PINLESS_FLOW` | (presence) | Skip PIN/OTP step (if DigiLocker supports it) |

### Theme Config Example
```json
{
  "themeColor": "hsl(216, 89%, 49%)",
  "logoAlignment": "center",
  "fontFamily": "Poppins",
  "successMessage": "Verification complete!",
  "redirectionMessage": "Please wait, redirecting..."
}
```

### Enterprise Webhook Config Example
```json
{
  "webhookUrl": "https://client.com/digilocker-webhook",
  "authKey": "Authorization",
  "authValue": "Bearer secret-token-here"
}
```

---

## 9. Service Classes — What Each One Does

Think of services like departments in a company:

| Service | Responsibility | Analogy |
|---|---|---|
| `DigilockerGenerateURLService` | Creates transactions and URL tokens | **Receptionist** — creates the appointment |
| `DigilockerSourceService` | Talks to DigiLocker's APIs | **Liaison officer** — communicates with the government |
| `DigilockerAadhaarService` | Downloads and parses Aadhaar XML | **Document specialist** — reads and extracts info from Aadhaar |
| `DigilockerAdditionalFilesService` | Downloads PAN, DL, etc. in parallel | **Field agent** — fetches multiple documents simultaneously |
| `DigilockerInfoService` | Retrieves stored data for client queries | **Records officer** — provides information from storage |
| `JwtService` | Creates and validates JWT tokens | **Security guard** — issues and checks passes |
| `S3Service` | Uploads/downloads files from S3 | **Warehouse manager** — stores and retrieves files |
| `WebhookService` | Sends HTTP notifications to clients | **Messenger** — delivers results to clients |
| `WebhookUtil` | Prepares the webhook payload | **Report writer** — formats data for delivery |

---

## 10. Error Codes — The Full List

When something goes wrong, we throw a `SingleErrorException` with one of these codes:

| Code | Scenario | What the client should do |
|---|---|---|
| `DG1001` | User clicked "Deny" in DigiLocker | Show user a message explaining they need to approve |
| `DG1002` | User doesn't have e-Aadhaar in their DigiLocker account | Ask user to link Aadhaar to DigiLocker first |
| `DG1003` | User unchecked Aadhaar consent in DigiLocker | Ask user to retry and approve all required documents |
| `DG1006` | DigiLocker API timed out fetching a document | Retry the request |
| `DG1007` | Enterprise called get-details but transaction isn't SUCCESS yet | Check status first with get-status endpoint |
| `DG1009` | URL already used or expired | Generate a new URL |
| `DG1011` | Technical error fetching additional documents | Check if docs are optional; may need to retry |
| `DG1013` | User tried to use a link for an already-completed transaction | Inform user their verification is already done |
| `DG1014` | User tried to use a link for a failed transaction | Generate a new URL |

---

## 11. Security — How Authentication Works

### Enterprise APIs (Basic Auth)

When ABC Bank calls `/ent/v1/kyc/generate-url`:

```
Request Header:
Authorization: Basic base64(clientId:clientSecret)
```

Spring Security checks:
1. Decodes the base64 header
2. Looks up `EntClient` by `clientId`
3. Compares the password (clientSecret)
4. If valid, the `EntClient` object is available in the controller

### User Flow APIs (JWT Bearer Token)

When the KYC frontend calls `/digilocker/v1/get-token`:

```
Request Header:
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
```

Spring Security checks:
1. Validates the JWT signature using RSA public key (`app.public.pem`)
2. Checks token expiry
3. The JWT claims are available as a `Jwt` object in the controller

The JWT is signed with our private key (`app.private.pem`). Only we can create valid tokens, but anyone with the public key can verify them.

### Two Security Chains

Spring Security has two separate chains configured:

```
Chain 1: /ent/v1/** and dashboard paths → Basic Auth
Chain 2: /digilocker/v1/** → JWT Bearer Token
```

Each chain is completely independent. A JWT token won't work on Basic Auth endpoints and vice versa.

---

## 12. Webhooks — Notifying Clients

After a transaction completes (success or failure), we notify the enterprise via a webhook.

### Webhook Types

**1. Transaction-specific webhook** — Set in `generate-url` request as `webhookUrl`:
```json
{
  "transactionId": "123456789012345678",
  "status": "Success",
  "errorCode": null
}
```

**2. Enterprise webhook** — Configured via `ENTERPRISE_WEBHOOK_URL` config:
```json
{
  "webhookUrl": "https://client.com/webhook",
  "authKey": "Authorization",
  "authValue": "Bearer xyz"
}
```

### Webhook Payload

**Success:**
```json
{
  "transactionId": "123456789012345678",
  "status": "Success",
  "errorCode": ""
}
```

**Failure:**
```json
{
  "transactionId": "123456789012345678",
  "status": "Failure",
  "errorCode": "DG1001",
  "message": "Access denied by user"
}
```

**Detailed webhook** (if `DETAILED_WEBHOOK` config is ACTIVE):
```json
{
  "transactionId": "...",
  "status": "Success",
  "data": {
    "uniqueId": "user_abc_123",
    "name": "John Doe",
    "maskedAdharNumber": "XXXX-XXXX-1234",
    "gender": "M",
    "dob": "01-01-1990",
    "address": { ... },
    "pdfLink": "base64...",
    "link": "base64...",
    "digilockerFileInfos": [...]
  }
}
```

### Retry Behavior

- Webhooks are sent **asynchronously** (don't block the main flow)
- If webhook fails, we retry **up to 2 times** with a **2-second delay**
- All attempts are logged in `digilocker_webhook_log`
- **No rollback** if webhook ultimately fails — the transaction is still SUCCESS

---

## 13. S3 File Storage — How Files Are Organized

All files are stored in S3 under a consistent path structure:

```
{eid}/{clientId}/{transactionId}/{docId}/{docType}.{extension}
```

**Example:**
```
101/mobile-app/123456789012345678/uuid-here/ADHAR.xml
101/mobile-app/123456789012345678/uuid-here/ADHAR.pdf
101/mobile-app/123456789012345678/uuid-here/PANCR.pdf
101/mobile-app/123456789012345678/photo.png
```

**Presigned URLs:** When a client calls `list-docs` or `download-docs`, we generate a temporary URL that expires in **4 hours**. This lets the client download the file without needing AWS credentials.

---

## 14. How to Debug a Problem

### Step 1: Get the Transaction ID

You need a `transactionId` to start debugging. Get it from:
- The client's request logs
- The SMS sent to the user
- The webhook payload they received

### Step 2: Check the Transaction Record

```sql
SELECT * FROM digilocker_transaction WHERE transaction_id = '123456789012345678';
```

Look at:
- `status` — what state is it in?
- `url_status` — is the URL still active?
- `error_code` — what error occurred?

### Step 3: Check the Activity Log

```sql
SELECT * FROM digilocker_activity_log 
WHERE transaction_id = '123456789012345678' 
ORDER BY id ASC;
```

This shows you exactly which step the flow reached. If `AADHAAR_XML_FETCHED_FROM_SOURCE` is logged but `FLOW_COMPLETED` is not, the problem happened during additional file downloads or PDF generation.

### Step 4: Check the Error Code

```sql
SELECT ec.error_code, ec.description 
FROM digilocker_transaction dt
JOIN digilocker_error_codes ec ON dt.error_code = ec.id
WHERE dt.transaction_id = '123456789012345678';
```

### Step 5: Check DigiLocker API Calls

```sql
SELECT * FROM digilocker_apis_log 
WHERE transaction_id = '123456789012345678' 
ORDER BY id ASC;
```

This shows exactly what DigiLocker returned at each API call.

### Step 6: Check Webhook Delivery

```sql
SELECT * FROM digilocker_webhook_log 
WHERE transaction_id = '123456789012345678' 
ORDER BY id ASC;
```

Shows if webhooks were sent, what status codes were returned, and how many retries happened.

### Common Problems and Their Causes

| Symptom | Likely Cause | Where to Look |
|---|---|---|
| Transaction stuck in `INITIATED` | User abandoned the flow | No further activity log entries after `KYC_INITIATED` |
| Transaction `FAILED` with DG1001 | User clicked "Deny" in DigiLocker | Activity log ends at `ACCESS_TOKEN_GENERATED` |
| Transaction `FAILED` with DG1002 | User's Aadhaar isn't linked to DigiLocker | Check DigiLocker API log for the doc list response |
| Transaction `SUCCESS` but no webhook received | Webhook URL wrong / client server down | Check `digilocker_webhook_log` |
| Enterprise can't fetch details | Transaction not SUCCESS yet | Check `status` column |
| URL shows "Link expired" | `url_status = INACTIVE` or JWT expired | Check `url_status` and `created_on` + expiry config |

---

## 15. Common Mistakes New Developers Make

**1. Confusing the two auth systems**  
Enterprise endpoints use Basic Auth. KYC flow endpoints use JWT. Don't mix them up.

**2. Not understanding the transaction-level vs enterprise-level config**  
Some settings come from `generate-url` request (per transaction), and some come from `digilocker_config` (enterprise-wide). Always check both.

**3. Forgetting that webhooks are async**  
`WebhookService.sendWebhook()` is marked `@Async`. It returns immediately — the actual sending happens in a separate thread. Don't assume the webhook is delivered when `processDigilockerDetails` returns.

**4. Directly reading `Optional.get()` without checking `isPresent()`**  
This will throw `NoSuchElementException` with no useful message. Always use `.orElseThrow()` or `.orElse()`.

**5. Changing configs in code instead of the database**  
If a client wants different behavior (different expiry, different webhook URL), the answer is almost always a config record in `digilocker_config` — not a code change.

**6. Assuming the S3 key is a URL**  
The `url` field in `digilocker_docs` is an S3 **key** (path), not a downloadable URL. You need to call `s3Service.generatePresignedUrl()` to get an actual URL.

**7. Forgetting that `url_status = INACTIVE` after completion**  
Once a transaction is SUCCESS or FAILED, the URL is disabled. If you're testing with the same URL repeatedly, generate a new one.

---

## 16. Adding a New Feature — Step-by-Step Checklist

When adding a new config-driven feature (most common task):

- [ ] Add the config name to `DigilockerConfig.ConfigName` enum
- [ ] Add a DB migration to document the new config name (or inform the DB team)
- [ ] Read the config in the appropriate service using `DigilockerConfigUtility` or `configRepository.findByEidAndClientIdAndConfigName()`
- [ ] Remember to handle the case where config doesn't exist (use defaults)
- [ ] Add the new feature to this documentation

When adding a new API endpoint:

- [ ] Decide if it's a Basic Auth (enterprise) or JWT (user flow) endpoint
- [ ] Add the controller method to the correct controller
- [ ] Register the new path in `SecurityConfiguration.java` under the correct chain
- [ ] Create request/response DTO classes
- [ ] Write the service method
- [ ] Log an activity event if it's a significant step
- [ ] Write unit tests for the service method
- [ ] Update this documentation

---

## 17. Glossary

| Term | Meaning |
|---|---|
| **Ent / Enterprise** | A company using our service (e.g., ABC Bank). Has an `eid`. |
| **EntClient** | A specific app/integration within an enterprise. Has `clientId` and `clientSecret`. |
| **Transaction** | One KYC verification attempt for one user. Has a unique 18-digit `transactionId`. |
| **DigiLocker** | India's government platform where citizens store their official documents. |
| **e-Aadhaar** | Digital version of the Aadhaar card, available as an XML document in DigiLocker. |
| **OAuth2** | Authorization framework used by DigiLocker. User gives permission, we get a code. |
| **PKCE** | Security extension to OAuth2 to prevent code interception attacks. |
| **Authorization Code** | Temporary code DigiLocker gives us after user approves. We exchange it for an access token. |
| **Access Token** | DigiLocker credential that lets us fetch the user's documents. Valid for a short time. |
| **JWT** | JSON Web Token — a signed token we use for KYC flow authentication. |
| **S3** | Amazon's cloud file storage (or compatible GCP storage). Where we store documents. |
| **Presigned URL** | A temporary URL that lets anyone download an S3 file without AWS credentials. Expires in 4 hours. |
| **Webhook** | An HTTP callback we send to the enterprise's server when a transaction completes. |
| **Tracking ID** | A UUID generated for each user session. Used to correlate events in activity logs. |
| **ADHAR** | DigiLocker document type code for Aadhaar. |
| **PANCR** | DigiLocker document type code for PAN card. |
| **DRVLC** | DigiLocker document type code for Driving License. |
| **Purge** | GDPR compliance — deleting user data after a certain period. |
| **HMAC** | Hash-based Message Authentication Code — DigiLocker sends this to verify the XML wasn't tampered with. |

---

*This document was last updated: 2026-04-12*  
*Maintained by: DKYC Backend Engineering Team*  
*For questions, raise a PR or contact the team.*
