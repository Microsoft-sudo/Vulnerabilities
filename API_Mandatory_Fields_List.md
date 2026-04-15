# BimaSugam API - Mandatory Fields List
**Note:** This document lists ONLY fields that will cause API errors (400 Bad Request) if missing.

## Motor Insurance APIs

### 1. POST /motor/get_quotes
**Description:** Get insurance quotes with premium calculation

**Mandatory Fields (will throw error if missing):**
- `lead_id` (string, max 255 chars) - Lead identifier
- `product_code` (string, max 255 chars) - Product code
- `product_type` (string, max 255 chars) - Product type (e.g., PRIVATE_CAR, TWO_WHEELER)
- `quote_type` (string, max 255 chars) - Quote type (e.g., THIRD_PARTY, COMPREHENSIVE)

---

### 2. POST /motor/proposals
**Description:** Create a proposal from a quote

**Mandatory Fields (will throw error if missing):**
- `product_code` (string, max 255 chars) - Product code
- `proposer` (object) - Proposer information (entire object required)
- `vehicle` (object) - Vehicle information (entire object required)

---

### 3. POST /motor/payment_tag
**Description:** Tag payment to a proposal/quote

**Mandatory Fields (will throw error if missing):**
- Either `quote_number` OR `proposal_number` (string, max 255 chars) - At least one is required
- `payment` (object) - Payment information (entire object required)

---

### 4. POST /motor/lead_transfer
**Description:** Receive lead transfer from Bima Sugam (callback request)

**Mandatory Fields (will throw error if missing):**
- None explicitly validated in controller (relies on model validation only)

---

## Health Insurance APIs

### 5. POST /health/get_quotes
**Description:** Get health insurance quotes with premium calculation

**Mandatory Fields (will throw error if missing):**
- None explicitly validated in controller (all validations are commented out)
- Note: Controller checks for null request but does not validate individual fields

---

### 6. POST /health/lead
**Description:** Upsert health lead to health_leads table

**Mandatory Fields (will throw error if missing):**
- `lead_id` (string) - Lead identifier

---

### 7. POST /health/redirection
**Description:** Health redirection endpoint for journey redirection

**Mandatory Fields (will throw error if missing):**
- None (accepts raw JSON payload without validation)

---

### 8. POST /health/get-request-payload
**Description:** Get request payload by proposal number

**Mandatory Fields (will throw error if missing):**
- `proposal_number` (string) - Proposal number

---

## Application Status & Tracking APIs

### 9. POST /application_status
**Description:** Get application status by lead ID and Bima Pehchan ID

**Mandatory Fields (will throw error if missing):**
- `lead_id` (string) - Lead identifier
- `bima_pehchan_id` (string) - Bima Pehchan ID
- `uuid` (string, max 255 chars) - UUID

---

## One-View Policy APIs

### 10. POST /policy/list
**Description:** List all policies for a customer

**Mandatory Fields (will throw error if missing):**
- Either `policy_number` OR `proposer.bima_pehchan_id` - At least one is required

---

### 11. POST /policy/details
**Description:** Get detailed policy information

**Mandatory Fields (will throw error if missing):**
- Either `policy_number` OR `proposer.bima_pehchan_id` - At least one is required
- Either `consent_token` OR `consent_given` (boolean) - Consent is required

---

### 12. POST /policy/document
**Description:** Download policy document (Certificate of Insurance)

**Mandatory Fields (will throw error if missing):**
- `policy_number` (string) - Policy number
- `document_type` (string) - Must be "CERTIFICATE_OF_INSURANCE"

---

## Document Upload API

### 13. POST /document_upload
**Description:** Upload document to Azure Blob Storage

**Mandatory Fields (will throw error if missing):**
- `reference_type` (string) - Reference type (e.g., QUOTE, PROPOSAL, POLICY)
- `reference_number` (string) - Reference number
- `document_key` (string) - Document key/type
- `file.file_content` (string, base64) - Base64 encoded file content

---

## Authentication API

### 14. POST /oauth/token
**Description:** Get JWT Access Token (OAuth2 Client Credentials)

**Mandatory Fields (will throw error if missing):**
- `grant_type` (string) - Must be "client_credentials"
- `client_id` (string) - Client ID
- `client_secret` (string) - Client secret

**Content-Type:** application/x-www-form-urlencoded

---

## Redirection API

### 15. POST /redirection
**Description:** Handle encrypted relay data for redirection flow

**Mandatory Fields (will throw error if missing):**
- `encrypted_data` (string, form-data) - AES-256-GCM encrypted relay payload
- `key_id` (string, form-data) - Key identifier

---

## Lead Management APIs

### 16. PUT /motor/status
**Description:** Update lead status

**Mandatory Fields (will throw error if missing):**
- None explicitly validated in controller

---

### 17. GET /motor/details/{leadNumber}
**Description:** Get lead details by lead number

**Mandatory Fields (will throw error if missing):**
- `leadNumber` (path parameter, string) - Lead number

---

### 18. GET /motor/list
**Description:** Get all leads with optional filtering

**Mandatory Fields (will throw error if missing):**
- None (all parameters are optional)

---

## RTI Client Creation API

### 19. GET /api/RtiClientCreation/{policyNumber}
**Description:** Get RTI request JSON for a policy

**Mandatory Fields (will throw error if missing):**
- `policyNumber` (path parameter, string) - Policy number

---

### 20. POST /api/RtiClientCreation/submit-by-policy/{policyNumber}
**Description:** Submit RTI request to external API

**Mandatory Fields (will throw error if missing):**
- `policyNumber` (path parameter, string) - Policy number

---

## Elasticsearch Search API

### 21. POST /es/search
**Description:** Search Elasticsearch by IBB code

**Mandatory Fields (will throw error if missing):**
- `code` (string) - IBB code to search

---

## Test/Utility APIs

### 22. POST /test/encrypt
**Description:** Encrypt any JSON payload for testing

**Mandatory Fields (will throw error if missing):**
- Raw JSON body (any valid JSON)

---

### 23. POST /test/decrypt
**Description:** Decrypt an encrypted response

**Mandatory Fields (will throw error if missing):**
- `encrypted_data` (string) - Encrypted data
- `key_id` (string) - Key identifier

---

### 24. GET /test/health
**Description:** Health check endpoint

**Mandatory Fields (will throw error if missing):**
- None

---

## Notes

### Common Validation Rules:
1. **Date Formats:**
   - `YYYY-MM-DD` for dates (e.g., 2024-01-15)
   - `YYYY-MM-DDTHH:mm:ss+ZZ:ZZ` for timestamps (e.g., 2024-01-15T10:30:00+05:30)
   - `YYYY-MM` for month-year (e.g., 2024-01)

2. **String Length:**
   - Most string fields have a maximum length of 255 characters
   - Check specific field constraints in the API documentation

3. **Decimal Precision:**
   - Premium amounts: Up to 16 total digits with 4 decimal places
   - Format: `^\d{1,20}(\.\d{0,4})?$`

4. **Phone Number:**
   - Must be 10 digits
   - Must start with 5, 6, 7, 8, or 9
   - Format: `^[5-9]\d{9}$`

5. **Email:**
   - Standard email format validation
   - Format: `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`

6. **Pincode:**
   - Must be exactly 6 digits
   - Format: `^\d{6}$`

7. **Gender:**
   - Valid values: MALE, FEMALE, OTHER (case-sensitive)

### Response Format:
All APIs return responses in the following format:
```json
{
  "uuid": "optional-uuid",
  "status_code": "200",
  "error": {
    "code": "",
    "message": ""
  },
  "data": {
    // Response data
  }
}
```

### Error Codes:
- `400` - Bad Request (validation errors, missing fields)
- `401` - Unauthorized (authentication failed)
- `403` - Forbidden (consent required)
- `404` - Not Found (resource not found)
- `500` - Internal Server Error

---

## Summary

This document lists ONLY the fields that are explicitly validated in the controller code and will return a 400 Bad Request error if missing. Many APIs have additional fields marked with `[Required]` attribute in the model classes, but if the controller doesn't explicitly check for them, they are not included here.

**Key Findings:**
- Most APIs rely on ASP.NET Core model validation (`[Required]` attributes)
- Only a few controllers have explicit validation checks that return BadRequest
- Health APIs have most validations commented out
- Many "required" fields in models don't actually throw errors if missing

**Generated on:** 2026-04-15
**API Version:** 1.0
**Total APIs Documented:** 24
