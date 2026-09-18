# 📱 Phone Number Retrieval Implementation Guide (Android)

This document explains how to retrieve a user's SIM-based phone number on Android devices using two approaches:

1. **Google Phone Number Hint API** — No permission required
2. **Permission-based fallback** using `TelephonyManager` and `SubscriptionManager`

---

## Overview

| Method | Permissions | Works Without Play Services |
|---------|--------------|-----------------------------|
| **Option 1: Phone Number Hint API** | ❌ No | ❌ No |
| **Option 2: TelephonyManager Fallback** | ✅ Yes | ✅ Yes |

---

## Option 1: Google Phone Number Hint API (No Permissions Required)

<img src="https://github.com/user-attachments/assets/ab967ffb-5fdd-452c-99cc-191a399f9686" alt="Phone Number Hint dialog" width="300"/>

### 🔍 Description

The [Phone Number Hint API](https://developers.google.com/identity/phone-number-hint/android) displays a **secure, system dialog** listing SIM phone numbers for the user to choose from.
It doesn't require any dangerous permissions and is privacy-safe.

---

### Step 1: Add Dependencies

```groovy
// app/build.gradle
implementation 'com.google.android.gms:play-services-auth:21.3.0'

// Optional: for parsing/formatting phone numbers
implementation("com.googlecode.libphonenumber:libphonenumber:8.13.24")
```

---

### Step 2: Create a Request and Launch the Hint Picker

```kotlin
val request = GetPhoneNumberHintIntentRequest.builder().build()
val signInClient = Identity.getSignInClient(this)

signInClient.getPhoneNumberHintIntent(request)
    .addOnSuccessListener { result: PendingIntent ->
        try {
            phoneNumberHintIntentResultLauncher.launch(
                IntentSenderRequest.Builder(result).build()
            )
        } catch (e: Exception) {
            Log.e("PhoneHint", "Launching PendingIntent failed", e)
        }
    }
    .addOnFailureListener { e ->
        Log.e("PhoneHint", "Phone Number Hint failed", e)
    }
```

---

### 🧾 Step 3: Handle the Result

```kotlin
private val phoneNumberHintIntentResultLauncher =
    registerForActivityResult(ActivityResultContracts.StartIntentSenderForResult()) { result ->
        if (result.resultCode == Activity.RESULT_OK && result.data != null) {
            try {
                val phoneNumber = Identity.getSignInClient(requireActivity())
                    .getPhoneNumberFromIntent(result.data)

                val phone = detectCountryAndExtractNationalNumber(phoneNumber)
                binding.phoneCodeEditText.setText(phone.second)

                Log.d("PhoneHint", "Retrieved phone number: $phoneNumber")
            } catch (e: Exception) {
                Log.e("PhoneHint", "Failed to retrieve phone number: ${e.message}")
                Toast.makeText(requireContext(), "Failed to retrieve phone number", Toast.LENGTH_SHORT).show()
            }
        } else {
            Log.w("PhoneHint", "Hint cancelled or no result.")
            Toast.makeText(requireContext(), "Phone number hint cancelled", Toast.LENGTH_SHORT).show()
        }
    }
```

---

### Optional: Normalize with libphonenumber

```kotlin
fun detectCountryAndExtractNationalNumber(phoneNumber: String): Pair<String?, String?> {
    val phoneUtil = PhoneNumberUtil.getInstance()
    return try {
        val parsed = phoneUtil.parse(phoneNumber, null)
        val region = phoneUtil.getRegionCodeForNumber(parsed)
        val national = parsed.nationalNumber.toString()
        Pair(region, national)
    } catch (e: NumberParseException) {
        Log.e("PhoneHint", "Number parse failed: ${e.message}")
        Pair(null, null)
    }
}
```

---

### ✅ Advantages

- Zero runtime permissions required
- Secure and privacy-friendly flow
- Automatically handles multiple SIMs
- Compliant with Android Play policy

---

### ⚠️ Limitations

- Requires Google Play Services
- Returns `null` if unsupported or unavailable or user cancels

---

## Option 2: TelephonyManager Fallback (READ_PHONE_STATE + READ_PHONE_NUMBERS)

### 🔍 Description

When Play Services is unavailable, or the user dismisses the hint dialog, the number can be read directly from the SIM. This requires two runtime permissions and depends on the carrier having written the MSISDN to the SIM — some carriers don't, in which case the result is `null`.

> Requires **minSdk 24** (`TelephonyManager.createForSubscriptionId`, `SubscriptionManager.getDefaultDataSubscriptionId`).

---

### Manifest Permissions

```xml
<!--
  Required to read basic telephony information such as the device's network state
  and active subscriptions.
  This permission is needed to call:
    - TelephonyManager.getLine1Number()                     → retrieves the device's phone number (MSISDN)
    - SubscriptionManager.getActiveSubscriptionInfoList()   → lists SIMs/subscriptions
-->
<uses-permission android:name="android.permission.READ_PHONE_STATE" />

<!--
  Required on Android 13 (API 33 / Tiramisu) and above to access the user's phone number
  via SubscriptionManager.getPhoneNumber(int subId).
  Without this permission, even READ_PHONE_STATE cannot return the MSISDN anymore.
-->
<uses-permission android:name="android.permission.READ_PHONE_NUMBERS" />
```

<img src="https://github.com/user-attachments/assets/5a0f2f7b-7717-455b-856c-63e392e32fa8" alt="Permission dialog 1" width="300"/>
<img src="https://github.com/user-attachments/assets/dfd04470-07e1-484f-9d9e-b49685ea8da4" alt="Permission dialog 2" width="300"/>

---

### Step 1: The Reader (pure logic, no UI)

`SimPhoneNumberReader` resolves the active SIM's subscription id, then reads the number with the best API for the OS version. It never throws and is safe to call from the main thread.

```kotlin
import android.Manifest
import android.content.Context
import android.content.pm.PackageManager
import android.os.Build
import android.telephony.SubscriptionManager
import android.telephony.TelephonyManager
import android.util.Log
import androidx.core.content.ContextCompat
import com.ipification.mobile.sdk.android.utils.DeviceUtils // IPification SDK
import java.util.Locale

/** Result of a SIM lookup: the raw MSISDN plus the ISO country of the SIM it came from. */
data class SimPhoneNumber(val number: String, val countryIso: String)

object SimPhoneNumberReader {

    private const val TAG = "PhoneFetch"

    val REQUIRED_PERMISSIONS = arrayOf(
        Manifest.permission.READ_PHONE_STATE,   // activeSubscriptionInfoList, line1Number
        Manifest.permission.READ_PHONE_NUMBERS  // SubscriptionManager.getPhoneNumber (API 33+)
    )

    fun hasPermissions(context: Context): Boolean = REQUIRED_PERMISSIONS.all {
        ContextCompat.checkSelfPermission(context, it) == PackageManager.PERMISSION_GRANTED
    }

    /**
     * Reads the phone number of the SIM currently used for data.
     * Returns null when permissions are missing or the carrier doesn't expose the MSISDN.
     * Safe to call from the main thread; never throws.
     */
    fun read(context: Context): SimPhoneNumber? {
        if (!hasPermissions(context)) {
            Log.w(TAG, "Phone permissions not granted")
            return null
        }

        val sm = context.getSystemService(SubscriptionManager::class.java)
        val tm = context.getSystemService(TelephonyManager::class.java)

        val subId = activeSubscriptionId(context)
        val countryIso = simCountryIso(sm, subId) ?: Locale.getDefault().country.uppercase()

        // Android 13+: SubscriptionManager.getPhoneNumber() is the supported API.
        // Older versions (and carriers that don't populate it): fall back to line1Number.
        val number = readWithSubscriptionManager(sm, subId)
            ?: readWithTelephonyManager(tm, subId)

        Log.i(TAG, "subId=$subId number=${number ?: "n/a"} country=$countryIso")
        return number?.let { SimPhoneNumber(it, countryIso) }
    }

    // ---- helpers ------------------------------------------------------------

    /** The SIM the IPification SDK considers active, else the system's default data SIM. */
    private fun activeSubscriptionId(context: Context): Int {
        val sdkSubId = DeviceUtils.getInstance(context).activeSimOperator().getSubID()
        return if (SubscriptionManager.isValidSubscriptionId(sdkSubId)) sdkSubId
        else SubscriptionManager.getDefaultDataSubscriptionId()
    }

    private fun simCountryIso(sm: SubscriptionManager, subId: Int): String? = safely {
        sm.activeSubscriptionInfoList
            ?.firstOrNull { it.subscriptionId == subId }
            ?.countryIso
            ?.takeIf { it.isNotBlank() }
            ?.uppercase()
    }

    private fun readWithSubscriptionManager(sm: SubscriptionManager, subId: Int): String? {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU) return null
        return safely { sm.getPhoneNumber(subId).takeIf { it.isNotBlank() } }
    }

    @Suppress("DEPRECATION") // line1Number is deprecated on 33+, but still the only option below it
    private fun readWithTelephonyManager(tm: TelephonyManager, subId: Int): String? = safely {
        tm.createForSubscriptionId(subId).line1Number?.takeIf { it.isNotBlank() }
    }

    /** Telephony calls throw SecurityException on some OEMs even with permissions granted. */
    private inline fun <T> safely(block: () -> T?): T? = try {
        block()
    } catch (e: SecurityException) {
        Log.e(TAG, "Telephony call rejected: ${e.message}")
        null
    }
}
```

---

### Step 2: Request Permissions and Use the Result (Compose)

```kotlin
@Composable
fun PhoneNumberPrefill(viewModel: ProcessViewModel) {
    val context = LocalContext.current

    // Shared outcome handler: number found → prefill it; otherwise prefill the dial code only.
    fun applyResult(result: SimPhoneNumber?) {
        if (result != null) {
            viewModel.onPhoneNumberFromHint(result.number)
        } else {
            viewModel.onCountryCodeFromHint(Util.getSystemDialCode(context))
        }
    }

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { grants ->
        val granted = SimPhoneNumberReader.REQUIRED_PERMISSIONS.all { grants[it] == true }
        applyResult(if (granted) SimPhoneNumberReader.read(context) else null)
    }

    LaunchedEffect(Unit) {
        if (SimPhoneNumberReader.hasPermissions(context)) {
            applyResult(SimPhoneNumberReader.read(context))
        } else {
            permissionLauncher.launch(SimPhoneNumberReader.REQUIRED_PERMISSIONS)
        }
    }
}
```

---

### How It Decides

| Step | What happens |
|------|--------------|
| 1 | Bail out with `null` if either permission is missing (also satisfies lint). |
| 2 | Pick the subscription id: the SDK's active SIM, else the system default-data SIM. |
| 3 | Look up that SIM's country ISO; fall back to the device locale. |
| 4 | On Android 13+, try `SubscriptionManager.getPhoneNumber(subId)`. |
| 5 | If that returns blank (or on Android < 13), try `TelephonyManager.line1Number` for the same SIM. |
| 6 | Return `SimPhoneNumber(number, countryIso)`, or `null` if the carrier doesn't expose the MSISDN. |

---

### ✅ Advantages

- Works on most Android versions
- Provides direct access to carrier/MSISDN info
- Can enumerate multiple SIMs

---

### ⚠️ Limitations

- Needs explicit permissions
- May return `null` depending on carrier
