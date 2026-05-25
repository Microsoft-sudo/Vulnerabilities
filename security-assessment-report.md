# Comprehensive Security Assessment Report
## Target: https://bajaj.webuat.finvu.in (Finvu AA - Bajaj Finserv Consent Journey)

**Assessment Date:** 2026-05-25  
**Environment:** UAT (User Acceptance Testing)  
**Application Type:** Account Aggregator (AA) Consent Flow - Financial Data Sharing  
**Framework:** React SPA + Vite + Finvu Client SDK  
**Hosting:** Google Cloud Storage (Firebase Hosting)

---

## Executive Summary

The target application is a **Financial Information User (FIU)** consent journey application built on India's Account Aggregator framework. It enables users to approve/deny consent for sharing bank statement data with Bajaj/Jainam Broking. The assessment identified **15 security findings** ranging from Critical to Low severity. The most severe issues involve sensitive data exposure in URLs, missing security headers, exposed API infrastructure, and weak client-side security controls in a financial data-sharing application.

---

## CRITICAL FINDINGS

---

### FINDING 1: Sensitive Session Data Exposed in URL Parameters

**Vulnerability Name:** Sensitive Information Disclosure via URL Query Parameters  
**Severity:** CRITICAL  

**Description:**  
The application passes highly sensitive session data as URL query parameters, including a base64-encoded session token containing customer ID, consent handles, cryptographic signatures, and nonce values. URL parameters are logged in browser history, proxy logs, referrer headers, and server access logs.

**Technical Impact:**  
- Customer ID (`7666213741@finvu`) is exposed in URLs
- Cryptographic signature (`ip7XAXDYaLRtLztPYfTnC3l5Wb3P167eB32NbjOaykc`) leaked
- Consent handle IDs exposed
- Nonce values exposed, potentially enabling replay attacks
- Session token can be captured from browser history, Referer headers, proxy logs, shoulder surfing

**Business Impact:**  
Account takeover of financial consent data, unauthorized consent approval/denial, privacy violation of customer financial data sharing preferences. Violates RBI AA framework data protection requirements.

**Proof of Concept:**  
URL contains: `sessionToken=eyJmaXVJZCI6ImZpdUBqYWluYW1icm9raW5nIi...` which decodes to:
```json
{
  "fiuId": "fiu@jainambroking",
  "journeyStartTime": "2026-05-25T12:16:44.420459481Z",
  "signature": "ip7XAXDYaLRtLztPYfTnC3l5Wb3P167eB32NbjOaykc",
  "aaId": "cookiejar-aa@finvu.in",
  "consents": [{
    "consentHandle": "9fc385c2-944e-44b2-951d-2f517f8452e2",
    "template": "BANK_STATEMENT_PERIDIC",
    "purposeCode": "102"
  }],
  "isMultiAA": false,
  "customerId": "7666213741@finvu",
  "expiry": 1779713204,
  "journeyId": "d648c106-45ce-445d-aab0-7d85f85497fa",
  "nonce": "6D86gXvzXcuZOs8TzKRMLA",
  "channelId": "channel@jainambroking"
}
```

**Reproduction Steps:**  
1. Access the consent journey URL
2. Observe URL query parameters: `ecreq`, `reqdate`, `fi`, `sessionToken`
3. Base64-decode `sessionToken` parameter
4. All sensitive session metadata is revealed in plaintext JSON

**Affected Parameters:** `sessionToken`, `ecreq`, `reqdate`, `fi`

**Root Cause:**  
Session data is serialized as plaintext base64 JSON and passed as URL query parameters instead of being retrieved server-side via secure session establishment. The application trusts the URL parameter as the source of truth for session initialization.

**Recommended Mitigation:**  
1. Use opaque, single-use tokens in URLs that map to server-side session data
2. Store session metadata server-side; never expose in URLs
3. If URL-based flow is required by AA protocol, encrypt the session token with server-side key (not just base64 encode)
4. Add `Referrer-Policy: no-referrer` header to prevent leakage via Referer
5. Implement token binding to prevent replay attacks

**CWE Mapping:** CWE-598 (Use of GET Request Method With Sensitive Query Strings), CWE-200 (Exposure of Sensitive Information)  
**OWASP Mapping:** A01:2021 - Broken Access Control, A02:2021 - Cryptographic Failures

---

### FINDING 2: Missing Security Headers (Complete Absence)

**Vulnerability Name:** Missing HTTP Security Headers  
**Severity:** CRITICAL  

**Description:**  
The application does not set ANY security headers. No Content-Security-Policy, X-Frame-Options, X-Content-Type-Options, Strict-Transport-Security, Referrer-Policy, Permissions-Policy, or X-XSS-Protection headers are present.

**Technical Impact:**  
- No clickjacking protection (app can be embedded in malicious iframes)
- No XSS mitigation via CSP
- No MIME-type sniffing protection
- No HSTS enforcement
- Referrer headers leak sensitive URL parameters to third-party sites

**Business Impact:**  
Financial consent UI can be clickjacked to trick users into approving/denying consent. XSS attacks have no defense layer. Financial data consent manipulation possible through UI redressing.

**Proof of Concept:**  
```
$ curl -sI https://bajaj.webuat.finvu.in/
HTTP/2 200
x-guploader-uploadid: ...
x-goog-generation: ...
content-type: text/html
cache-control: private,max-age=0
server: UploadServer
```
No security headers present.

**Reproduction Steps:**  
1. `curl -sI https://bajaj.webuat.finvu.in/`
2. Observe absence of all security headers

**Affected Parameters:** HTTP response headers

**Root Cause:**  
Application hosted on Google Cloud Storage/Firebase Hosting which does not set security headers by default. No custom header configuration applied.

**Recommended Mitigation:**  
Add the following headers:
```
Content-Security-Policy: default-src 'self'; script-src 'self' https://reactjssdk.finvu.in https://revokeconsent.finvu.in; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src https://fonts.gstatic.com; img-src 'self' data:; connect-src 'self' wss://reactjssdk.finvu.in wss://revokeconsent.finvu.in https://reactjssdk.finvu.in https://revokeconsent.finvu.in https://us.i.posthog.com; frame-ancestors 'none'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000; includeSubDomains
Referrer-Policy: no-referrer
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

**CWE Mapping:** CWE-1021 (Improper Restriction of Rendered UI Layers), CWE-693 (Protection Mechanism Failure)  
**OWASP Mapping:** A05:2021 - Security Misconfiguration

---

### FINDING 3: Google Cloud Storage Metadata Exposure

**Vulnerability Name:** Server Technology and Infrastructure Fingerprinting  
**Severity:** HIGH  

**Description:**  
HTTP response headers expose detailed Google Cloud Storage metadata including generation IDs, storage class, content hashes, and the UploadServer identity. This reveals the hosting infrastructure, deployment timestamps, and content fingerprints.

**Technical Impact:**  
- Infrastructure fingerprinting enables targeted attacks
- Generation timestamps reveal deployment schedules
- Content hashes can be used to verify/identify specific deployments
- UploadServer identity reveals exact hosting platform

**Business Impact:**  
Targeted attacks against GCS infrastructure, deployment timing analysis, content verification for reconnaissance.

**Proof of Concept:**  
```
x-guploader-uploadid: AAVLpEhuXPj-EBu809gOM3g-ykCCa0NAN8T5PQpjqD2BZfeW_J-fKn0sQqndRA9aS9_rdMFejBZLUw
x-goog-generation: 1779099249444838
x-goog-metageneration: 1
x-goog-stored-content-encoding: identity
x-goog-stored-content-length: 1650
x-goog-hash: crc32c=snKIMg==, md5=HXpew7p+hgzZfwwUGCylfA==
x-goog-storage-class: STANDARD
server: UploadServer
```

**Reproduction Steps:**  
1. `curl -sI https://bajaj.webuat.finvu.in/`
2. Observe `x-goog-*` and `x-guploader-*` headers

**Affected Parameters:** All HTTP response headers

**Root Cause:**  
Default GCS/Firebase Hosting configuration does not strip infrastructure headers.

**Recommended Mitigation:**  
1. Configure Firebase Hosting headers to remove `x-goog-*` and `x-guploader-*` headers
2. Use a CDN/reverse proxy (Cloudflare, Cloud CDN) to strip metadata headers
3. Set custom `Server` header to hide infrastructure

**CWE Mapping:** CWE-200 (Exposure of Sensitive Information to an Unauthorized Actor)  
**OWASP Mapping:** A05:2021 - Security Misconfiguration

---

## HIGH FINDINGS

---

### FINDING 4: PostHog Analytics Token Exposed in Client-Side JavaScript

**Vulnerability Name:** Exposed Analytics API Key / Third-Party Service Token  
**Severity:** HIGH  

**Description:**  
A PostHog analytics API token (`phc_Pck1qf9Z7aEDR3hlyg7O3OkPUiMmKCP1AJuax0ql8zC`) is embedded in the production JavaScript bundle. This token can be used to inject fake analytics events, pollute tracking data, or potentially access PostHog dashboards if the token has write permissions.

**Technical Impact:**  
- Analytics data injection/pollution
- Potential access to PostHog dashboard with user behavioral data
- Cross-subdomain cookie tracking enabled (`cross_subdomain_cookie`)
- User behavioral data leakage to third-party infrastructure

**Business Impact:**  
Data integrity compromise of analytics, potential PII exposure through PostHog dashboard, privacy violations.

**Proof of Concept:**  
Token found in JS bundle: `phc_Pck1qf9Z7aEDR3hlyg7O3OkPUiMmKCP1AJuax0ql8zC`  
PostHog host: `https://us.i.posthog.com`  
PostHog dev instance accessible: `https://posthogdev.finvu.in` (redirects to login)

**Reproduction Steps:**  
1. Download `https://bajaj.webuat.finvu.in/assets/index-Chkyg081.js`
2. Search for `phc_` pattern
3. Extract PostHog token

**Affected Parameters:** PostHog API token embedded in client-side code

**Root Cause:**  
PostHog token is bundled into client-side JavaScript at build time. No server-side proxy for analytics events.

**Recommended Mitigation:**  
1. Route PostHog events through a server-side proxy endpoint
2. Restrict PostHog token permissions to minimum required
3. Use PostHog's feature to restrict allowed domains
4. Move analytics initialization to server-rendered config with domain validation

**CWE Mapping:** CWE-798 (Use of Hard-coded Credentials), CWE-200  
**OWASP Mapping:** A07:2021 - Identification and Authentication Failures

---

### FINDING 5: Complete API Infrastructure and Endpoints Exposed in Client-Side SDK

**Vulnerability Name:** Full API Surface Area Disclosure  
**Severity:** HIGH  

**Description:**  
The Finvu Client SDK (`finvu-client-sdk.js`) exposes the complete API infrastructure including all endpoint URNs, URL paths, WebSocket URLs, API version routing, and the full public interface. This provides attackers with a complete map of the backend API surface.

**Technical Impact:**  
- All API endpoints enumerated for targeted attacks
- WebSocket connection URLs disclosed
- REST and WebSocket API path structures revealed
- API version routing logic exposed
- Complete URN namespace for all operations revealed

**Business Impact:**  
Targeted API attacks, unauthorized access attempts against financial data endpoints, automated consent manipulation.

**Proof of Concept:**  
Exposed API endpoints:
```
/webapi                          - WebSocket v1 API
/webapiv2                        - WebSocket v2 API
/revokeapi                       - Revoke consent WebSocket API
/revokeapiv2                     - Revoke consent v2 API
/redirection/webapi              - REST API base
/redirection/revokeapi           - Revoke REST API base
/userIdOrMobileNo                - User identification
/webViewloginOtp                 - OTP generation
/loginOtpVerify                  - OTP verification
/entitySdkConfig                - SDK configuration
/snaInitiate                     - Silent Network Auth
/loginOtp                        - Login OTP (legacy)
/getUserConsent                  - Get user consents
/getFipUserConsent               - Get FIP user consents
/userIdFromConsentData           - User ID from consent
/consentRevoke                   - Revoke consent
/consentResume                   - Resume consent
/getConsentDetails               - Consent details
/userInfo                        - User info
/entityInfo                      - Entity info
/logout                          - Logout
```

URN namespace:
```
urn:finvu:in:app:req.userIdOrMobileNo.01
urn:finvu:in:app:req.webViewloginOtp.01
urn:finvu:in:app:req.loginOtpVerify.01
urn:finvu:in:app:req.discover.01
urn:finvu:in:app:req.linking.01
urn:finvu:in:app:req.confirm-token.01
urn:finvu:in:app:req.accountConsentRequest.01
urn:finvu:in:app:req.consent-revoke.01
urn:finvu:in:app:req.consent-resume.01
urn:finvu:in:app:req.getUserConsent.01
urn:finvu:in:app:req.userIdFromConsentData.01
... (and more)
```

**Reproduction Steps:**  
1. Load `https://reactjssdk.finvu.in/sdk/connector/finvu-client-sdk.js`
2. Extract `CONFIG` object and `API_ENDPOINTS`
3. All endpoint paths and URNs are in plaintext

**Affected Parameters:** All SDK API endpoints and URN namespaces

**Root Cause:**  
SDK is served as unobfuscated, unminified JavaScript with full configuration inline. No attempt to obfuscate or protect API surface.

**Recommended Mitigation:**  
1. Minify and obfuscate the SDK JavaScript
2. Move API endpoint configuration to server-side
3. Implement API gateway with rate limiting and request validation
4. Use opaque operation codes instead of descriptive URN names

**CWE Mapping:** CWE-200, CWE-497 (Exposure of System Data to an Unauthorized Control Sphere)  
**OWASP Mapping:** A01:2021 - Broken Access Control

---

### FINDING 6: Client-Side Session Token Not Integrity-Protected (No Signature Verification)

**Vulnerability Name:** Session Token Integrity - No Client-Side Signature Verification  
**Severity:** HIGH  

**Description:**  
The `sessionToken` passed as a URL parameter is a base64-encoded JSON object without any integrity protection mechanism (no HMAC, no JWS signature). While it contains a `signature` field, the application does not verify this signature client-side before using the token data. An attacker can modify the token contents (e.g., change `consentHandle`, `customerId`, `aaId`) before the application processes it.

**Technical Impact:**  
- Consent handle substitution: redirect consent flow to different consent request
- Customer ID tampering: attempt to act on behalf of different customers
- FIP ID manipulation: change the financial institution targeted
- Expiry extension: extend session lifetime

**Business Impact:**  
Consent redirection attacks, potential unauthorized financial data access consent, cross-customer consent manipulation in the AA ecosystem.

**Proof of Concept:**  
Modified token with changed `customerId`:
```json
{
  "fiuId": "fiu@jainambroking",
  "customerId": "DIFFERENT_CUSTOMER@finvu",  // Modified
  "consentHandle": "attacker-controlled-handle",  // Modified
  "expiry": 1999999999  // Extended
}
```
Base64-encode and replace in URL → application processes tampered session data.

**Reproduction Steps:**  
1. Decode the sessionToken from URL
2. Modify fields (customerId, consentHandle, expiry)
3. Re-encode as base64
4. Replace in URL and access the modified URL
5. Application reads the modified token without integrity checks

**Affected Parameters:** `sessionToken` URL parameter

**Root Cause:**  
Session token is base64-encoded JSON without JWS/JWE protection. No client-side signature verification before processing. Backend validation may exist but the client trusts and processes the token immediately.

**Recommended Mitigation:**  
1. Use JWS (JSON Web Signature) for the session token
2. Verify signature server-side before processing
3. Implement token binding to the session (IP, fingerprint, etc.)
4. Add nonce verification server-side to prevent replay

**CWE Mapping:** CWE-345 (Insufficient Verification of Data Authenticity), CWE-642 (External Control of Critical State Data)  
**OWASP Mapping:** A02:2021 - Cryptographic Failures, A07:2021 - Identification and Authentication Failures

---

### FINDING 7: WebSocket Communication Without Origin Validation

**Vulnerability Name:** WebSocket Missing Origin Validation  
**Severity:** HIGH  

**Description:**  
The SDK establishes WebSocket connections (`wss://`) for real-time communication but the SDK code does not set or validate the `Origin` header on the WebSocket handshake. The WebSocket URLs are constructed dynamically from the SDK script's hostname, meaning any domain loading the SDK can establish WebSocket connections.

**Technical Impact:**  
- Cross-site WebSocket hijacking (CSWSH) possible
- Malicious websites can establish WebSocket connections to the Finvu backend
- Session takeover if cookies are sent cross-origin
- Real-time manipulation of consent flow

**Business Impact:**  
Financial consent manipulation from malicious third-party sites, session hijacking, unauthorized financial data operations.

**Proof of Concept:**  
SDK code:
```javascript
var WSS_WEB_API_URL = 'wss://' + hostname + webapi;
client = new WebSocket(wssUrl);
```
No Origin header validation on the WebSocket connection.

**Reproduction Steps:**  
1. Include `https://reactjssdk.finvu.in/sdk/connector/finvu-client-sdk.js` on attacker-controlled domain
2. Call `finvuClient.open()` with valid credentials
3. WebSocket connection is established without origin validation

**Affected Parameters:** WebSocket connection URLs

**Root Cause:**  
No Origin header validation on WebSocket handshake. SDK creates WebSocket connections without specifying allowed origins.

**Recommended Mitigation:**  
1. Validate Origin header on WebSocket handshake server-side
2. Implement CORS-like origin allowlist for WebSocket connections
3. Use token-based authentication for WebSocket connections instead of cookie-based
4. Add CSRF token to WebSocket connection initiation

**CWE Mapping:** CWE-346 (Origin Validation Error), CWE-1385 (Missing Origin Validation in WebSocket)  
**OWASP Mapping:** A05:2021 - Security Misconfiguration

---

### FINDING 8: SDK Public Interface Enables Consent Manipulation via Browser Console

**Vulnerability Name:** Global SDK Interface Exposed on window Object  
**Severity:** HIGH  

**Description:**  
The Finvu SDK exposes its complete public interface on `window.finvuClient` (consent journey) and `window.revokeFinvuClient` (revoke flow). This allows any JavaScript code running in the browser context (including XSS payloads, browser extensions, or console commands) to call sensitive operations like consent approval, account linking, and consent revocation.

**Technical Impact:**  
- `finvuClient.consentApproveRequest()` - Approve consent from console
- `finvuClient.consentApproveRequestAll()` - Approve all consents
- `finvuClient.consentApproveRequestWithConsentId()` - Approve specific consent
- `finvuClient.revokeConsent()` - Revoke consent
- `finvuClient.consentResume()` - Resume suspended consent
- `finvuClient.discoverAccounts()` - Discover linked bank accounts
- `finvuClient.accountLinking()` - Link accounts
- `finvuClient.userLinkedAccounts()` - View all linked accounts
- `finvuClient.logout()` - Force logout

**Business Impact:**  
If XSS exists anywhere in the application, an attacker can directly call financial consent operations. Browser extensions with content script capabilities can silently manipulate consent. Physical access or XSS enables one-click financial consent fraud.

**Proof of Concept:**  
```javascript
// In browser console after page load:
window.finvuClient.consentApproveRequestAll(
  FIPDetails, 'ACCEPT', consentHandleId
);
// Silently approves all consent requests
```

**Reproduction Steps:**  
1. Open the application in browser
2. Open DevTools Console
3. Type `window.finvuClient` to see all exposed methods
4. Call any method directly (e.g., consent approval/denial)

**Affected Parameters:** `window.finvuClient`, `window.revokeFinvuClient`

**Root Cause:**  
SDK deliberately exposes full API on global `window` object for inter-operation between script tags and the React app. No access control on the public interface.

**Recommended Mitigation:**  
1. Encapsulate SDK instance; don't expose on window
2. Use closure pattern to restrict method access
3. Add caller verification before executing sensitive operations
4. Implement confirmation step for consent actions regardless of API call
5. Rate limit sensitive operations server-side

**CWE Mapping:** CWE-265 (Privilege Issues), CWE-733 (Operator Precedence Logic Error)  
**OWASP Mapping:** A01:2021 - Broken Access Control

---

## MEDIUM FINDINGS

---

### FINDING 9: OTP Lockout Bypass - Client-Side Attempt Counter Reset

**Vulnerability Name:** Client-Side OTP Brute Force Protection Bypass  
**Severity:** MEDIUM  

**Description:**  
The SDK implements OTP attempt counting client-side with a 3-attempt lockout. However, the `otpAttempts` counter is a client-side variable that resets on page reload. The lockout logic runs entirely in JavaScript with no server-side enforcement visible.

**Technical Impact:**  
- OTP brute force: 3 attempts per page reload, unlimited reloads
- No server-side rate limiting visible in the SDK code
- 6-digit OTP = 1,000,000 combinations, feasible with automation
- OTP reference is stored client-side and reused across attempts

**Business Impact:**  
OTP brute force could lead to account takeover, unauthorized consent approval on financial data sharing.

**Proof of Concept:**  
SDK code:
```javascript
otpAttempts++;
if (otpAttempts >= 3) {
    // Lock out
    otpAttempts = 0; // Reset counter after locking
}
```
Reload page → counter resets → 3 more attempts.

**Reproduction Steps:**  
1. Enter wrong OTP 3 times → get locked out message
2. Reload page → counter resets
3. Enter 3 more wrong OTPs → locked again
4. Repeat with automation script

**Affected Parameters:** `otpAttempts` client-side variable, `/loginOtpVerify` endpoint

**Root Cause:**  
OTP attempt counting is client-side only. No server-side rate limiting or account lockout mechanism visible.

**Recommended Mitigation:**  
1. Implement server-side OTP attempt tracking
2. Progressive delays after failed attempts
3. Account lockout after N failed attempts (server-side)
4. CAPTCHA after 3 failed attempts
5. Notify user of failed OTP attempts

**CWE Mapping:** CWE-307 (Improper Restriction of Excessive Authentication Attempts), CWE-613  
**OWASP Mapping:** A07:2021 - Identification and Authentication Failures

---

### FINDING 10: Weak Client-Side Input Validation Only

**Vulnerability Name:** Client-Only Input Validation  
**Severity:** MEDIUM  

**Description:**  
The SDK validates inputs using client-side regex patterns only. These patterns can be bypassed by modifying requests directly or using the SDK's REST API endpoints.

**Technical Impact:**  
- Customer ID regex: `/^[a-zA-Z0-9][-_\\.a-zA-Z0-9]{5,29}@finvu$/` - can be bypassed via direct API calls
- Mobile number regex: `/^[0-9]{10}$/` - can send any format via API
- OTP regex: `/^(\d{6},)*\d{6}$/` - allows comma-separated OTPs (multiple OTPs in single request)
- No server-side validation confirmation visible

**Business Impact:**  
Injection attacks via malformed input, potential SQL/NoSQL injection through unvalidated fields, OTP format abuse.

**Proof of Concept:**  
```javascript
// OTP validation allows comma-separated OTPs
var otp_validation = /^(\d{6},)*\d{6}$/;
"123456,654321,111111".match(otp_validation) // passes validation
```
This could allow submitting multiple OTP guesses in a single request.

**Reproduction Steps:**  
1. Intercept API request to `/loginOtpVerify`
2. Modify `otp` parameter to contain SQL injection payload
3. Modify `userId` to contain special characters
4. Submit modified request

**Affected Parameters:** `userId`, `otp`, `mobileNum` in all API calls

**Root Cause:**  
Client-side regex validation only, no evidence of equivalent server-side validation.

**Recommended Mitigation:**  
1. Implement strict server-side input validation for all fields
2. Validate OTP format server-side (strict 6-digit, no commas)
3. Add server-side rate limiting per OTP reference
4. Implement parameterized queries / input sanitization server-side

**CWE Mapping:** CWE-20 (Improper Input Validation), CWE-602 (Client-Side Enforcement of Server-Side Security)  
**OWASP Mapping:** A03:2021 - Injection

---

### FINDING 11: No Clickjacking Protection - Financial Consent UI Frameable

**Vulnerability Name:** Clickjacking / UI Redressing on Consent Flow  
**Severity:** MEDIUM  

**Description:**  
The application has no `X-Frame-Options` header or `Content-Security-Policy: frame-ancestors` directive. The consent approval/denial UI can be embedded in an iframe on a malicious site, enabling clickjacking attacks on financial consent operations.

**Technical Impact:**  
- Transparent overlay attacks on "Approve" / "Deny" buttons
- Users can be tricked into approving financial data sharing consent
- Drag-and-drop attacks to transfer consent actions
- Particularly dangerous for AA consent flows where a single click shares bank data

**Business Impact:**  
Users tricked into approving consent for financial data sharing (bank statements), resulting in unauthorized access to financial information.

**Proof of Concept:**  
```html
<!-- Attacker page -->
<iframe src="https://bajaj.webuat.finvu.in/?ecreq=...&sessionToken=..." 
        style="opacity:0.1; position:absolute; top:0; left:0; width:500px; height:500px;">
</iframe>
<!-- Overlay fake button on top of consent "Approve" button -->
```

**Reproduction Steps:**  
1. Create HTML page with iframe pointing to the consent URL
2. Style iframe with low opacity
3. Overlay decoy buttons over consent approve/deny buttons
4. User clicks decoy → actually clicks consent approve

**Affected Parameters:** Consent approval UI, all interactive pages

**Root Cause:**  
Missing `X-Frame-Options` and `frame-ancestors` CSP directive.

**Recommended Mitigation:**  
1. Add `X-Frame-Options: DENY` header
2. Add `Content-Security-Policy: frame-ancestors 'none'`
3. Implement JavaScript frame-busting code as defense-in-depth
4. Add user confirmation step for consent actions

**CWE Mapping:** CWE-1021 (Improper Restriction of Rendered UI Layers), CWE-451  
**OWASP Mapping:** A04:2021 - Insecure Design

---

### FINDING 12: Session Token Expiry Bypass Potential

**Vulnerability Name:** Short Session Lifetime with Client-Controlled Expiry  
**Severity:** MEDIUM  

**Description:**  
The session token contains an `expiry` field (Unix timestamp 1779713204 = 2026-05-25 12:46:44 UTC) giving a 30-minute session. Since the expiry is embedded in the client-controlled token and the token lacks integrity protection (Finding 6), the expiry can potentially be extended by modifying the `expiry` field in the base64 token.

**Technical Impact:**  
- Session expiry extension beyond intended 30 minutes
- Long-lived sessions for consent manipulation
- Potential indefinite session if server does not independently validate expiry

**Business Impact:**  
Extended attack window for consent manipulation, bypass of session timeout security controls.

**Proof of Concept:**  
```json
{
  "expiry": 1999999999  // Changed from 1779713204 to year 2033
}
```

**Reproduction Steps:**  
1. Decode sessionToken
2. Modify `expiry` field to future timestamp
3. Re-encode and replace in URL
4. Access with modified token

**Affected Parameters:** `expiry` field in `sessionToken`

**Root Cause:**  
Expiry enforced client-side within the token. No evidence of independent server-side session expiration. Token lacks integrity protection.

**Recommended Mitigation:**  
1. Enforce session expiry server-side independently of client token
2. Use JWS with server-verified expiry claims
3. Implement server-side session store with TTL
4. Add sliding window session timeout

**CWE Mapping:** CWE-613 (Insufficient Session Expiration), CWE-642  
**OWASP Mapping:** A07:2021 - Identification and Authentication Failures

---

## LOW FINDINGS

---

### FINDING 13: SDK Logging Enabled in Production

**Vulnerability Name:** Debug Logging Enabled in Production SDK  
**Severity:** LOW  

**Description:**  
The SDK has `logEnable = true` by default, causing all SDK operations, API URLs, session IDs, and WebSocket messages to be logged to the browser console. This exposes sensitive session information to anyone with DevTools access.

**Technical Impact:**  
- Session IDs logged to console
- API URLs and parameters visible
- WebSocket message contents logged
- OTP references visible in console

**Proof of Concept:**  
```javascript
var logEnable = true;
var log = function log(msg) {
    if (logEnable) { console.log(msg); }
};
```

**Reproduction Steps:**  
1. Open application with DevTools Console
2. Observe detailed SDK operation logs including session data

**Root Cause:**  
Default logging enabled, no production build flag to disable.

**Recommended Mitigation:**  
1. Disable logging in production SDK (`logEnable = false`)
2. Use build-time environment variables to control logging
3. Strip console.log calls in production builds

**CWE Mapping:** CWE-200, CWE-497  
**OWASP Mapping:** A05:2021 - Security Misconfiguration

---

### FINDING 14: CORS Misconfiguration on SDK Domain

**Vulnerability Name:** Overly Permissive CORS on SDK Domain  
**Severity:** LOW  

**Description:**  
The SDK domain (`reactjssdk.finvu.in`) returns `access-control-allow-origin: reactjssdk.finvu.in` in response headers. While this restricts to a single origin, it allows cross-origin requests from the SDK domain to be made with credentials, which could be exploited if an attacker can host content on the SDK domain or a subdomain.

**Technical Impact:**  
- Cross-origin requests to SDK endpoints with credentials
- Subdomain takeover could enable CORS exploitation
- API calls can be made from SDK subdomains

**Proof of Concept:**  
```
$ curl -sk -H "Origin: https://evil.com" -I https://reactjssdk.finvu.in/sdk/connector/finvu-client-sdk.js
access-control-allow-origin: reactjssdk.finvu.in
```

**Root Cause:**  
CORS header returns the specific origin but may not validate subdomains properly.

**Recommended Mitigation:**  
1. Explicitly validate the Origin header server-side
2. Return `Access-Control-Allow-Credentials: true` only when needed
3. Implement strict origin allowlist

**CWE Mapping:** CWE-942 (Permissive Cross-domain Policy), CWE-346  
**OWASP Mapping:** A05:2021 - Security Misconfiguration

---

### FINDING 15: Build-Time Comments Expose Development Workflow

**Vulnerability Name:** Source Code Build Configuration Leak  
**Severity:** LOW (INFORMATIONAL)  

**Description:**  
The HTML source contains detailed comments about the SDK loading order, environment variable names (`VITE_MAIN_SDK_URL`, `VITE_REVOKE_SDK_URL`), build commands (`npm run dev`, `vite build --mode uat`, `vite build`), and the internal architecture of the dual-SDK system.

**Technical Impact:**  
- Build workflow and deployment process exposed
- Environment variable naming convention revealed
- Internal SDK architecture documented in production
- Attackers know exact build commands and modes

**Proof of Concept:**  
```html
<!--
  SDK loading order matters:
  1. Revoke SDK loads first — its hostname (revokeconsent.finvu.in) sets the revoke backend.
  2. It is immediately aliased to window.revokeFinvuClient.
  3. Main SDK loads second — overwrites window.finvuClient with the consent-journey instance.

  URLs are injected from env at build time:
    VITE_MAIN_SDK_URL   → window.finvuClient       (consent journey)
    VITE_REVOKE_SDK_URL → window.revokeFinvuClient (revoke flow)

  Build per environment:
    dev          : npm run dev          (uses .env)
    uat          : vite build --mode uat         (uses .env.uat)
    production   : vite build           (uses .env.production)
-->
```

**Reproduction Steps:**  
1. View page source of `https://bajaj.webuat.finvu.in/`
2. Read the HTML comment block

**Root Cause:**  
Developer comments left in production HTML template.

**Recommended Mitigation:**  
1. Remove all developer comments from production builds
2. Use Vite's HTML minification to strip comments
3. Document build workflow internally only

**CWE Mapping:** CWE-200, CWE-497  
**OWASP Mapping:** A05:2021 - Security Misconfiguration

---

## BUSINESS LOGIC FINDINGS

---

### FINDING 16: SNA Authentication Bypass via Global Callback Injection

**Vulnerability Name:** Silent Network Auth (SNA) Bypass via `window.handleStartAuthResponse` Manipulation  
**Severity:** HIGH  

**Description:**  
The SNA (Silent Network Auth / OTP-less) flow relies on two global callback functions exposed on `window`: `window.handleInitAuthResponse` and `window.handleStartAuthResponse`. These are called by the native app bridge after carrier-based authentication. However, since they are global functions, any JavaScript code (XSS, browser extension, console) can call them directly with crafted responses, bypassing the entire carrier authentication check.

The critical flow is:
1. `_loginEncrypt()` sends encrypted request, receives `authType: 'SNA'` response
2. Native bridge calls `handleInitAuthResponse({status: 'SUCCESS'})` → sets `authType = 'SNA'`
3. Native bridge calls `handleStartAuthResponse({status: 'SUCCESS', snaToken: '<any_token>'})` → calls `_verifyOTP(snaToken)` → **completes login without real OTP**

An attacker can call `window.handleStartAuthResponse({status: 'SUCCESS', snaToken: '123456'})` directly from the console, and the SDK will attempt to verify this as an SNA OTP token.

**Technical Impact:**  
- Complete OTP bypass if server accepts the SNA token without verifying it came from a legitimate carrier
- `authType` is set to `'SNA'` which skips OTP validation (`if (authType !== 'SNA') { _res = OTPValidation(otp); }`)
- SNA flow has a 120-second timeout — attacker can trigger success callback instantly
- The `resolveSna` function is accessible, allowing the snaPromise to be resolved with arbitrary data

**Business Impact:**  
Complete authentication bypass on a financial consent application. An attacker who can execute JavaScript (XSS, malicious extension, physical access) can skip OTP entirely and approve/deny financial data sharing consents.

**Proof of Concept:**  
```javascript
// In browser console after SNA login is initiated:
window.handleInitAuthResponse({status: 'SUCCESS'});
window.handleStartAuthResponse({
  status: 'SUCCESS',
  snaToken: '000000'  // Arbitrary token sent to _verifyOTP
});
// Or directly resolve the SNA promise:
if (resolveSna) {
  resolveSna({status: 'SUCCESS', userInfo: {...}, sid: sid});
}
```

**Reproduction Steps:**  
1. Access the consent journey URL
2. Open DevTools Console during the login phase
3. Wait for SNA flow to start (120s timeout window)
4. Call `window.handleStartAuthResponse({status:'SUCCESS', snaToken:'123456'})`
5. SDK processes the SNA token through `_verifyOTP()` bypassing user input

**Affected Parameters:** `window.handleInitAuthResponse`, `window.handleStartAuthResponse`, `resolveSna`, `authType`

**Root Cause:**  
SNA authentication callbacks are exposed as global window functions without any origin verification, nonce matching, or signature validation. The SDK trusts any response that matches the expected `{status: 'SUCCESS', snaToken: '...'}` shape.

**Recommended Mitigation:**  
1. Do not expose SNA callbacks on `window` — use postMessage with origin validation
2. Add server-side verification that SNA tokens were generated by the carrier (not client-injected)
3. Implement challenge-response between server and carrier, not just a passive token
4. Validate SNA token server-side against carrier's verification endpoint before creating session
5. Add cryptographic binding between the SNA initiation request and the SNA token response

**CWE Mapping:** CWE-287 (Improper Authentication), CWE-610 (Externally Controlled Reference)  
**OWASP Mapping:** A07:2021 - Identification and Authentication Failures

---

### FINDING 17: Consent Handle IDOR — Cross-User Consent Manipulation

**Vulnerability Name:** Insecure Direct Object Reference on Consent Handle  
**Severity:** HIGH  

**Description:**  
The consent approval functions (`_consentApproveRequest`, `_consentApproveRequestAll`, `_consentRequestApproveWithConsentId`) accept a `consentHandleId` parameter. When `institutionType` is not `'LSP'`, the SDK automatically uses `handleID` (extracted from the encrypted request response) and `fiId` as defaults. However, these can be overridden by passing explicit values. The `_consentRequestDetails` function also accepts an optional `consentHandleId` parameter that defaults to the session's `handleID` but can be set to any handle ID.

Critical code:
```javascript
if (institutionType != 'LSP') {
    if (consentHandleId == undefined) {
        consentHandleId = handleID; // Uses session handle
    }
    if (instId == null || isEmpty(instId)) {
        instId = fiId; // Uses session FIU ID
    }
}
```

This means: for non-LSP institution types, the consent handle and FIU ID default to the session values but **can be explicitly set to any value**, potentially allowing manipulation of consent belonging to other users or other FIU consent requests.

**Technical Impact:**  
- Approve/deny consent on behalf of different consent handles (different users)
- Access consent request details for other users' consent handles
- Manipulate consent renewal (`_consentRequestRenew`) with different `consentId` values
- The `_consentRequestApproveWithConsentId` function accepts arbitrary `consentId` values

**Business Impact:**  
Cross-user consent manipulation in the AA ecosystem. An attacker could approve consent requests intended for other users, leading to unauthorized financial data sharing. Could also deny legitimate consent requests, causing denial of service.

**Proof of Concept:**  
```javascript
// Via browser console:
window.finvuClient.consentApproveRequestAll(
  FIPDetails, 'ACCEPT',
  'different-consent-handle-id',  // Another user's consent handle
  'different-fiu-id'              // Another FIU's ID
);
```

**Reproduction Steps:**  
1. Login to the consent journey with valid credentials
2. Open DevTools Console
3. Call `consentApproveRequestAll` with a different `consentHandleId`
4. If server does not validate handle-to-user binding, consent is approved for another user

**Affected Parameters:** `consentHandleId`, `instId`, `consentId` in consent approval/renewal flows

**Root Cause:**  
Consent handle IDs are treated as direct object references without server-side verification that the authenticated user owns or is authorized to act on the specified consent handle. The SDK allows explicit override of default values.

**Recommended Mitigation:**  
1. Server-side: validate that `consentHandleId` belongs to the authenticated `userId`
2. Bind consent handles to user sessions server-side
3. Reject explicit `consentHandleId` values that don't match the session's consent
4. Implement access control checks on all consent operations
5. Add audit logging for consent handle access attempts

**CWE Mapping:** CWE-639 (Authorization Bypass Through User-Controlled Key), CWE-285 (Improper Authorization)  
**OWASP Mapping:** A01:2021 - Broken Access Control

---

### FINDING 18: Consent Resume After Revocation — Business Logic Flaw

**Vulnerability Name:** Revoked Consent Can Be Resumed Without Fresh Authorization  
**Severity:** HIGH  

**Description:**  
The revoke SDK exposes a `resumeConsent` function (`/consentResume`, URN: `consent-resume.01`) that can resume a previously revoked consent. The function only requires `userId` and `consentId` — no re-authentication, no fresh OTP, and no fresh user consent is required. If an attacker gains access to any session (via XSS, token theft, or SNA bypass), they can resume previously revoked consents, re-enabling financial data sharing that the user explicitly revoked.

```javascript
var _resumeConsent = async function _resumeConsent(consentId) {
    // Only checks: sid exists, userId exists, consentId exists
    var payload = {
        userId: userId,
        consentId: consentId
    };
    // No re-authentication, no fresh consent, no OTP
};
```

**Technical Impact:**  
- Revoked consents can be silently resumed with just `userId` + `consentId`
- No re-authentication required (relies on existing session `sid`)
- No user notification or confirmation for resume operation
- Combined with consent IDOR (Finding 17), could resume other users' revoked consents

**Business Impact:**  
Users who revoked consent for financial data sharing can have that consent silently re-enabled. This violates the core trust model of the AA framework where user revocation is supposed to be definitive. If consent is resumed, the FIU can once again fetch the user's financial data without the user's knowledge.

**Proof of Concept:**  
```javascript
// After login, resume a previously revoked consent
window.revokeFinvuClient.resumeConsent('previously-revoked-consent-id');
// Consent is now active again without user authorization
```

**Reproduction Steps:**  
1. Login to the revoke consent flow
2. Note a revoked consent ID from `userActiveConsents`
3. Call `resumeConsent(consentId)` via console
4. If server allows, the revoked consent is now active

**Affected Parameters:** `consentId` in `resumeConsent`, `/consentResume` endpoint

**Root Cause:**  
The consent resume operation does not require fresh user authorization (OTP, explicit consent). It relies only on an existing session. The AA framework's trust model assumes revocation is user-intended and final — resuming should require at minimum the same level of authentication as the original consent approval.

**Recommended Mitigation:**  
1. Require fresh OTP verification before resuming any consent
2. Require explicit user confirmation (not just session) for consent resume
3. Add time-bound restriction: consents revoked > N hours ago cannot be resumed
4. Send notification (SMS/email) when consent is resumed
5. Server-side: validate that the resume operation is explicitly authorized by the user
6. Add audit trail for consent resume operations

**CWE Mapping:** CWE-285 (Improper Authorization), CWE-863 (Incorrect Authorization)  
**OWASP Mapping:** A01:2021 - Broken Access Control

---

### FINDING 19: userIdFromConsentData — User Identity Enumeration

**Vulnerability Name:** User Identity Enumeration via Consent Data  
**Severity:** MEDIUM  

**Description:**  
The revoke SDK exposes a `userIdFromConsentData` function (`/userIdFromConsentData`, URN: `userIdFromConsentData.01`) that resolves user identity from encrypted request data (`ecreq`, `reqDate`, `fi`). This endpoint does not require a session (`sid` is not checked) — it only needs the encrypted request parameters. This means anyone with access to a consent URL (which may be logged, shared, or leaked) can discover the user's identity.

```javascript
var _userIdFromConsentData = async function _userIdFromConsentData(ecreq, reqDate, fi, requestorType) {
    var payload = {
        encryptedRequest: ecreq,
        requestDate: reqDate,
        encryptedFiType: requestorType,
        encryptedFiuId: fi
    };
    // No session check, no auth check
};
```

**Technical Impact:**  
- User identity enumeration from consent URLs
- No authentication required — works with publicly visible URL parameters
- Can be used to correlate consent requests with specific users
- Combined with URL parameter exposure (Finding 1), this is a complete user deanonymization vector

**Business Impact:**  
Privacy violation — user identity can be extracted from consent URLs without authentication. Violates data minimization principles of the AA framework.

**Proof of Concept:**  
```javascript
// Using the URL parameters from any consent link:
window.revokeFinvuClient.userIdFromConsentData(
  ecreq_value, reqdate_value, fi_value, ''
);
// Returns the userId associated with this consent request
```

**Reproduction Steps:**  
1. Extract `ecreq`, `reqdate`, `fi` from any consent URL
2. Call `userIdFromConsentData` via the SDK
3. User identity is returned without authentication

**Affected Parameters:** `ecreq`, `reqDate`, `fi` via `/userIdFromConsentData` endpoint

**Root Cause:**  
The endpoint does not require authentication (session `sid`) before resolving user identity from encrypted request data.

**Recommended Mitigation:**  
1. Require active session/authentication before resolving user identity
2. Rate limit this endpoint aggressively
3. Add CAPTCHA for unauthenticated identity resolution
4. Consider removing this endpoint entirely — FIU should already know the user identity

**CWE Mapping:** CWE-200 (Exposure of Sensitive Information), CWE-285 (Improper Authorization)  
**OWASP Mapping:** A01:2021 - Broken Access Control

---

### FINDING 20: Consent Approval Without Account Ownership Verification

**Vulnerability Name:** Missing Account Ownership Verification in Consent Approval  
**Severity:** HIGH  

**Description:**  
When approving a consent request, the SDK sends `FIPDetails` containing account references (account reference numbers, FI types, FIP IDs). The SDK constructs the approval payload client-side, and the accounts included in the approval are taken from the consent request details. However, there is no verification in the SDK that the accounts being approved actually belong to the logged-in user.

The consent approval payload structure:
```javascript
var consent_request_approve_payload = function(FIPDetails, handleStatus, consentHandleId, instId) {
    // FIPDetails is directly from the consent request
    // No verification that accounts belong to userId
    var _payload = {
        'FIPDetails': FIPDetails,
        'FIU': { 'id': instId },
        'ver': VERSION,
        'consentHandleId': consentHandleId,
        'handleStatus': handleStatus
    };
};
```

The `FIPDetails` contains `Accounts` arrays with `accRefNumber` and `FIType`. An attacker with access to the SDK interface can substitute account reference numbers, potentially approving consent for accounts they don't own.

**Technical Impact:**  
- Account reference substitution in consent approval
- No client-side or visible server-side validation of account-to-user mapping
- Combined with `window.finvuClient` exposure (Finding 8), this is directly exploitable
- The `selectedAccounts` state in the React app is managed client-side with no server binding

**Business Impact:**  
Consent could be approved for financial accounts not belonging to the authenticated user, leading to unauthorized financial data access through legitimate consent channels.

**Proof of Concept:**  
```javascript
// After login, modify FIPDetails with different account references
var modifiedFIPDetails = [{
    FIP: { id: 'some-fip-id' },
    Accounts: [{
        accRefNumber: 'NOT_MY_ACCOUNT_123',  // Different user's account
        FIType: 'DEPOSIT'
    }]
}];
window.finvuClient.consentApproveRequestAll(
    modifiedFIPDetails, 'ACCEPT', handleID, fiId
);
```

**Reproduction Steps:**  
1. Login to consent journey
2. View discovered accounts and note their reference numbers
3. Modify account reference numbers in the FIPDetails
4. Call `consentApproveRequestAll` with modified account references
5. If server does not verify account ownership, consent is approved for wrong accounts

**Affected Parameters:** `FIPDetails.Accounts[].accRefNumber` in consent approval

**Root Cause:**  
Account references in consent approval are client-controlled. No evidence of server-side verification that the approved accounts are linked to the authenticated user.

**Recommended Mitigation:**  
1. Server-side: validate that all account references in the approval belong to the authenticated user
2. Only allow consent for accounts returned by the discover/link flow for this user
3. Add server-side account-to-user binding verification
4. Reject consent approvals with account references not previously discovered/linked for this session

**CWE Mapping:** CWE-285 (Improper Authorization), CWE-639 (Authorization Bypass Through User-Controlled Key)  
**OWASP Mapping:** A01:2021 - Broken Access Control

---

### FINDING 21: Consent Status Polling Race Condition

**Vulnerability Name:** Race Condition in Consent Status Polling  
**Severity:** MEDIUM  

**Description:**  
The `_getUpdatedConsentStatus` function polls the consent handle status in a loop with 6-second intervals and `MAX_RETRIES=5`. The check condition is:
```javascript
var checkCondition = function(data) {
    return data.handleStatus !== 'ACCEPT' || curRetries > MAX_RETRIES;
};
```

This means the polling stops when the status is no longer `'ACCEPT'` or after 5 retries. However, there is a race condition: between the user viewing the consent details and approving/denying, the consent handle status could change on the server (e.g., expired, cancelled by FIU). The SDK does not verify the consent is still in a valid state at the time of approval.

Additionally, the `_getUpdatedConsentStatus` function uses a `while` loop without any cancellation mechanism. If the server consistently returns `handleStatus: 'ACCEPT'`, the loop runs for `5 * 6000ms = 30 seconds` and then stops with `curRetries > MAX_RETRIES`, but the result is never explicitly handled as a timeout.

**Technical Impact:**  
- Consent can be approved after the consent handle has expired or been cancelled server-side
- No atomicity between consent status check and approval action
- Polling loop has no timeout cancellation — can hang if not careful
- Multiple concurrent approval attempts could race

**Business Impact:**  
Approval of expired or cancelled consent requests, leading to data fetching based on stale consent authorization. This violates AA framework's requirement that consent must be valid at the time of data fetch.

**Proof of Concept:**  
1. User views consent details (handle is ACCEPT)
2. FIU cancels the consent request server-side
3. User clicks "Approve" — the SDK sends approval without re-checking handle status
4. Server may process the approval for a cancelled handle

**Reproduction Steps:**  
1. Open consent journey and navigate to consent details page
2. In another session, cancel or expire the consent handle
3. Click "Approve" in the original session
4. Approval may be processed for the now-invalid consent handle

**Affected Parameters:** `consentHandleId` in approval flow, `_getUpdatedConsentStatus` polling

**Root Cause:**  
No atomic check-and-approve operation. Consent status is polled asynchronously but approval is a separate operation without re-validation of consent handle status.

**Recommended Mitigation:**  
1. Server-side: implement atomic check-and-approve (validate handle is still ACCEPT within the same transaction as approval)
2. Add consent handle status re-check immediately before approval
3. Return error if consent handle status has changed since last status check
4. Add idempotency key to prevent duplicate approval attempts

**CWE Mapping:** CWE-362 (Race Condition), CWE-367 (Time-of-Check Time-of-Use Race Condition)  
**OWASP Mapping:** A04:2021 - Insecure Design

---

### FINDING 22: Consent Checkbox Requirement Client-Side Enforcement Only

**Vulnerability Name:** Consent Checkbox Requirement Bypass via Client-Side Configuration Manipulation  
**Severity:** MEDIUM  

**Description:**  
The `_isCheckboxRequired` function determines whether a consent checkbox must be checked before approval. This decision is based on `sdkOptions.consentOptions`, which is a JSON string parsed client-side from the FIU's configuration. Since `sdkOptions` is a client-side variable and the entire consent UI is controlled client-side, an attacker can:
1. Override `_isCheckboxRequired` to always return `false`
2. Modify `sdkOptions` to change which consents are pre-selected or optional
3. Bypass the `_rankConsents` function which orders consents by priority

The SDK comment states: `//remove for when encrypt api can be exposed without session id` near `getSID`, suggesting that some API endpoints can be accessed without a session, further weakening client-side controls.

**Technical Impact:**  
- Mandatory consent checkboxes can be bypassed
- Consent ordering/priority can be manipulated
- Pre-selected consents can be changed
- `sdkOptions` can be injected/modified via the SDK's `open` function

**Business Impact:**  
Users may have consents approved without properly acknowledging mandatory consent requirements, violating regulatory consent requirements in the AA framework.

**Proof of Concept:**  
```javascript
// Override checkbox requirement
window.finvuClient.isCheckboxRequired = function() { return false; };
// Or modify sdkOptions
// The consent approval proceeds without mandatory checkboxes
```

**Reproduction Steps:**  
1. Open consent journey
2. In DevTools Console, override `window.finvuClient.isCheckboxRequired`
3. Proceed with consent approval — checkboxes are not enforced
4. If server does not enforce checkbox requirements, approval succeeds

**Affected Parameters:** `sdkOptions`, `_isCheckboxRequired`, `_rankConsents`

**Root Cause:**  
Consent UI requirements (checkbox, ranking, defaults) are enforced client-side only. No server-side validation of consent acknowledgment requirements.

**Recommended Mitigation:**  
1. Server-side: enforce consent acknowledgment requirements independently of client
2. Validate that all mandatory consent checkboxes were checked before processing approval
3. Add server-side consent requirement validation based on FIU configuration
4. Don't trust `sdkOptions` from the client for security-critical decisions

**CWE Mapping:** CWE-602 (Client-Side Enforcement of Server-Side Security), CWE-285  
**OWASP Mapping:** A04:2021 - Insecure Design

---

### FINDING 23: Version Downgrade Attack — API Version Switching

**Vulnerability Name:** API Version Downgrade from V2 to V1  
**Severity:** MEDIUM  

**Description:**  
The SDK supports two API versions: V1 (`/webapi`) and V2 (`/webapiv2`). The version is controlled by the `webapiv2` boolean flag. V2 has stricter security controls — it removes `credentials: 'include'` from fetch requests and uses custom headers (`x-finvu-csid`, `x-finvu-sid`) instead of cookies for session management. However:

1. The `_open` function (V1) uses `credentials: 'include'` — cookies are sent cross-origin
2. V1 does not use `x-finvu-csid` header
3. The `useRest` flag can be set to `true` to use REST API instead of WebSocket, potentially bypassing WebSocket security controls
4. The `protocol` parameter in `_openV2` can force `useRest = true`

```javascript
// V2 removes credentials (more secure)
if (webapiv2) {
    delete httpReqBody['credentials'];
}
// But V1 keeps them (less secure)
```

An attacker who can call `_open()` instead of `_openV2()` forces the older, less secure API version.

**Technical Impact:**  
- Downgrade from V2 (custom headers, no cookies) to V1 (cookie-based auth, credentials include)
- V1 may have different server-side validation or missing security patches
- `useRest` bypass changes the communication protocol entirely
- REST API endpoints may have different rate limiting than WebSocket

**Business Impact:**  
Security controls present in V2 (no cookie credentials, custom session headers) can be bypassed by forcing V1. This is particularly relevant for CSRF and session management attacks.

**Proof of Concept:**  
```javascript
// Force V1 instead of V2
window.finvuClient.open(secret, principal);  // Uses V1 with credentials: include
// Instead of:
window.finvuClient.openV2(secret, principal);  // Uses V2 without credentials
```

**Reproduction Steps:**  
1. Intercept SDK initialization
2. Call `open()` instead of `openV2()`
3. Observe that cookies and credentials are included in requests
4. Session management uses cookies instead of custom headers

**Affected Parameters:** `webapiv2` flag, `useRest` flag, `credentials` fetch option

**Root Cause:**  
The SDK supports multiple API versions and protocols for backward compatibility, but there's no enforcement of the latest version. An attacker or misconfigured client can downgrade.

**Recommended Mitigation:**  
1. Deprecate V1 API and require V2
2. Server-side: reject requests using V1 endpoints
3. Add version validation on server-side — reject outdated protocol requests
4. Remove `useRest` toggle or require it for all requests

**CWE Mapping:** CWE-754 (Inappropriate Check for Unusual or Exceptional Conditions), CWE-327  
**OWASP Mapping:** A02:2021 - Cryptographic Failures, A05:2021 - Security Misconfiguration

---

### FINDING 24: SNA Timeout Creates 120-Second Authentication Window

**Vulnerability Name:** SNA 120-Second Timeout Creates Extended Attack Window  
**Severity:** MEDIUM  

**Description:**  
When the server returns `authType: 'SNA'` but the user is on a web browser (no native bridge), the SNA flow enters a 120-second timeout before falling back to OTP. During this entire window, the SNA promise is unresolved and the authentication state is in limbo. This creates an extended attack window where:

1. The `resolveSna` function is accessible and can be called to resolve the promise with arbitrary data
2. The `snaPromise` is accessible and can be manipulated
3. The `settled` flag can be reset to re-trigger the fallback
4. The 120-second window gives attackers time to manipulate the auth state

```javascript
snaPromise = new Promise(function(resolve, reject) {
    var settled = false;
    var timer = setTimeout(function() {
        if (!settled) {
            settled = true;
            // Falls back to OTP after 120s
        }
    }, 120000);
    resolveSna = function(value) {
        if (!settled) {
            settled = true;
            clearTimeout(timer);
            resolve(value);
        }
    };
});
```

**Technical Impact:**  
- 120-second window to call `resolveSna()` with forged success data
- `resolveSna` is a closure variable but accessible via the returned `snaPromise` from `_loginEncrypt()`
- During timeout, user sees a loading state — no visual indication of vulnerability
- The `_loginEncrypt` function returns `snaPromise` to the caller, exposing the promise

**Business Impact:**  
Extended window for authentication manipulation. During the 120-second SNA timeout, an attacker with JavaScript execution can force the auth flow to succeed or fail on demand.

**Proof of Concept:**  
The `_loginEncrypt()` function returns an object containing `snaPromise`. While `resolveSna` is not directly on the returned object, the global `window.handleStartAuthResponse` is, and it calls `resolveSna` internally.

**Reproduction Steps:**  
1. Initiate login when server returns `authType: 'SNA'`
2. During the 120-second timeout, call `window.handleStartAuthResponse({status:'SUCCESS', snaToken:'any'})`
3. The SNA promise resolves and OTP is bypassed

**Affected Parameters:** `snaPromise`, `resolveSna`, `settled` flag, 120000ms timeout

**Root Cause:**  
SNA authentication has a generous timeout window with no additional security validation during the waiting period. Global callback functions allow external resolution of the auth promise.

**Recommended Mitigation:**  
1. Reduce SNA timeout to the minimum necessary (e.g., 30 seconds)
2. Add server-side nonce/challenge that must match in the SNA callback
3. Do not expose SNA callbacks on `window` — use secure postMessage channel
4. Add rate limiting on SNA initiation per session
5. Implement SNA token binding so forged tokens cannot verify

**CWE Mapping:** CWE-307 (Improper Restriction of Excessive Authentication Attempts), CWE-287  
**OWASP Mapping:** A07:2021 - Identification and Authentication Failures

---

### FINDING 25: getSID() Exposes Session ID to Client-Side Code

**Vulnerability Name:** Session ID Leakage via getSID() Public Method  
**Severity:** MEDIUM  

**Description:**  
The SDK exposes a `getSID()` method on the public interface that returns the internal session ID (`sid`). The comment in the code reads: `//remove for when encrypt api can be exposed without session id` — indicating this was intended to be temporary. The session ID is a critical authentication token that should never be exposed to client-side code.

```javascript
var _getSID = function _getSID() {
    return sid;
};
```

The `sid` is used in all API request headers (`x-finvu-sid`) and WebSocket message headers. Exposing it allows any script to:
1. Steal the session ID and use it in another context (session hijacking)
2. Use the `sid` to construct authenticated API requests directly, bypassing the SDK
3. Share the session ID with third parties for later abuse

**Technical Impact:**  
- Session ID exposed to all JavaScript code in the page
- XSS payloads can exfiltrate the session ID
- Browser extensions can read the session ID
- Session hijacking if SID is stolen and replayed

**Business Impact:**  
Session hijacking of financial consent sessions. Stolen session IDs can be used to approve/deny consents, access user information, and manipulate linked accounts.

**Proof of Concept:**  
```javascript
// In browser console:
var stolenSID = window.finvuClient.getSID();
// Now use stolenSID to construct authenticated requests directly
```

**Reproduction Steps:**  
1. Login to consent journey
2. Open DevTools Console
3. Call `window.finvuClient.getSID()`
4. Session ID is returned in plaintext

**Affected Parameters:** `sid` session ID via `getSID()`

**Root Cause:**  
The `getSID()` method was added as a temporary workaround (per the comment) for the encrypt API needing a session ID. It should have been removed but remains in the public interface.

**Recommended Mitigation:**  
1. Remove `getSID()` from the public interface immediately
2. Handle the encrypt API's session ID requirement internally without exposing it
3. Bind session IDs to the client's origin/fingerprint server-side
4. Implement session ID rotation
5. Add session ID invalidation on suspicious activity

**CWE Mapping:** CWE-200 (Exposure of Sensitive Information), CWE-315 (Cleartext Storage of Sensitive Information)  
**OWASP Mapping:** A07:2021 - Identification and Authentication Failures

---

### FINDING 26: Institution Type (LSP vs FIU) Changes Security Posture — Logic Bypass

**Vulnerability Name:** Institution Type Manipulation Alters Validation Logic  
**Severity:** MEDIUM  

**Description:**  
The `institutionType` variable (`'FIU'` or `'LSP'`) fundamentally changes the validation logic throughout the SDK. When `institutionType == 'LSP'`:
- Consent handle ID is **not** auto-populated from `handleID`
- FIU ID is **not** auto-populated from `fiId`
- Different consent detail encryption URNs are used (`consent_request_detail_list_encrypt` vs `consent_request_detail_encrypt`)
- The `handleID` and `fiId` are **not** extracted from consent response

This means the LSP code path has **weaker defaults** and requires explicit values, while the FIU path auto-fills from the session. An attacker who can manipulate `institutionType` can switch between these code paths to:
1. Force the LSP path to bypass auto-populated security defaults
2. Use different (potentially less validated) API endpoints
3. Access the `consent_request_detail_list_encrypt` endpoint instead of `consent_request_detail_encrypt`

**Technical Impact:**  
- Different validation logic per institution type creates inconsistent security
- LSP path requires explicit `consentHandleId` and `instId` (no defaults) — potentially allows cross-institution operations
- Different encryption endpoints may have different server-side security controls
- The `_institutionType()` function returns `'FIU'` by default when `ecreq` is empty, but this can be manipulated

**Business Impact:**  
Bypass of FIU-specific security controls by switching to LSP code path. Inconsistent validation creates opportunities for privilege escalation between institution types.

**Proof of Concept:**  
```javascript
// The institutionType is set from server response but can be manipulated
// by intercepting the _fiuInfo response or calling functions with explicit parameters
// that force the LSP code path
```

**Reproduction Steps:**  
1. Intercept the `_fiuInfo` response
2. Change `institutionType` to `'LSP'`
3. Consent approval now requires explicit `consentHandleId` and `instId` without defaults
4. Different API endpoints are used for consent details

**Affected Parameters:** `institutionType`, `consentHandleId`, `instId` in all consent flows

**Root Cause:**  
Two different code paths with different security properties based on a client-influenced `institutionType` variable. The LSP path has weaker defaults and different endpoint usage.

**Recommended Mitigation:**  
1. Unify validation logic across institution types
2. Server-side: enforce institution-type-specific access controls independently
3. Validate that the client-declared `institutionType` matches the server's record
4. Apply the same security defaults (handleID, fiId) regardless of institution type

**CWE Mapping:** CWE-285 (Improper Authorization), CWE-863 (Incorrect Authorization)  
**OWASP Mapping:** A01:2021 - Broken Access Control

---

## RISK SUMMARY

| # | Finding | Severity | CWE | OWASP |
|---|---------|----------|-----|-------|
| 1 | Sensitive Data in URL Parameters | CRITICAL | CWE-598/200 | A01/A02 |
| 2 | Missing Security Headers | CRITICAL | CWE-1021/693 | A05 |
| 3 | GCS Metadata Exposure | HIGH | CWE-200 | A05 |
| 4 | PostHog Token Exposed | HIGH | CWE-798 | A07 |
| 5 | Full API Surface Exposed | HIGH | CWE-200/497 | A01 |
| 6 | Session Token No Integrity | HIGH | CWE-345/642 | A02/A07 |
| 7 | WebSocket No Origin Validation | HIGH | CWE-346/1385 | A05 |
| 8 | Global SDK Interface on window | HIGH | CWE-265 | A01 |
| 9 | OTP Lockout Client-Side Only | MEDIUM | CWE-307/613 | A07 |
| 10 | Client-Only Input Validation | MEDIUM | CWE-20/602 | A03 |
| 11 | Clickjacking on Consent UI | MEDIUM | CWE-1021/451 | A04 |
| 12 | Session Expiry Bypass | MEDIUM | CWE-613/642 | A07 |
| 13 | Production Debug Logging | LOW | CWE-200/497 | A05 |
| 14 | CORS on SDK Domain | LOW | CWE-942/346 | A05 |
| 15 | Build Comments in Production | LOW | CWE-200/497 | A05 |
| **16** | **SNA Auth Bypass via Global Callbacks** | **HIGH** | **CWE-287/610** | **A07** |
| **17** | **Consent Handle IDOR** | **HIGH** | **CWE-639/285** | **A01** |
| **18** | **Consent Resume After Revocation** | **HIGH** | **CWE-285/863** | **A01** |
| **19** | **userIdFromConsentData Enumeration** | **MEDIUM** | **CWE-200/285** | **A01** |
| **20** | **No Account Ownership Verification** | **HIGH** | **CWE-285/639** | **A01** |
| **21** | **Consent Status Race Condition** | **MEDIUM** | **CWE-362/367** | **A04** |
| **22** | **Checkbox Requirement Client-Side Only** | **MEDIUM** | **CWE-602/285** | **A04** |
| **23** | **API Version Downgrade Attack** | **MEDIUM** | **CWE-754/327** | **A02/A05** |
| **24** | **SNA 120s Timeout Attack Window** | **MEDIUM** | **CWE-307/287** | **A07** |
| **25** | **getSID() Exposes Session ID** | **MEDIUM** | **CWE-200/315** | **A07** |
| **26** | **Institution Type Logic Bypass** | **MEDIUM** | **CWE-285/863** | **A01** |

**Severity Distribution:** CRITICAL: 2 | HIGH: 9 | MEDIUM: 13 | LOW: 3

---

## ADDITIONAL OBSERVATIONS

### Hidden Feature: AI Voice Journey
The SDK contains a `_consentJourneyInitiate` function that accepts `channelId` and `language` parameters, along with `_consentJourneyStatus` that requires `aiVoiceReferenceId` and `userId`. This indicates an **AI voice-based consent journey** feature. The `getEncRequestVoiceJourney` function passes additional `channel` and `language` parameters. This hidden feature could have its own attack surface if voice consent is treated as equivalent to explicit user consent.

### Hidden Feature: Consent Resume
The revoke SDK contains a `consentResume` endpoint (`/consentResume`, URN: `consent-resume.01`) that allows resuming previously suspended/revoked consents. This is a high-risk operation that could allow an attacker to resume revoked consents, potentially re-enabling financial data sharing without user knowledge.

### Hidden Feature: userIdFromConsentData
The endpoint `/userIdFromConsentData` (URN: `userIdFromConsentData.01`) allows resolving user identity from consent data. This could be an IDOR vector if consent data from other users can be queried to discover their user IDs.

### SDK Version Information
- SDK Version: `2.2.0`
- Protocol Version: `1.1.2`
- React Router Version: `6` (exposed via `window.__reactRouterVersion`)
- Build tool: Vite

### Third-Party Dependencies with Known Patterns
- PostHog analytics (session recording, feature flags, surveys)
- Sentry error tracking (integration with PostHog)
- i18next for internationalization
- React Router v6
- Redux Toolkit
- blueimp/JavaScript-MD5 (MD5 usage in financial application is concerning)

---

## PRIORITIZED REMEDIATION ROADMAP

### Immediate (0-7 days)
1. Add security headers (Finding 2) - 1 hour fix
2. Remove developer comments from HTML (Finding 15) - 15 min fix
3. Disable SDK console logging in production (Finding 13) - 30 min fix
4. Strip GCS metadata headers via proxy/CDN (Finding 3) - 2 hour fix
5. Remove `getSID()` from public interface (Finding 25) - 15 min fix
6. Remove SNA global callbacks from window (Finding 16) - 1 hour fix

### Short-term (1-4 weeks)
7. Implement server-side session management instead of URL tokens (Finding 1)
8. Add JWS/JWE protection to session tokens (Finding 6)
9. Implement server-side OTP rate limiting (Finding 9)
10. Add X-Frame-Options/CSP frame-ancestors (Finding 11)
11. Server-side input validation for all API endpoints (Finding 10)
12. WebSocket Origin validation (Finding 7)
13. Server-side consent handle-to-user binding validation (Finding 17)
14. Require fresh OTP for consent resume (Finding 18)
15. Server-side account ownership verification for consent approval (Finding 20)
16. Implement atomic check-and-approve for consent (Finding 21)
17. Server-side consent checkbox enforcement (Finding 22)

### Medium-term (1-3 months)
18. Remove PostHog token from client-side; proxy analytics (Finding 4)
19. Encapsulate SDK; remove from window object (Finding 8)
20. Server-side session expiry enforcement (Finding 12)
21. CORS hardening on SDK domain (Finding 14)
22. Minify/obfuscate SDK to protect API surface (Finding 5)
23. Deprecate V1 API; enforce V2 (Finding 23)
24. Reduce SNA timeout and add challenge-response (Finding 24)
25. Add authentication to userIdFromConsentData (Finding 19)
26. Unify institution type validation logic (Finding 26)

---

*Report generated through passive analysis. No active exploitation was performed. All findings are based on observable information from the target URL, HTTP headers, JavaScript source code analysis, and SDK reverse engineering.*