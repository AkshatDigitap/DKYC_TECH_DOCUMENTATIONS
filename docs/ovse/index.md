# OVSE API Reference

**Version:** v1  
**Base URL:** `https://<host>/ovse/v1`  
**Format:** JSON (except `/callback` which is XML)

---

## Authentication

OVSE uses two distinct authentication mechanisms depending on the caller:

| Auth Type | Used By | How |
|-----------|---------|-----|
| **Basic Auth** | Enterprise server-to-server calls | `Authorization: Basic <base64(clientId:apiKey)>` |
| **Bearer JWT** | Frontend / SDK flows | `Authorization: Bearer <jwt>` |

The JWT is either a **URL token** (issued by `generate-url`, short-lived, carries the `transactionId` as `sub`) or a **flow token** (issued by `get-token`, carries flow context).

---

## Standard Response Envelope

All JSON APIs return responses wrapped in a common envelope:

```json
{
  "status": "success",
  "statusCode": 200,
  "result": { ... }
}
```

On error:

```json
{
  "status": "failure",
  "statusCode": 400,
  "errorCode": "INVALID_TOKEN",
  "message": "Invalid Token."
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"success"` or `"failure"` |
| `statusCode` | integer | HTTP status code |
| `errorCode` | string | Machine-readable error identifier (present on failure) |
| `message` | string | Human-readable error description (present on failure) |
| `result` | object | Response payload (present on success, null or absent on failure) |

---

## APIs

### 1. Generate URL

Creates an OVSE transaction and returns a URL that opens the Aadhaar App eKYC flow on the user's device.

**Endpoint:** `POST /ovse/v1/generate-url`  
**Auth:** Basic Auth (enterprise credentials)

#### Request Body

```json
{
  "uniqueId": "user_ref_12345",
  "authMode": "online",
  "channel": "APP",
  "appId": "com.example.myapp",
  "appSignature": "abc123signature",
  "requestedClaims": ["residentName", "dob", "gender", "address"],
  "profileHint": "mobile_hint",
  "langCode": "23",
  "expirySeconds": 3600,
  "mobile": "9876543210",
  "sendSms": false,
  "isHideExplanationScreen": false,
  "redirectionUrl": "https://example.com/kyc-done",
  "webhookUrl": "https://example.com/webhook/ovse"
}
```

| Field | Type | Required | Description                                                                                                                                         |
|-------|------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| `uniqueId` | string | Yes | Enterprise's own reference ID for this KYC session                                                                                                  |
| `authMode` | string | Yes | UIDAI auth mode. Accepted values: `online`, `face`                                                                                                  |
| `channel` | string | Yes | Delivery channel. Accepted values: `APP`, `WEB`, `SMS`                                                                                              |
| `appId` | string | Conditional | Android package name or iOS bundle ID. Required when `channel = APP`                                                                                |
| `appSignature` | string | Conditional | App signature for UIDAI's app verification. Required when `channel = APP`                                                                           |
| `requestedClaims` | string[] | No | List of Aadhaar claims to request. Defaults to enterprise's configured default claims                                                               |
| `profileHint` | string | No | Hint sent to Aadhaar App to pre-fill the profile selection                                                                                          |
| `langCode` | string | No | Language code for localized fields. Default: `"23"` (English)                                                                                       |
| `expirySeconds` | integer | No | URL validity in seconds. Falls back to enterprise config, then system default                                                                       |
| `mobile` | string | No | Mobile number to send the short URL to end user                                                                                                     |
| `sendSms` | boolean | No | Whether to send the generated URL as an SMS to the `mobile` number. Has no effect if `mobile` is not provided                                       |
| `isHideExplanationScreen` | boolean | No | Suppresses the UIDAI explanation screen before consent                                                                                              |
| `redirectionUrl` | string | No | URL to redirect the user after flow completion. Overrides enterprise config                                                                         |
| `webhookUrl` | string | No | Transaction-level webhook URL. If provided, success/failure notifications are posted here (no auth header). Overrides enterprise-configured webhook |

#### Success Response `200 OK`

```json
{
  "status": "success",
  "statusCode": 200,
  "result": {
    "transactionId": "OVSE-20240115-ABC123",
    "shortUrl": "https://short.url/xyz",
    "url": "aadhaar://kyc?token=eyJhbGci...",
    "redirectionUrl": "https://example.com/kyc-done"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `transactionId` | string | Unique OVSE transaction ID — store this to retrieve results later |
| `shortUrl` | string | Shortened URL (send via SMS or show as QR) |
| `url` | string | Full Aadhaar deep-link URL |
| `redirectionUrl` | string | The effective redirection URL that will be used after flow completion |

#### Error Responses

| HTTP | `errorCode` | When |
|------|-------------|------|
| 400 | `INVALID_INPUT_ERROR` | Request validation failed (invalid `authMode`, `channel`, claim names, etc.) |
| 400 | `VALIDATION_ERROR` | Provided `uniqueId` already exists for this client |
| 401 | `UNAUTHORIZED` | Invalid or missing Basic Auth credentials |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server error |

---

### 2. Get Token

Validates the URL token and returns a short-lived **flow token** that the Aadhaar SDK uses for subsequent calls. This is the first call the frontend makes after the user opens the OVSE URL.

**Endpoint:** `POST /ovse/v1/get-token`  
**Auth:** Bearer JWT (URL token — the JWT embedded in the generated URL)

No request body.

#### Success Response `200 OK`

```json
{
  "status": "success",
  "statusCode": 200,
  "result": {
    "flowToken": "eyJhbGciOiJSUzI1NiJ9...",
    "redirectionUrl": "https://example.com/kyc-done",
    "hideExplanationScreen": false
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `flowToken` | string | Short-lived JWT for SDK-to-server communication. Pass to the Aadhaar SDK |
| `redirectionUrl` | string | Where the SDK should redirect the user after flow completion |
| `hideExplanationScreen` | boolean | Whether to hide the UIDAI consent/explanation screen |

#### Error Responses

| HTTP | `errorCode` | When |
|------|-------------|------|
| 400 | `INVALID_TOKEN` | URL token is invalid, does not match any transaction, or URL is inactive |
| 400 | `TRANSACTION_ALREADY_SUCCESS` | Transaction is already completed successfully |
| 400 | `TRANSACTION_ALREADY_FAILED` | Transaction has already failed |
| 401 | `UNAUTHORIZED` | Missing or malformed Authorization header |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server error |

---

### 3. Generate Intent

Issues the UIDAI **intent string** that the Aadhaar App uses to initiate the actual eKYC request. Called by the Aadhaar SDK after `get-token`.

**Endpoint:** `POST /ovse/v1/intent`  
**Auth:** Bearer JWT (flow token from `get-token`)

No request body.

#### Success Response `200 OK`

```json
{
  "status": "success",
  "statusCode": 200,
  "result": {
    "intent": "aadhaar://kyc?...&srd=...&pid=..."
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `intent` | string | UIDAI-signed intent string. Pass directly to the Aadhaar App |

#### Error Responses

| HTTP | `errorCode` | When |
|------|-------------|------|
| 400 | `INVALID_TOKEN` | Flow token is invalid, does not match any transaction, or URL is inactive |
| 401 | `UNAUTHORIZED` | Missing or malformed Authorization header |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server error |

---

### 4. UIDAI Callback

Receives the Aadhaar eKYC result from UIDAI after the user completes the Aadhaar App flow. **This endpoint is called by UIDAI, not by the enterprise.** The enterprise does not need to integrate with this endpoint.

**Endpoint:** `POST /ovse/v1/callback`  
**Content-Type:** `application/xml`  
**Response Content-Type:** `application/xml`  
**Auth:** None — UIDAI uses its own signing mechanism in the XML payload

#### Request Body (XML — sent by UIDAI)

**Success case** — authentication completed, SD-JWT present:
```xml
<Request>
  <TxnID>uidai-txn-id-xyz</TxnID>
  <Credential>header.payload.signature~disclosure1~disclosure2</Credential>
</Request>
```

**Failure case** — UIDAI sends an error code when authentication fails on their side:
```xml
<Request>
  <TxnID>uidai-txn-id-xyz</TxnID>
  <ErrorCode>301</ErrorCode>
  <ErrorMsg>User rejected the authentication request</ErrorMsg>
</Request>
```

**UIDAI Error Codes (3XX):**

| ErrorCode | Name | Scenario |
|-----------|------|----------|
| `301` | User Reject | User explicitly tapped Deny/Reject in mAadhaar |
| `302` | Face Liveness Fail | Face liveness detection failed after 3 attempts |
| `303` | User Abort | User pressed back, switched app, or closed mAadhaar |
| `304` | Face Mismatch | Face captured but does not match Aadhaar records |
| `305` | Biometric Lock | Biometric lock is enabled in UIDAI system for this user |
| `306` | FaceRD Server Down | FaceRD service is currently unavailable |
| `307` | HTTP Error | Network/API failure (timeout, 5xx, invalid response) |
| `308` | Callback Retry | Auth succeeded but callback delivery previously failed |

#### Response (XML — returned to UIDAI)

```xml
<Response>
  <TxnID>uidai-txn-id-xyz</TxnID>
  <ResponseCode>200</ResponseCode>
  <ResponseMsg>Success</ResponseMsg>
</Response>
```

UIDAI always receives a 200 ACK once the request is accepted (success or 3XX error). Internal processing failures (invalid XML, missing TxnID) return a 400-level ResponseCode.

> **Note:** The enterprise does not interact with this endpoint. A failure webhook fires to the enterprise automatically when a 3XX error is received.

---

### 5. App Callback (App-to-App Flow)

In the App-to-App flow, the mAadhaar app returns the result directly to the calling mobile app via `onActivityResult` (Android) or URL scheme (iOS) — UIDAI never calls our `/callback` endpoint. The mobile app must forward the result here.

**Endpoint:** `POST /ovse/v1/app-callback`  
**Content-Type:** `application/json`  
**Auth:** Bearer JWT (flow token from `get-token`)

#### Request Body

**Success case** — SD-JWT received from mAadhaar:
```json
{
  "sdJwt": "header.payload.signature~disclosure1~disclosure2"
}
```

**Failure case** — mAadhaar returned an error (user cancelled, face mismatch, etc.):
```json
{
  "errorCode": "303",
  "errorMessage": "Authentication process aborted by user"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sdJwt` | string | Conditional | SD-JWT string from mAadhaar. Required if `errorCode` is not provided |
| `errorCode` | string | Conditional | UIDAI error code (301–308). Required if `sdJwt` is not provided |
| `errorMessage` | string | No | Human-readable error message corresponding to `errorCode` |

> Either `sdJwt` or `errorCode` must be present. Sending both or neither returns a 400 error.

#### Success Response `200 OK`

```json
{
  "status": "success",
  "statusCode": 200,
  "result": null
}
```

#### Error Responses

| HTTP | `errorCode` | When |
|------|-------------|------|
| 400 | `INVALID_REQUEST` | Neither `sdJwt` nor `errorCode` provided, or intent was never generated |
| 400 | `INVALID_TOKEN` | Flow token is invalid or transaction not found |
| 400 | `TRANSACTION_ALREADY_SUCCESS` | Transaction already completed successfully |
| 400 | `TRANSACTION_ALREADY_FAILED` | Transaction has already failed |
| 400 | `CALLBACK_FAILED` | SD-JWT processing failed (expiry, verification failure, storage error) |
| 401 | `UNAUTHORIZED` | Missing or malformed Authorization header |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server error |

---

### 6. Transaction Status

Returns the current status of a transaction. Used by the frontend to poll for completion after the user returns from the Aadhaar App flow.

**Endpoint:** `GET /ovse/v1/status`  
**Auth:** Bearer JWT (flow token or URL token — carries `transactionId` as `sub`)

No request body.

#### Success Response — Pending/Initiated `200 OK`

```json
{
  "status": "success",
  "statusCode": 200,
  "result": {
    "status": "PENDING"
  }
}
```

#### Success Response — Completed `200 OK`

```json
{
  "status": "success",
  "statusCode": 200,
  "result": {
    "status": "SUCCESS"
  }
}
```

#### Success Response — Failed `200 OK`

```json
{
  "status": "success",
  "statusCode": 200,
  "result": {
    "status": "FAILED",
    "failureReason": "Transaction expired"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | Transaction status. One of: `CREATED`, `INITIATED`, `PENDING`, `SUCCESS`, `FAILED` |
| `failureReason` | string | Human-readable reason for failure. Only present when `status = FAILED`. Absent for all other statuses |

**Transaction Status Values**

| Value | Meaning |
|-------|---------|
| `CREATED` | URL generated, user has not yet opened it |
| `INITIATED` | User opened the URL (`get-token` was called) |
| `PENDING` | User completed the Aadhaar App flow, awaiting UIDAI callback |
| `SUCCESS` | UIDAI callback received and data stored successfully |
| `FAILED` | Transaction failed at some stage (see `failureReason`) |

**Common `failureReason` values:**

*UIDAI authentication failures (also have `errorCode` in webhook):*
- `"User rejected the authentication request"` — User tapped Deny (301)
- `"Face liveness verification failed after 3 attempts"` — Face liveness failure (302)
- `"Authentication process aborted by user"` — User pressed back or switched app (303)
- `"Face authentication failed due to mismatch with registered data"` — Face mismatch (304)
- `"Biometric authentication is locked for this user"` — Biometric lock enabled (305)
- `"FaceRD service is currently unavailable. Please try again later"` — Service down (306)
- `"Failed to complete authentication due to network or server error"` — Network error (307)
- `"Authentication completed but callback delivery failed. Retrying"` — Delivery retry (308)

*Internal processing failures (no `errorCode` in webhook):*
- `"Transaction expired"` — Intent window elapsed before UIDAI callback arrived
- `"Verification failed"` — SD-JWT signature or disclosure hash check failed
- `"Data storage error"` — Internal error persisting the verified claims
- `"Transaction failed"` — Generic fallback when reason cannot be determined

#### Error Responses

| HTTP | `errorCode` | When |
|------|-------------|------|
| 400 | `INVALID_TOKEN` | Token is invalid, expired, or belongs to a different enterprise |
| 401 | `UNAUTHORIZED` | Missing or malformed Authorization header |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server error |

---

### 6. Fetch Details (Frontend Flow)

Returns the full verified Aadhaar data for a completed transaction. Called by the **frontend** using the flow token after polling `status` and receiving `SUCCESS`.

**Endpoint:** `POST /ovse/v1/fetch-details`  
**Auth:** Bearer JWT (flow token — carries `transactionId` as `sub`)

No request body.

#### Success Response `200 OK`

```json
{
  "status": "success",
  "statusCode": 200,
  "result": {
    "txnId": "OVSE-20240115-ABC123",
    "credentialIssuingDate": "2019-05-10",
    "enrolmentDate": "2012-03-15",
    "enrolmentNumber": "1234/56789/12345",
    "isNRI": false,
    "resident": {
      "name": {
        "value": "Rahul Sharma",
        "local": "राहुल शर्मा"
      },
      "image": "/9j/4AAQSkZJRgAB...",
      "dob": "1990-07-22",
      "gender": "M",
      "ageFlags": {
        "above18": true,
        "above50": false,
        "above60": false,
        "above75": false
      }
    },
    "address": {
      "careOf": { "value": "S/O Rajesh Sharma", "local": "पुत्र राजेश शर्मा" },
      "building": { "value": "Block A, Flat 201", "local": null },
      "locality": { "value": "Sector 15", "local": "सेक्टर 15" },
      "street": { "value": "MG Road", "local": null },
      "landmark": { "value": "Near Metro Station", "local": null },
      "vtc": { "value": "New Delhi", "local": "नई दिल्ली" },
      "subDistrict": { "value": "Central Delhi", "local": null },
      "district": { "value": "New Delhi", "local": "नई दिल्ली" },
      "state": { "value": "Delhi", "local": "दिल्ली" },
      "poName": { "value": "New Delhi GPO", "local": null },
      "pincode": "110001",
      "fullAddress": {
        "value": "Flat 201, Block A, Sector 15, MG Road, New Delhi - 110001",
        "local": "फ्लैट 201, ब्लॉक ए, सेक्टर 15, नई दिल्ली - 110001"
      }
    },
    "contact": {
      "mobile": "9876543210",
      "maskedMobile": "XXXXXXX210",
      "email": "rahul@example.com",
      "maskedEmail": "r***@example.com"
    }
  }
}
```

#### Response Fields

**Top-level**

| Field | Type | Description |
|-------|------|-------------|
| `txnId` | string | OVSE transaction ID |
| `credentialIssuingDate` | string | Date the Aadhaar credential was issued (`YYYY-MM-DD`) |
| `enrolmentDate` | string | Date of Aadhaar enrolment (`YYYY-MM-DD`) |
| `enrolmentNumber` | string | Aadhaar enrolment number |
| `isNRI` | boolean | Whether the Aadhaar holder is an NRI |

All top-level credential fields are `null` if not requested.

**`resident` object** — `null` if no resident claims were requested

| Field | Type | Description |
|-------|------|-------------|
| `name` | LocalizedField | Full name in English (`value`) and regional script (`local`) |
| `image` | string | Base64-encoded JPEG photograph. Can be several hundred KB |
| `dob` | string | Date of birth in `YYYY-MM-DD` format |
| `gender` | string | `M`, `F`, or `T` (Male / Female / Transgender) |
| `ageFlags` | AgeFlagsDto | Boolean age threshold flags (see below) |

**`ageFlags` object** — `null` if no age-threshold claims were requested

| Field | Type | Description |
|-------|------|-------------|
| `above18` | boolean | Whether holder is 18 or older |
| `above50` | boolean | Whether holder is 50 or older |
| `above60` | boolean | Whether holder is 60 or older |
| `above75` | boolean | Whether holder is 75 or older |

**`address` object** — `null` if no address claims were requested

| Field | Type | Description |
|-------|------|-------------|
| `careOf` | LocalizedField | Care-of (e.g. "S/O", "D/O") |
| `building` | LocalizedField | House/apartment/flat number |
| `locality` | LocalizedField | Locality/area name |
| `street` | LocalizedField | Street name |
| `landmark` | LocalizedField | Nearby landmark |
| `vtc` | LocalizedField | Village/Town/City |
| `subDistrict` | LocalizedField | Sub-district |
| `district` | LocalizedField | District |
| `state` | LocalizedField | State |
| `poName` | LocalizedField | Post office name |
| `pincode` | string | 6-digit postal code (no regional variant) |
| `fullAddress` | LocalizedField | Full concatenated address string |

Each `LocalizedField` has:

| Field | Type | Description |
|-------|------|-------------|
| `value` | string | English value |
| `local` | string | Regional-language value (`null` if not requested or unavailable) |

**`contact` object** — `null` if no contact claims were requested

| Field | Type | Description |
|-------|------|-------------|
| `mobile` | string | Mobile number (`null` if not disclosed) |
| `maskedMobile` | string | Partially masked mobile (e.g. `XXXXXXX210`) |
| `email` | string | Email address (`null` if not disclosed) |
| `maskedEmail` | string | Partially masked email |

#### Error Responses

| HTTP | `errorCode` | When |
|------|-------------|------|
| 400 | `INVALID_TRANSACTION_ID` | Transaction not found or belongs to a different enterprise/client |
| 400 | `TRANSACTION_NOT_SUCCESS` | Transaction is not yet in `SUCCESS` state |
| 401 | `UNAUTHORIZED` | Missing or malformed Authorization header |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server error |

---

### 7. Get Details (Enterprise Server-to-Server)

Returns the full verified Aadhaar data for a completed transaction. Called by the **enterprise backend** using Basic Auth when the enterprise wants to retrieve data independently of the frontend flow.

**Endpoint:** `POST /ovse/v1/get-details`  
**Auth:** Basic Auth (enterprise credentials)

#### Request Body

```json
{
  "transactionId": "OVSE-20240115-ABC123"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transactionId` | string | Yes | The `transactionId` returned by `generate-url` |

#### Success Response `200 OK`

Same structure as `fetch-details` — see [Fetch Details](#6-fetch-details-frontend-flow) for the full response schema.

#### Error Responses

| HTTP | `errorCode` | When |
|------|-------------|------|
| 400 | `INVALID_TRANSACTION_ID` | `transactionId` not found or belongs to a different enterprise/client |
| 400 | `TRANSACTION_NOT_SUCCESS` | Transaction is not yet in `SUCCESS` state |
| 401 | `UNAUTHORIZED` | Invalid or missing Basic Auth credentials |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server error |

---

### 8. Get Config

Retrieves all active OVSE configuration entries for the authenticated enterprise client.

**Endpoint:** `GET /ovse/v1/config`  
**Auth:** Basic Auth (enterprise credentials)

No request body.

#### Success Response `200 OK`

```json
{
  "status": "success",
  "statusCode": 200,
  "result": [
    {
      "configName": "DEFAULT_EXPIRY_SECONDS",
      "configValue": "3600",
      "status": "ACTIVE"
    },
    {
      "configName": "ENTERPRISE_WEBHOOK_URL",
      "configValue": "{\"webhookUrl\":\"https://example.com/webhook\",\"authKey\":\"X-API-Key\",\"authValue\":\"secret\"}",
      "status": "ACTIVE"
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `configName` | string | Configuration key (see Config Names below) |
| `configValue` | string | Configuration value (format varies by config name) |
| `status` | string | `ACTIVE` or `INACTIVE` |

**Supported Config Names**

| Config Name | Value Format | Description |
|-------------|-------------|-------------|
| `DEFAULT_EXPIRY_SECONDS` | Integer string, e.g. `"3600"` | Default URL validity duration in seconds |
| `DEFAULT_CLAIMS` | JSON array string, e.g. `["residentName","dob"]` | Claims requested when none are specified in `generate-url` |
| `DEFAULT_REDIRECTION_URL` | URL string | Default post-flow redirect URL |
| `ENTERPRISE_WEBHOOK_URL` | JSON object (see below) | Webhook endpoint with optional auth header |

**`ENTERPRISE_WEBHOOK_URL` config value format:**

```json
{
  "webhookUrl": "https://example.com/webhook/ovse",
  "authKey": "X-Api-Key",
  "authValue": "your-secret-key"
}
```

`authKey` and `authValue` are optional. If omitted, the webhook POST has no auth header.

#### Error Responses

| HTTP | `errorCode` | When |
|------|-------------|------|
| 401 | `UNAUTHORIZED` | Invalid or missing Basic Auth credentials |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server error |

---

### 9. Save / Update Config

Creates or updates a configuration entry for the authenticated enterprise client.

**Endpoint:** `POST /ovse/v1/config`  
**Auth:** Basic Auth (enterprise credentials)

#### Request Body

```json
{
  "configName": "ENTERPRISE_WEBHOOK_URL",
  "configValue": "{\"webhookUrl\":\"https://example.com/webhook\",\"authKey\":\"X-API-Key\",\"authValue\":\"secret\"}",
  "configStatus": "ACTIVE"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `configName` | string | Yes | One of the supported config names (see Get Config section) |
| `configValue` | string | Yes | The value to store. Format depends on `configName` |
| `configStatus` | string | No | `ACTIVE` or `INACTIVE`. Defaults to `ACTIVE` |

#### Success Response `200 OK`

```json
{
  "status": "success",
  "statusCode": 200,
  "result": null
}
```

#### Error Responses

| HTTP | `errorCode` | When |
|------|-------------|------|
| 400 | `INVALID_PAYLOAD` | Unsupported `configName`, invalid `configStatus`, or invalid value format |
| 401 | `UNAUTHORIZED` | Invalid or missing Basic Auth credentials |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server error |

---

## Webhook Notifications

If a webhook URL is configured (either at the transaction level via `generate-url.webhookUrl` or at the enterprise level via the `ENTERPRISE_WEBHOOK_URL` config), OVSE will POST a notification after the transaction reaches a terminal state.

Webhooks fire asynchronously after the transaction is committed. Up to 3 delivery attempts are made (5-second delay between attempts). Retries happen on 5xx responses and connection failures. 4xx responses are not retried.

### Success Webhook

Posted when `status = SUCCESS`. Payload is the same structure as the `get-details` / `fetch-details` response (see [Response Fields](#response-fields)).

```json
{
  "txnId": "OVSE-20240115-ABC123",
  "credentialIssuingDate": "2019-05-10",
  "resident": {
    "name": { "value": "Rahul Sharma", "local": "राहुल शर्मा" },
    "dob": "1990-07-22",
    "gender": "M"
  },
  "address": { ... },
  "contact": { ... }
}
```

### Failure Webhook

Posted when `status = FAILED`.

**UIDAI authentication failure** (errorCode present):
```json
{
  "transactionId": "OVSE-20240115-ABC123",
  "status": "FAILED",
  "failureReason": "User rejected the authentication request",
  "errorCode": "301"
}
```

**Internal processing failure** (errorCode absent):
```json
{
  "transactionId": "OVSE-20240115-ABC123",
  "status": "FAILED",
  "failureReason": "Transaction expired"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `transactionId` | string | OVSE transaction ID |
| `status` | string | Always `"FAILED"` |
| `failureReason` | string | Human-readable reason. `"Transaction failed"` if reason could not be determined |
| `errorCode` | string | UIDAI error code (301–308). Present only for UIDAI-side auth failures. Absent for internal failures (expiry, SD-JWT invalid, storage error) |

### Webhook Auth (Enterprise-Configured Webhooks Only)

When using the `ENTERPRISE_WEBHOOK_URL` config with `authKey`/`authValue`, those are sent as an HTTP header on every webhook POST:

```
POST https://example.com/webhook/ovse
X-Api-Key: your-secret-key
Content-Type: application/json

{ ... }
```

Transaction-level webhook URLs (from `generate-url.webhookUrl`) receive plain POST requests with no auth header.

---

## Requested Claims Reference

The following claim keys can be passed in `requestedClaims` in `generate-url` or configured via `DEFAULT_CLAIMS`:

### Identity Claims

| Claim Key | Description |
|-----------|-------------|
| `residentName` | Full name in English |
| `localResidentName` | Full name in regional script |
| `dob` | Date of birth |
| `gender` | Gender (M / F / T) |
| `residentImage` | Photograph (Base64 JPEG) |
| `ageAbove18` | Age threshold: 18+ |
| `ageAbove50` | Age threshold: 50+ |
| `ageAbove60` | Age threshold: 60+ |
| `ageAbove75` | Age threshold: 75+ |

### Address Claims

| Claim Key | Description |
|-----------|-------------|
| `careOf` | Care-of field (English) |
| `localCareOf` | Care-of field (regional script) |
| `building` | House/flat number (English) |
| `localBuilding` | House/flat number (regional script) |
| `locality` | Locality name (English) |
| `localLocality` | Locality name (regional script) |
| `street` | Street name (English) |
| `localStreet` | Street name (regional script) |
| `landmark` | Landmark (English) |
| `localLandmark` | Landmark (regional script) |
| `vtc` | Village/Town/City (English) |
| `localVtc` | Village/Town/City (regional script) |
| `subDistrict` | Sub-district (English) |
| `localSubDistrict` | Sub-district (regional script) |
| `district` | District (English) |
| `localDistrict` | District (regional script) |
| `state` | State (English) |
| `localState` | State (regional script) |
| `poName` | Post office name (English) |
| `localPoName` | Post office name (regional script) |
| `pincode` | Postal code |
| `address` | Full address string (English) |
| `regionalAddress` | Full address string (regional script) |

### Contact Claims

| Claim Key | Description |
|-----------|-------------|
| `mobile` | Mobile number |
| `maskedMobile` | Masked mobile number |
| `email` | Email address |
| `maskedEmail` | Masked email address |

### Credential Claims

| Claim Key | Description |
|-----------|-------------|
| `isNRI` | NRI flag |
| `credentialIssuingDate` | Date the credential was issued |
| `enrolmentDate` | Aadhaar enrolment date |
| `enrolmentNumber` | Aadhaar enrolment number |

---

## Error Code Reference

| HTTP | `errorCode` | Description |
|------|-------------|-------------|
| 400 | `INVALID_TOKEN` | JWT is invalid, does not match any transaction, or URL is inactive |
| 400 | `INVALID_INPUT_ERROR` | Request body validation failed (invalid field values) |
| 400 | `VALIDATION_ERROR` | Business rule violation (e.g. duplicate `uniqueId`) |
| 400 | `TRANSACTION_ALREADY_SUCCESS` | Transaction is already completed successfully |
| 400 | `TRANSACTION_ALREADY_FAILED` | Transaction has already failed |
| 400 | `INVALID_TRANSACTION_ID` | Provided `transactionId` not found or belongs to a different client |
| 400 | `TRANSACTION_NOT_SUCCESS` | Data fetch attempted before transaction reaches `SUCCESS` state |
| 400 | `INVALID_REQUEST` | App callback: neither `sdJwt` nor `errorCode` provided, or intent not generated |
| 400 | `CALLBACK_FAILED` | App callback: SD-JWT processing failed after successful receipt |
| 400 | `INVALID_PAYLOAD` | Invalid config name, status, or value format (config API only) |
| 401 | `UNAUTHORIZED` | Authentication credentials are missing or invalid |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected error — contact Digitap support |
