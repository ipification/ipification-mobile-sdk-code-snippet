# TS43 CIBA Integration Guide for Android
## Seamless SIM Phone Number Verification via Client-Initiated Backchannel Authentication

---

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Integration Steps](#integration-steps)
5. [API Endpoints](#api-endpoints)

---

## Overview

TS43 (Client-Initiated Backchannel Authentication - CIBA) enables seamless phone number verification using the Android Credential Manager API in conjunction with OpenID4VP (OpenID for Verifiable Presentations) protocol. This implementation provides a passwordless authentication experience without requiring SMS or additional user actions.

### Key Benefits
- ✅ **Seamless UX**: No SMS codes, no manual input
- ✅ **Secure**: Uses device-level credential management
- ✅ **Standards-based**: Built on OpenID4VP and CIBA specifications
- ✅ **Network-agnostic**: Works across cellular carriers
- ✅ **Privacy-focused**: User controls credential sharing

---

## Architecture

TS43 CIBA Flow Diagram:

<img width="1654" height="1795" alt="ts43" src="https://github.com/user-attachments/assets/263e124a-9292-4e6d-89c1-17ed0c964497" />


### Key Stages

**IMPORTANT**: The Android app only calls 2 backend APIs: `/ts43/auth` and `/ts43/token`. The app does NOT call IPification Service directly. All communication with IPification Service is handled by your backend.

1. **App → Backend: CIBA Authentication Request** (`/ts43/auth`): App initiates authentication with phone number  
2. **Backend → IPification Service**: Backend forwards request to IPification CIBA endpoint  
3. **Backend → App: Authentication Response**: Backend returns `auth_req_id` and `digital_request`  
4. **App → Credential Manager: Request Credential**: App passes `digital_request` to Android Credential Manager  
5. **Credential Manager → App: VP Token**: Credential Manager returns `vp_token` to App  
6. **App → Backend: Token Exchange** (`/ts43/token`): App sends `vp_token` and `auth_req_id` to backend  
7. **Backend → IPification Service**: Backend validates `vp_token` and exchanges for tokens  
8. **Backend → App: Token Response**: Backend returns result including `phone_number_verified`
9. **Verification Complete**: User is authenticated with verified phone number  

**Android official doc:** https://developer.android.com/identity/digital-credentials/phone-number-verification

---

## Prerequisites

### Required Dependencies

Add these to your `app/build.gradle`:

```gradle
dependencies {
    // Android Credential Manager
    implementation 'androidx.credentials:credentials:1.6.0-beta01'
    implementation 'androidx.credentials:credentials-play-services-auth:1.6.0-beta01'
    
    // Google Play Services
    implementation 'com.google.android.gms:play-services-auth:21.4.0'
    
    // Networking
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'
    
    // Kotlin Coroutines
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
}
```

### Requirements
- **Android SDK**: Minimum API 31 (Android 12.0)
- **Target SDK**: 33 or higher (recommended)
- **Google Play Services**: Latest version installed on device

---

## Integration Steps

### Step 1: Initialize Credential Manager

```kotlin
import androidx.credentials.CredentialManager
import androidx.credentials.GetCredentialRequest
import androidx.credentials.GetDigitalCredentialOption
import android.content.Context

class TS43AuthManager(private val context: Context) {
    private val credentialManager = CredentialManager.create(context)
    
    // Initialize in your activity or fragment
    companion object {
        fun initialize(context: Context): TS43AuthManager {
            return TS43AuthManager(context.applicationContext)
        }
    }
}
```

### Step 2: Configure Backend Endpoints

**IMPORTANT**: Configure your backend API endpoints, NOT IPification Service endpoints. Your backend will handle all communication with IPification Service.

```kotlin
object TS43Config {
    // Your Backend API URLs
    const val BACKEND_URL_SANDBOX = "https://your-backend-api-url.com"
    const val BACKEND_URL_PRODUCTION = "https://your-prod-backend-api-url.com"
    
    // Set environment
    var BACKEND_URL = BACKEND_URL_SANDBOX // Change to PRODUCTION for live
    
    // Backend API Endpoints (only 2 endpoints needed)
    val TS43_AUTH_ENDPOINT = "$BACKEND_URL/ts43/auth"
    val TOKEN_ENDPOINT = "$BACKEND_URL/ts43/token"
    
    // Your client ID
    var CLIENT_ID = "your_client_id"
}
```

### Step 3: Implement CIBA Authentication Request

**This calls your backend `/ts43auth` endpoint, which then communicates with IPification Service.**

```kotlin
suspend fun initiateTS43Auth(
    phoneNumber: String,
    scope: String = "openid ip:phone",
    clientId: String
): TS43AuthResponse {
    val url = TS43Config.TS43_AUTH_ENDPOINT
    
    // Get carrier information
    val carrierHint = getCarrierHint() // optional
    
    val requestBody = JSONObject().apply {
        put("client_id", clientId)
        put("login_hint", phoneNumber)
        put("carrier_hint", carrierHint) // optional
        put("scope", scope) // "openid ip:phone_verify" or "openid ip:phone"
        put("operation", operation) // "VerifyPhoneNumber" or "GetPhoneNumber"
    }.toString()
    
    return withContext(Dispatchers.IO) {
        val client = OkHttpClient()
        val request = Request.Builder()
            .url(url)
            .post(requestBody.toRequestBody("application/json".toMediaType()))
            .build()
        
        client.newCall(request).execute().use { response ->
            if (!response.isSuccessful) {
                throw IOException("TS43 Auth failed: ${response.code}")
            }
            
            val responseBody = response.body?.string() ?: ""
            parseTS43AuthResponse(responseBody)
        }
    }
}
```

### Step 4: Parse CIBA Authentication Response

**Your backend returns both `auth_req_id` and `digital_request` in a single response.**

```kotlin
data class TS43AuthResponse(
    val authReqId: String,
    val digitalRequest: String,
    val expiresIn: Int,
    val interval: Int
)

private fun parseTS43AuthResponse(jsonResponse: String): TS43AuthResponse {
    val json = JSONObject(jsonResponse)
    return TS43AuthResponse(
        authReqId = json.getString("auth_req_id"),
        digitalRequest = json.getJSONObject("digital_request").toString(),
        expiresIn = json.optInt("expires_in", 120),
        interval = json.optInt("interval", 5)
    )
}
```

### Step 5: Extract Digital Request

```kotlin
fun extractDigitalRequest(ts43Response: TS43AuthResponse): String {
    // The digital_request is already included in the TS43 auth response
    return ts43Response.digitalRequest
}
```

### Step 6: Request Credential via Credential Manager

```kotlin
import android.app.Activity
import androidx.core.content.ContextCompat
import androidx.credentials.CredentialManager
import androidx.credentials.CredentialManagerCallback
import androidx.credentials.GetCredentialRequest
import androidx.credentials.GetCredentialResponse
import androidx.credentials.exceptions.GetCredentialException
import androidx.credentials.provider.CallingAppInfo
import androidx.credentials.GetDigitalCredentialOption
import androidx.credentials.DigitalCredential
import androidx.credentials.ExperimentalDigitalCredentialApi

@OptIn(ExperimentalDigitalCredentialApi::class)
fun requestCredential(
    activity: Activity,
    digitalRequest: String,
    onSuccess: (String) -> Unit,
    onError: (Exception) -> Unit
) {
    val credentialManager = CredentialManager.create(activity)
    val requestJson = """{"requests": [$digitalRequest]}"""

    val digitalCredentialOption = GetDigitalCredentialOption(
        requestJson = requestJson
    )

    val getCredentialRequest = GetCredentialRequest(
        listOf(digitalCredentialOption)
    )

    credentialManager.getCredentialAsync(
        context = activity,
        request = getCredentialRequest,
        cancellationSignal = null,
        executor = ContextCompat.getMainExecutor(activity),
        callback = object : CredentialManagerCallback<GetCredentialResponse, GetCredentialException> {
            override fun onResult(result: GetCredentialResponse) {
                val credential = result.credential

                if (credential is DigitalCredential) {
                    onSuccess(credential.credentialJson)
                } else {
                    onError(IllegalStateException("Unexpected credential type: ${credential.type}"))
                }
            }

            override fun onError(e: GetCredentialException) {
                onError(e)
            }
        }
    )
}

```

### Step 7: Extract VP Token

```kotlin
private fun extractVpToken(credentialJson: String): String? {
    // The credential response has structure:
    // {"protocol":"...","data":{"vp_token":{"ipification.com":["token_value"]}}}
    
    val ipificationPattern = "\"ipification\\.com\"\\s*:\\s*\\[\\s*\"([^\"]+)\"".toRegex()
    return ipificationPattern.find(credentialJson)?.groupValues?.get(1)
}
```

### Step 8: Token Exchange (includes VP Token validation)

**Send both `vp_token` and `auth_req_id` to your backend `/ts43/token` endpoint. Your backend handles validation with IPification Service.**

```kotlin
suspend fun exchangeToken(
    vpToken: String,
    authReqId: String
): TokenResponse {
    val url = TS43Config.TOKEN_ENDPOINT
    
    val requestBody = JSONObject().apply {
        put("vp_token", vpToken)
        put("auth_req_id", authReqId)
    }.toString()
    
    return withContext(Dispatchers.IO) {
        val client = OkHttpClient()
        val request = Request.Builder()
            .url(url)
            .post(requestBody.toRequestBody("application/json".toMediaType()))
            .build()
        
        client.newCall(request).execute().use { response ->
            if (!response.isSuccessful) {
                throw IOException("Token exchange failed: ${response.code}")
            }
            
            val responseBody = response.body?.string() ?: ""
            parseTokenResponse(responseBody)
        }
    }
}

data class TokenResponse(
    val accessToken: String,
    val tokenType: String,
    val expiresIn: Int,
    val refreshToken: String?,
    val idToken: String
)

private fun parseTokenResponse(jsonResponse: String): TokenResponse {
    val json = JSONObject(jsonResponse)
    return TokenResponse(
        accessToken = json.getString("access_token"),
        tokenType = json.getString("token_type"),
        expiresIn = json.getInt("expires_in"),
        refreshToken = json.optString("refresh_token", null),
        idToken = json.getString("id_token")
    )
}
```

### Step 9: Parse Token Response

**Your backend returns the final tokens after validating with IPification Service.**

```kotlin
data class TokenResponse(
    val accessToken: String,
    val tokenType: String,
    val expiresIn: Int,
    val refreshToken: String?,
    val idToken: String
)

private fun parseTokenResponse(jsonResponse: String): TokenResponse {
    val json = JSONObject(jsonResponse)
    return TokenResponse(
        accessToken = json.getString("access_token"),
        tokenType = json.getString("token_type"),
        expiresIn = json.getInt("expires_in"),
        refreshToken = json.optString("refresh_token", null),
        idToken = json.getString("id_token")
    )
}
```

---

## API Endpoints 


### 1. CIBA Authentication Endpoint

**POST** `/ts43/auth`

Your Android app calls this backend endpoint to initiate authentication. Your backend then communicates with IPification Service.

#### Request Headers
```
Content-Type: application/json
```

#### Request Body
```json
{
  "client_id": your_client_id,
  "login_hint": phoneNumber, // for VerifyPhoneNumber
  "carrier_hint": carrier_hint_value,
  "scope": "openid ip:phone_verify",
  "operation": "VerifyPhoneNumber"
}
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `client_id` | string | Yes | Your OAuth client ID |
| `login_hint` | string | Yes | The phone number to verify (E.164 format) for VerifyPhoneNumber, put `anonymous` for GetPhoneNumber |
| `scope` | string | Yes | "openid ip:phone_verify" or "openid ip:phone" |
| `carrier_hint` | string | No | Mobile Country Code + Mobile Network Code (MCCMNC) |
| `operation` | string | Yes | to Verify Input Phone or Get Phone Number |
#### Response (Success - 200 OK)
```json
{
  "auth_req_id": "eyJhbGciOiJIUzI1...",
  "expires_in": 120,
  "interval": 5,
  "digital_request": {
    "protocol": "openid4vp",
    "version": "1.0",
    "request": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjFkOGUzMDE5LWM4YjctNDNkYi1iM2M5LTAzODQ4ZmEyZWM3OCJ9..."
  }
}
```

#### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `auth_req_id` | string | Unique identifier for this auth session |
| `expires_in` | integer | Session expiration time in seconds |
| `interval` | integer | Recommended polling interval (if applicable) |
| `digital_request` | object | Digital credential request for Credential Manager |

#### Error Responses

**400 Bad Request**
```json
{
  "error": "invalid_request",
  "error_description": "Missing required parameter: login_hint"
}
```

**401 Unauthorized**
```json
{
  "error": "invalid_client",
  "error_description": "Client authentication failed"
}
```

**503 Service Unavailable**
```json
{
  "error": "temporarily_unavailable",
  "error_description": "Service temporarily unavailable, please try again"
}
```

---

### 2. Token Exchange Endpoint

**POST** `/ts43/token`

Your Android app calls this backend endpoint to exchange the `vp_token` and `auth_req_id` for final tokens. Your backend validates the `vp_token` with IPification Service and returns the tokens.

#### Request Headers
```
Content-Type: application/json
```

#### Request Body
```json
{
  "vp_token": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9...",
  "auth_req_id": "eyJhbGciOiJIUzI1..."
}
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `vp_token` | string | Yes | VP token from Credential Manager |
| `auth_req_id` | string | Yes | Auth request ID from initial auth call |

#### Response (Success - 200 OK)
```json
{
  "login_hint": "381123456789",
  "phone_number_verified": "true"
}
```

#### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `login_hint` | string | Phone Return by the system |
| `phone_number_verified` | string | Authorization Result |




© 2025 IPification. Implementation code examples are provided as-is for integration purposes.
