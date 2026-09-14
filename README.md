# Partner Integration Guide - Account and Card Onboarding APIs

**Version:** 1.1.0  
**Last Updated:** May 2026  
**Base URL:** `https://<host>/api/v2`  
**Audience:** Authorized JDB partners

This document covers only these partner-facing APIs:

- Request account opening with card.
- Resubmit a rejected account/card opening request.
- List account/card opening requests submitted by the partner.

---

## 1. Integration Overview

### API Flow

```text
1. POST /login
   Obtain a JWT Bearer token.

2. Optional reference lookups
   GET  /prefix
   GET  /gender
   GET  /ISDCode
   GET  /country
   POST /getAddressE
   These help partners build valid request values.

3. POST /request/account/open
   Submit a new account opening and card request.

4. POST /account/resubmit
   Resubmit a rejected account/card request after correcting information.

5. GET /getAllListOpening
   List account/card opening requests submitted by the authenticated partner.
```

### Common Response Shape

Most endpoints return JSON in this format:

```json
{
  "message": "Human-readable result message",
  "responseCode": "00",
  "status": true
}
```

| Field | Type | Description |
|---|---:|---|
| `message` | string | Human-readable result or error message. |
| `responseCode` | string | Business response code. `00` means success unless stated otherwise. |
| `status` | boolean | `true` when the request is accepted or completed successfully. |

---

## 2. Authentication and Security

### Required Authentication Methods

Partner APIs use two credentials:

| Credential | Sent As | Description |
|---|---|---|
| Encrypted API key | `api-key` header | AES-encrypted partner identity issued by JDB. |
| JWT token | `Authorization: Bearer <token>` | Returned by `POST /login`. Valid for 1 hour. |

### API Key

JDB provides the partner API key as an encrypted value. Partners should store and send this value exactly as issued.

Internally, the decrypted API key contains partner identity fields similar to:

```json
{
  "partnerId": 123456,
  "partnerName": "PARTNER_NAME",
  "unitCode": "UNIT01",
  "type": "AGENT",
  "custCategory": "CUSTOMER_CATEGORY",
  "accClass": "ACCOUNT_CLASS"
}
```

Do not put API keys, AES secrets, JWT tokens, or webhook secrets in mobile apps, browser code, logs, screenshots, public repositories, or support tickets.

### Password Encryption for Login

The `password` field in `POST /login` must be AES-encrypted using the shared secret provided by JDB before it is sent.

Example plaintext password must not be sent:

```json
{
  "userName": "partner_user",
  "password": "plain_password",
  "actionNode": "1"
}
```

Send the encrypted password instead:

```json
{
  "userName": "partner_user",
  "password": "U2FsdGVkX1...",
  "actionNode": "1"
}
```

### Standard Headers

| Header | Required | Value |
|---|---:|---|
| `Content-Type` | Yes for requests with JSON body | `application/json` |
| `api-key` | Yes | Encrypted API key issued by JDB. |
| `Authorization` | Yes, except `POST /login` | `Bearer <jwt_token>` |

---

## 3. Login

### `POST /login`

Use this endpoint to obtain a JWT token.

**Auth required:** `api-key` only  
**Rate limit:** 10 requests per 15 minutes per IP  
**Token expiry:** 1 hour

### Request

```http
POST /api/v2/login HTTP/1.1
Host: <host>
Content-Type: application/json
api-key: <encrypted_api_key>
```

```json
{
  "userName": "partner_user",
  "password": "<AES_encrypted_password>",
  "actionNode": "1"
}
```

### Request Fields

| Field | Type | Required | Max Length | Rules |
|---|---:|---:|---:|---|
| `userName` | string | Yes | 20 | Partner username provided by JDB. |
| `password` | string | Yes | 60 | AES-encrypted password. |
| `actionNode` | string | Yes | 1 | Must be `"1"`. |

### Success Response

**HTTP:** `200 OK`

```json
{
  "response": [
    {
      "data": {
        "id": 123456,
        "name": "Partner User",
        "mail": "partner@example.com",
        "phone": "2055512345"
      },
      "status": true,
      "message": "Successfully",
      "responseCode": "00",
      "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
    }
  ]
}
```

Save `response[0].token` and send it in later requests:

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Login Errors

| HTTP | `responseCode` | Meaning | Partner Action |
|---:|---|---|---|
| 400 | `01` | Missing required input. | Check `userName`, `password`, `actionNode`, and `api-key`. |
| 400 | `31` | Field length exceeded. | Shorten the field shown in the message. |
| 400 | `32` | Invalid action node. | Send `actionNode: "1"`. |
| 401 | `02` | Username not found. | Verify username with JDB. |
| 401 | `03` | Password is incorrect. | Check password and AES encryption. |
| 401 | `09` | Account is not activated. | Contact JDB. |
| 401 | `10` | Account is locked. | Contact JDB. |
| 401 | `11` | Partner API key is invalid. | Verify the API key with JDB. |
| 500 | `99` | Internal server error. | Retry later or contact JDB. |

---

## 4. Reference Data APIs

Reference APIs are useful for building valid values for account/card opening requests.

**Auth required:** `api-key` + `Authorization: Bearer <token>`

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/prefix` | Valid title/prefix values. |
| `GET` | `/gender` | Valid gender values. |
| `GET` | `/ISDCode` | International dialing codes. |
| `GET` | `/country` | Country codes. |
| `POST` | `/getAddressE` | Province, district, and village names in English. |

Partners should cache these values and refresh them periodically.

Example request:

```http
GET /api/v2/prefix HTTP/1.1
Host: <host>
api-key: <encrypted_api_key>
Authorization: Bearer <jwt_token>
```

Example response:

```json
{
  "status": true,
  "message": "Successfully",
  "responseCode": "00",
  "data": [
    {
      "prefixCode": "MR",
      "prefixName": "Mr"
    }
  ]
}
```

---

## 5. Request Account Opening with Card

### `POST /request/account/open`

This endpoint lets a partner submit a new account opening and card request. The submitted information is saved for JDB staff review. If accepted, the response returns a `batch_no` that must be stored by the partner for tracking and webhook correlation.

**Auth required:** `api-key` + `Authorization: Bearer <token>`  
**Rate limit:** 1,000 requests per 15 minutes per IP

### Request

```http
POST /api/v2/request/account/open HTTP/1.1
Host: <host>
Content-Type: application/json
api-key: <encrypted_api_key>
Authorization: Bearer <jwt_token>
```

```json
{
  "prefix": "Mr",
  "first_name": "John",
  "last_name": "Doe",
  "gender": "M",
  "dob": "1990-01-15",
  "place_of_birth": "Vientiane",
  "birth_country": "LA",
  "village": "Simuang",
  "district": "Chanthabouly",
  "province": "Vientiane Capital",
  "ISDNo": "+856",
  "telephone": "2055512345",
  "mail": "john.doe@example.com",
  "occupation": "Engineer",
  "document_category": "NATIONAL_ID",
  "document_type": "NID",
  "document_desc": "National Identity Card",
  "document_id": "LA1234567",
  "document_issued_date": "2020-01-01",
  "document_expiry_date": "2030-01-01",
  "card_product_type": "VSDC_AG_C",
  "currency": "USD",
  "minimum_amount": "100",
  "card_first_name": "JOHN",
  "card_last_name": "DOE",
  "user_internet_banking": "N",
  "take_photo_card": "<base64_string>",
  "take_photo_with_card": "<base64_string>",
  "signature": "<base64_string>"
}
```

### Mandatory Fields

| Field | Type | Max Length | Description |
|---|---:|---:|---|
| `prefix` | string | 5 | Customer title or prefix. |
| `first_name` | string | 50 | Customer first name in English. |
| `last_name` | string | 50 | Customer last name in English. |
| `gender` | string | 1 | Gender code, for example `M` or `F`. |
| `dob` | string | 20 | Date of birth. Format: `YYYY-MM-DD`. |
| `place_of_birth` | string | 50 | Customer place of birth. |
| `birth_country` | string | 10 | Country code. |
| `village` | string | 50 | Current village. |
| `district` | string | 50 | Current district. |
| `province` | string | 50 | Current province. |
| `ISDNo` | string | 4 | International dialing code, for example `+856`. |
| `telephone` | string | 20 | Customer phone number. |
| `mail` | string | 50 | Valid email address. |
| `occupation` | string | 100 | Customer occupation. |
| `document_category` | string | 100 | Document category. |
| `document_type` | string | 100 | Document type code. |
| `document_desc` | string | 100 | Document description. |
| `document_id` | string | 20 | Document number. |
| `document_issued_date` | string | 20 | Document issue date. Format: `YYYY-MM-DD`. |
| `document_expiry_date` | string | 20 | Document expiry date. Format: `YYYY-MM-DD`; must be after the current date. |
| `card_product_type` | string | 10 | Card product type configured for the partner. |
| `currency` | string | 3 | Must be `USD` in the current validation logic. |
| `minimum_amount` | string or number | 7 | Minimum opening balance amount. |
| `card_first_name` | string | 50 | Embossed first name. English letters and spaces only. |
| `card_last_name` | string | 50 | Embossed last name. English letters and spaces only. |
| `take_photo_card` | string | See image rules | Base64 image of the identity document. |
| `take_photo_with_card` | string | See image rules | Base64 image of customer holding the identity document. |
| `signature` | string | See image rules | Base64 signature image. |

### Optional Fields

| Field | Type | Default | Description |
|---|---:|---|---|
| `user_internet_banking` | string | `N` when omitted or empty | `Y` to enroll in internet banking, `N` otherwise. |

### Validation Rules

| Field | Rule |
|---|---|
| `api-key` | Must decrypt to valid partner information. |
| `Authorization` | Must contain a valid JWT token. |
| `card_product_type` | Must exist and be active for the authenticated partner. |
| `mail` | Must be a valid email format. |
| `currency` | Must be `USD` in the current validation logic. |
| `dob`, `document_issued_date`, `document_expiry_date` | Must be real dates in `YYYY-MM-DD` format. |
| `document_expiry_date` | Must be after the current date. |
| `card_first_name`, `card_last_name` | English letters and spaces only. |
| `card_first_name` + `card_last_name` | Combined length must not exceed 20 characters. |
| Image fields | Must be Base64 images, maximum 5 MB decoded size per image. |

### Image Encoding

Image fields must contain Base64 image content. JPEG and PNG are supported.

The API currently strips a `data:image/...;base64,` prefix if one is supplied, but partners should send pure Base64 to keep payloads consistent.

Correct:

```text
/9j/4AAQSkZJRgABAQEASABIAAD...
```

Avoid:

```text
data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEASABIAAD...
```

| Image Rule | Value |
|---|---|
| Max decoded file size | 5 MB per image |
| Supported formats | JPEG, PNG |
| Transport | JSON string |
| API payload limit | 50 MB total request body |

### Success Response

**HTTP:** `200 OK`

```json
{
  "message": "Successful request; kindly wait as staff checks information.",
  "responseCode": "00",
  "status": true,
  "batch_no": 748291034,
  "transationId": 748291034
}
```

Save `batch_no`. It is the main reference for staff review and webhook callbacks.

### Error Responses

| HTTP | `responseCode` | Meaning | Partner Action |
|---:|---|---|---|
| 400 | `01` | Missing required field. | Add the field shown in `message`. |
| 400 | `04` | Invalid field value, email, card name, document data, or internet banking value. | Fix the invalid value. |
| 400 | `11` | Card product type is not valid for this partner. | Verify `card_product_type` with JDB. |
| 400 | `24` | Card embossing name contains unsupported characters. | Use English letters and spaces only. |
| 400 | `25` | Invalid document issue date. | Send `YYYY-MM-DD`. |
| 400 | `26` | Document has expired. | Use a valid, non-expired document. |
| 400 | `28` | Combined card first and last name exceeds 20 characters. | Shorten embossing name. |
| 400 | `29` | Invalid currency or internet banking value. | Use `currency: "USD"` and `user_internet_banking: "Y"` or `"N"`. |
| 400 | `30` | One or more date fields have invalid format. | Send real dates in `YYYY-MM-DD`. |
| 400 | `31` | Field length exceeded. | Shorten fields shown in `message`. |
| 400 | `ERR_MISSING_IMAGE` | One or more required image fields are missing. | Send all three image fields. |
| 401 | `05` | Authorization header is missing. | Send `Authorization: Bearer <token>`. |
| 401 | `11` | Invalid partner API key. | Verify API key with JDB. |
| 500 | `06` | JWT verification failed. | Re-authenticate and retry. |
| 500 | `12` | JWT token expired. | Re-authenticate and retry. |
| 500 | `ERR_MAIL_CONFIG` | Partner or unit email configuration is missing. | Contact JDB. |
| 500 | `99` | Internal server error. | Retry later or contact JDB. |

### Stored Procedure Response Codes

The endpoint saves the request through `P_Manage_Req_Register_Account_Card_Api`. Non-`00` results are returned with HTTP `400`.

| `responseCode` | Meaning |
|---|---|
| `00` | Request saved successfully. |
| `01` | Duplicate `idFile` in link file table. |
| `02` | Invalid link status. |
| `03` | Batch number is null. |
| `05` | Duplicate link URL or batch number. |
| `06` | Invalid status. |
| `07` | Duplicate short name and card type combination. |
| `08` | Invalid action. |
| `09` | Invalid card type. |
| `10` | Invalid currency code. |
| `11` | Resubmit is invalid; only rejected records can be resubmitted. |
| `99` | System error. |

---

## 6. Resubmit Account/Card Opening Request

### `POST /account/resubmit`

This endpoint lets a partner resubmit a previously rejected account/card opening request after correcting the customer information.

**Auth required:** `api-key` + `Authorization: Bearer <token>`  
**Rate limit:** 1,000 requests per 15 minutes per IP

### Request

```http
POST /api/v2/account/resubmit HTTP/1.1
Host: <host>
Content-Type: application/json
api-key: <encrypted_api_key>
Authorization: Bearer <jwt_token>
```

```json
{
  "batch_no": 748291034,
  "prefix": "Mr",
  "first_name": "John",
  "last_name": "Doe",
  "gender": "M",
  "dob": "1990-01-15",
  "place_of_birth": "Vientiane",
  "birth_country": "LA",
  "village": "Simuang",
  "district": "Chanthabouly",
  "province": "Vientiane Capital",
  "ISDNo": "+856",
  "telephone": "2055512345",
  "mail": "john.doe@example.com",
  "occupation": "Engineer",
  "document_category": "NATIONAL_ID",
  "document_type": "NID",
  "document_desc": "National Identity Card",
  "document_id": "LA1234567",
  "document_issued_date": "2020-01-01",
  "document_expiry_date": "2030-01-01",
  "card_product_type": "VSDC_AG_C",
  "currency": "USD",
  "minimum_amount": "100",
  "card_first_name": "JOHN",
  "card_last_name": "DOE",
  "user_internet_banking": "N",
  "take_photo_card": "<base64_string>",
  "take_photo_with_card": "<base64_string>",
  "signature": "<base64_string>"
}
```

### Mandatory Fields

All fields from `POST /request/account/open` are required, plus:

| Field | Type | Max Length | Description |
|---|---:|---:|---|
| `batch_no` | string or number | 10 | Batch number returned by the original request. Must not be `0`, `null`, or empty. |

### Important Rule

Only records rejected by JDB staff can be resubmitted. If the original request is not in a rejected state, the stored procedure returns `responseCode: "11"`.

### Success Response

**HTTP:** `200 OK`

```json
{
  "message": "Successful request; kindly wait as staff checks information.",
  "responseCode": "00",
  "status": true,
  "batch_no": 748291034,
  "transationId": 748291034
}
```

### Error Responses

| HTTP | `responseCode` | Meaning | Partner Action |
|---:|---|---|---|
| 400 | `ERR_BATCH_NO` | `batch_no` is missing, null, or zero. | Send the original rejected `batch_no`. |
| 400 | `01` | Missing required field. | Add the field shown in `message`. |
| 400 | `04` | Invalid field value, email, card name, document data, or internet banking value. | Fix the invalid value. |
| 400 | `11` | Request is not eligible for resubmit. | Resubmit only rejected requests. |
| 400 | `31` | Field length exceeded. | Shorten fields shown in `message`. |
| 400 | `ERR_MISSING_IMAGE` | One or more required image fields are missing. | Send all three image fields. |
| 401 | `11` | Invalid partner API key. | Verify API key with JDB. |
| 500 | `99` | Internal server error. | Retry later or contact JDB. |

---

## 7. List Account/Card Opening Requests

### `GET /getAllListOpening`

Returns account/card opening requests submitted by the authenticated partner.

**Auth required:** `api-key` + `Authorization: Bearer <token>`  
**Rate limit:** No route-specific limiter in the current route definition

### Request

```http
GET /api/v2/getAllListOpening HTTP/1.1
Host: <host>
api-key: <encrypted_api_key>
Authorization: Bearer <jwt_token>
```

No request body is required.

### Success Response

**HTTP:** `200 OK`

```json
{
  "message": "Successfully",
  "responseCode": "00",
  "status": true,
  "data": [
    {
      "batch_no": 748291034,
      "partner_id": 123456,
      "status": "REQUEST",
      "created_at": "2026-05-15T03:00:00.000Z"
    }
  ]
}
```

The exact fields in each `data[]` item come from the database view `v_get_list_customer_re_opent_account`.

### Error Responses

| HTTP | `responseCode` | Meaning | Partner Action |
|---:|---|---|---|
| 401 | `11` | Invalid partner API key. | Verify API key with JDB. |
| 500 | `06` | JWT verification failed. | Re-authenticate and retry. |
| 500 | `12` | JWT token expired. | Re-authenticate and retry. |
| 500 | `99` | Internal server error. | Retry later or contact JDB. |

---

## 8. Rate Limits

| Endpoint Group | Window | Limit per IP | Exceeded Response |
|---|---:|---:|---|
| `POST /login` | 15 minutes | 10 | `{ "error": "Too many login attempts, please try again later." }` |
| `POST /request/account/open` | 15 minutes | 1,000 | `{ "error": "Too many requests, please try again later." }` |
| `POST /account/resubmit` | 15 minutes | 1,000 | `{ "error": "Too many requests, please try again later." }` |
| `GET /getAllListOpening` | No route-specific limiter | Not configured | Protected by JWT and API key only. |

Rate limited requests return HTTP `429 Too Many Requests`.

---

## 9. Webhook Notifications

When an account/card opening request is accepted, the system records a webhook job for the partner webhook URL registered with JDB.

Webhook delivery is asynchronous. A successful API response means the request was saved; it does not guarantee the partner webhook endpoint has already received the notification.

### Account Opening Webhook Payload

```json
{
  "type": "APPLICATION_REQUEST",
  "batchNo": 748291034,
  "idFrom": 512847361
}
```

Depending on partner webhook configuration, the outbound webhook job may include partner metadata such as `partnerId`, registered API key, public key, secret key, request status, and token for signing or delivery.

Partner webhook requirements:

| Requirement | Description |
|---|---|
| HTTP method | `POST` |
| Content type | `application/json` |
| Success acknowledgement | Return HTTP `200` after processing or safely queuing the event. |
| Idempotency | Handle duplicate webhook deliveries by using `batchNo`. |
| Security | Verify any signature or secret agreed with JDB. |

---

## 10. Error Code Reference

### Global Codes

| `responseCode` | Meaning |
|---|---|
| `00` | Success. |
| `01` | Required input is missing. |
| `02` | Username not found or invalid link status depending on endpoint. |
| `03` | Password/bcrypt validation failed or batch number is null depending on endpoint. |
| `04` | Invalid data or invalid format. |
| `05` | Data is null, authorization header missing, or duplicate link URL/batch number depending on endpoint. |
| `06` | JWT verification failed or invalid status depending on endpoint. |
| `07` | Duplicate short name and card type combination. |
| `08` | Invalid action. |
| `09` | Account not activated or invalid card type depending on endpoint. |
| `10` | Account locked or invalid currency code depending on endpoint. |
| `11` | Partner API key invalid, card type invalid for partner, or resubmit not allowed depending on endpoint. |
| `12` | JWT token expired. |
| `24` | Invalid card embossing name. |
| `25` | Invalid document issue date. |
| `26` | Document expired. |
| `28` | Embossing name too long. |
| `29` | Invalid currency or internet banking value. |
| `30` | Date format invalid. |
| `31` | Field length exceeded. |
| `32` | Invalid login action node. |
| `99` | Internal server error. |

### Endpoint-Specific Codes

| `responseCode` | Endpoint | Meaning |
|---|---|---|
| `ERR_MISSING_IMAGE` | `/request/account/open`, `/account/resubmit` | One or more image fields are missing. |
| `ERR_MAIL_CONFIG` | `/request/account/open` | Partner or unit email configuration is missing. |
| `ERR_BATCH_NO` | `/account/resubmit` | `batch_no` is missing, null, or zero. |

---

## 11. cURL Examples

### Login

```bash
curl -X POST "https://<host>/api/v2/login" \
  -H "Content-Type: application/json" \
  -H "api-key: <encrypted_api_key>" \
  -d '{
    "userName": "partner_user",
    "password": "<AES_encrypted_password>",
    "actionNode": "1"
  }'
```

### Request Account Opening with Card

```bash
curl -X POST "https://<host>/api/v2/request/account/open" \
  -H "Content-Type: application/json" \
  -H "api-key: <encrypted_api_key>" \
  -H "Authorization: Bearer <jwt_token>" \
  -d '{
    "prefix": "Mr",
    "first_name": "John",
    "last_name": "Doe",
    "gender": "M",
    "dob": "1990-01-15",
    "place_of_birth": "Vientiane",
    "birth_country": "LA",
    "village": "Simuang",
    "district": "Chanthabouly",
    "province": "Vientiane Capital",
    "ISDNo": "+856",
    "telephone": "2055512345",
    "mail": "john.doe@example.com",
    "occupation": "Engineer",
    "document_category": "NATIONAL_ID",
    "document_type": "NID",
    "document_desc": "National Identity Card",
    "document_id": "LA1234567",
    "document_issued_date": "2020-01-01",
    "document_expiry_date": "2030-01-01",
    "card_product_type": "VSDC_AG_C",
    "currency": "USD",
    "minimum_amount": "100",
    "card_first_name": "JOHN",
    "card_last_name": "DOE",
    "user_internet_banking": "N",
    "take_photo_card": "<base64_string>",
    "take_photo_with_card": "<base64_string>",
    "signature": "<base64_string>"
  }'
```

### Resubmit Account/Card Opening Request

```bash
curl -X POST "https://<host>/api/v2/account/resubmit" \
  -H "Content-Type: application/json" \
  -H "api-key: <encrypted_api_key>" \
  -H "Authorization: Bearer <jwt_token>" \
  -d '{
    "batch_no": 748291034,
    "prefix": "Mr",
    "first_name": "John",
    "last_name": "Doe",
    "gender": "M",
    "dob": "1990-01-15",
    "place_of_birth": "Vientiane",
    "birth_country": "LA",
    "village": "Simuang",
    "district": "Chanthabouly",
    "province": "Vientiane Capital",
    "ISDNo": "+856",
    "telephone": "2055512345",
    "mail": "john.doe@example.com",
    "occupation": "Engineer",
    "document_category": "NATIONAL_ID",
    "document_type": "NID",
    "document_desc": "National Identity Card",
    "document_id": "LA1234567",
    "document_issued_date": "2020-01-01",
    "document_expiry_date": "2030-01-01",
    "card_product_type": "VSDC_AG_C",
    "currency": "USD",
    "minimum_amount": "100",
    "card_first_name": "JOHN",
    "card_last_name": "DOE",
    "user_internet_banking": "N",
    "take_photo_card": "<base64_string>",
    "take_photo_with_card": "<base64_string>",
    "signature": "<base64_string>"
  }'
```

### List Account/Card Opening Requests

```bash
curl -X GET "https://<host>/api/v2/getAllListOpening" \
  -H "api-key: <encrypted_api_key>" \
  -H "Authorization: Bearer <jwt_token>"
```

---

## 12. Go-Live Checklist

- [ ] Receive encrypted `api-key` from JDB.
- [ ] Receive partner username and password encryption secret from JDB.
- [ ] Register webhook URL and webhook secret/public-key material with JDB.
- [ ] Implement `POST /login` and token renewal every 1 hour or when token errors occur.
- [ ] Cache reference data for prefix, gender, country, ISD, and English address.
- [ ] Confirm valid `card_product_type` values with JDB before using `/request/account/open` or `/account/resubmit`.
- [ ] Validate all mandatory fields before sending requests.
- [ ] Validate dates using `YYYY-MM-DD`.
- [ ] Ensure document expiry date is in the future.
- [ ] Ensure card embossing names use English letters and spaces only.
- [ ] Ensure combined card first and last embossing name is 20 characters or fewer.
- [ ] Send images as Base64 and keep each decoded image under 5 MB.
- [ ] Store `batch_no` returned by `/request/account/open` and `/account/resubmit`.
- [ ] Use `/getAllListOpening` to retrieve submitted account/card opening requests when needed.
- [ ] Return HTTP `200` from partner webhook endpoint after receiving events.
- [ ] Never log `api-key`, AES secrets, JWT tokens, or webhook secrets.

---

## 13. Support Information to Provide JDB

When contacting JDB support, include:

- Endpoint path.
- HTTP status.
- `responseCode`.
- `message`.
- `batch_no`.
- Request timestamp and environment.
- Sanitized request body with secrets and images removed.

Do not include API keys, JWT tokens, AES secrets, webhook secrets, or full Base64 images.



# JDB Events Notification System (Webhook API)

This document outlines the webhook notification system used by JDB to notify Partners about various asynchronous events. When a specific event occurs within the JDB system (e.g., an account is approved or a card is activated), a JSON payload is sent via an HTTP POST request to the Partner's configured webhook URL.

## 1. Webhook Endpoint Requirements
- **Method:** `POST`
- **Content-Type:** `application/json`
- **Header:** Must accept the `X-Signature` header for payload verification.

---

## 2. Security & Signature Verification

To ensure that the webhook payload is genuinely from JDB and has not been tampered with, JDB signs each request using HMAC SHA-256. 

### Sender Side (JDB)
JDB generates the signature using the JSON payload and a shared Secret Key:
```javascript
// HMAC signature generation
const crypto = require('crypto');

function generateHmacSignature(secret, payload) {
  const jsonPayload = JSON.stringify(payload);
  return crypto.createHmac("sha256", secret).update(jsonPayload).digest("hex");
}
```

### Recipient Side (Partner)
The Partner must verify the `X-Signature` header in the incoming webhook request:
```javascript
const crypto = require('crypto');

function verifyHmacSignature(payload, receivedSignature, secretKey) {
  const computedSignature = crypto
    .createHmac('sha256', secretKey)
    .update(JSON.stringify(payload))
    .digest('hex');
    
  return computedSignature === receivedSignature;
}

// Example Webhook Endpoint (Express.js)
app.post('/webhook', (req, res) => {
  const payload = req.body;
  const signature = req.headers['x-signature'];

  if (!signature || !verifyHmacSignature(payload, signature, SECRET_KEY)) {
    console.warn('Signature verification failed');
    return res.status(401).json({ success: false, message: 'Invalid signature' });
  }

  // Signature verified, process the payload
  console.log('Signature verified, processing data:', payload);

  // Reply to JDB
  return res.status(200).json({ success: true, code: 'SUCCESS', message: 'Webhook received' });
});
```

---

## 3. Events & Payloads

The `type` field in the payload determines the structure and context of the event. All payloads are published as JSON objects.

### 3.1. UPDATE_MEMBER_REQUEST
Fired when there is a request to update a member by card.
```json
{
  "type": "UPDATE_MEMBER_REQUEST",
  "batchNo": "12345",
  "idFrom": "98765"
}
```

### 3.2. CARD_APPLICATION_APPROVED
Fired when a partner's card application has been successfully approved.
```json
{
  "type": "CARD_APPLICATION_APPROVED",
  "batchNo": "12345",
  "idFrom": null,
  "accountNo": "121212121212121",
  "cardNo": "1234123412341234"
}
```

### 3.3. CARD_APPLICATION_REJECTED
Fired when a partner's card application is rejected.
```json
{
  "type": "CARD_APPLICATION_REJECTED",
  "batchNo": "12345",
  "idFrom": "98765",
  "reason": "Passport image is not valid"
}
```

### 3.4. CARD_ACTIVATED
Fired when a card has been successfully activated.
```json
{
  "type": "CARD_ACTIVATED",
  "batchNo": "12345",
  "cardId": "C-98765",
  "accountNo": "121212121212121"
}
```

### 3.5. IB_ACCOUNT_ACTIVATED
Fired when Internet Banking (IB) has been activated for an account.
```json
{
  "type": "IB_ACCOUNT_ACTIVATED",
  "batchNo": "12345",
  "idFrom": "98765",
  "reason": "Internet Banking activated successfully"
}
```

### 3.6. CMS_CARD_ISSUANCE
Fired for CMS card issuance status updates.
```json
{
  "type": "CMS_CARD_ISSUANCE",
  "batchRef": "REF12345",
  "recordId": "REC67890",
  "status": "ISSUED",
  "cardNumber": "1234123412341234",
  "failure_reason": "Any failure details here if applicable"
}
```
*(Note: `cardNumber` and `failure_reason` are optional depending on the `status`)*

### 3.7. CARD_ISSUANCE
Fired for general card issuance request status updates.
```json
{
  "type": "CARD_ISSUANCE",
  "requestId": "REQ12345",
  "request_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "status": "COMPLETED",
  "failure_reason": "Any failure details here if applicable"
}
```
*(Note: `failure_reason` is optional)*

### 3.8. CLOSURE_REQUEST
Fired when there is a status update regarding an account or card closure request.
```json
{
  "type": "CLOSURE_REQUEST",
  "closure_id": "CLS12345",
  "request_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "status": "APPROVED",
  "failure_reason": "Any failure details here if applicable"
}
```
*(Note: `failure_reason` is optional)*

---

## 4. Response Expectations
When the Partner's system receives the webhook event, it should respond with an HTTP `200 OK` status and a JSON payload to confirm receipt. JDB uses this to know if the message was delivered successfully.

**Expected Partner Response:**
```json
{
  "success": true,
  "code": "SUCCESS",
  "message": "Webhook received"
}
```

