# Configure custom endpoint URLs

Use custom URLs when your integration requires coverage and authorization endpoints other than the SDK defaults. You can configure custom endpoints for both staging and production.

Each endpoint must be a complete HTTPS URL, including the hostname and full path. You can customize both the hostname and the path; you do not have to use the default paths:

- Coverage: `/auth/realms/ipification/coverage`
- Authorization: `/auth/realms/ipification/protocol/openid-connect/auth`

Use the exact endpoint URLs provided by IPification. Do not append the default paths to a complete endpoint URL.

## Before you begin

Make sure you have the following values for each environment:

- The complete coverage endpoint URL, if using custom URLs.
- The complete authorization endpoint URL, if using custom URLs.
- The client ID and redirect URI registered for that environment.

Configure the SDK before sending any coverage or authorization requests. The environment, endpoint URLs, client ID, and redirect URI must all correspond to the same environment.

## Configuration options

The examples below use two parameters:

- `isProduction`: Set to `true` for production or `false` for staging.
- `useCustomUrls`: Set to `true` to override both endpoint URLs or `false` to use the SDK’s default URLs for the selected environment.

Replace all placeholders with the values provided for your integration.

## iOS (Swift)

```swift
func configureIPification(
    isProduction: Bool,
    useCustomUrls: Bool
) {
    let configuration = IPConfiguration.sharedInstance

    configuration.ENV = isProduction
        ? IPEnvironment.PRODUCTION
        : IPEnvironment.SANDBOX

    configuration.customUrls = useCustomUrls

    if isProduction {
        configuration.CLIENT_ID = "<production-client-id>"
        configuration.REDIRECT_URL = "<production-redirect-uri>"

        if useCustomUrls {
            configuration.COVERAGE_URL =
                "https://<production-coverage-host>/<coverage-path>"
            configuration.AUTHORIZATION_URL =
                "https://<production-authorization-host>/<authorization-path>"
        }
    } else {
        configuration.CLIENT_ID = "<staging-client-id>"
        configuration.REDIRECT_URL = "<staging-redirect-uri>"

        if useCustomUrls {
            configuration.COVERAGE_URL =
                "https://<staging-coverage-host>/<coverage-path>"
            configuration.AUTHORIZATION_URL =
                "https://<staging-authorization-host>/<authorization-path>"
        }
    }
}
```

Choose the configuration that matches your integration:

```swift
// Staging with custom endpoint URLs.
configureIPification(isProduction: false, useCustomUrls: true)

// Production with the SDK's default endpoint URLs.
configureIPification(isProduction: true, useCustomUrls: false)

// Production with custom endpoint URLs.
configureIPification(isProduction: true, useCustomUrls: true)
```

Call only the configuration appropriate for your environment before starting SDK requests.

## Android (Kotlin)

```kotlin
import android.net.Uri

fun configureIPification(
    isProduction: Boolean,
    useCustomUrls: Boolean
) {
    val configuration = IPConfiguration.getInstance()

    configuration.ENV = if (isProduction) {
        IPEnvironment.PRODUCTION
    } else {
        IPEnvironment.SANDBOX
    }

    configuration.customUrls = useCustomUrls

    if (isProduction) {
        configuration.CLIENT_ID = "<production-client-id>"
        configuration.REDIRECT_URL = "<production-redirect-uri>"

        if (useCustomUrls) {
            configuration.COVERAGE_URL = Uri.parse(
                "https://<production-coverage-host>/<coverage-path>"
            )
            configuration.AUTHORIZATION_URL = Uri.parse(
                "https://<production-authorization-host>/<authorization-path>"
            )
        }
    } else {
        configuration.CLIENT_ID = "<staging-client-id>"
        configuration.REDIRECT_URL = "<staging-redirect-uri>"

        if (useCustomUrls) {
            configuration.COVERAGE_URL = Uri.parse(
                "https://<staging-coverage-host>/<coverage-path>"
            )
            configuration.AUTHORIZATION_URL = Uri.parse(
                "https://<staging-authorization-host>/<authorization-path>"
            )
        }
    }
}
```

Choose the configuration that matches your integration:

```kotlin
// Staging with custom endpoint URLs.
configureIPification(isProduction = false, useCustomUrls = true)

// Production with the SDK's default endpoint URLs.
configureIPification(isProduction = true, useCustomUrls = false)

// Production with custom endpoint URLs.
configureIPification(isProduction = true, useCustomUrls = true)
```

Call only the configuration appropriate for your environment before starting SDK requests.

## Android (Java)

```java
import android.net.Uri;

private void configureIPification(
        boolean isProduction,
        boolean useCustomUrls
) {
    IPConfiguration configuration = IPConfiguration.getInstance();

    configuration.setENV(
        isProduction
            ? IPEnvironment.PRODUCTION
            : IPEnvironment.SANDBOX
    );

    configuration.setCustomUrls(useCustomUrls);

    if (isProduction) {
        configuration.setCLIENT_ID("<production-client-id>");
        configuration.setREDIRECT_URI("<production-redirect-uri>");

        if (useCustomUrls) {
            configuration.setCOVERAGE_URL(Uri.parse(
                "https://<production-coverage-host>/<coverage-path>"
            ));
            configuration.setAUTHORIZATION_URL(Uri.parse(
                "https://<production-authorization-host>/<authorization-path>"
            ));
        }
    } else {
        configuration.setCLIENT_ID("<staging-client-id>");
        configuration.setREDIRECT_URI("<staging-redirect-uri>");

        if (useCustomUrls) {
            configuration.setCOVERAGE_URL(Uri.parse(
                "https://<staging-coverage-host>/<coverage-path>"
            ));
            configuration.setAUTHORIZATION_URL(Uri.parse(
                "https://<staging-authorization-host>/<authorization-path>"
            ));
        }
    }
}
```

Choose the configuration that matches your integration:

```java
// Staging with custom endpoint URLs.
configureIPification(false, true);

// Production with the SDK's default endpoint URLs.
configureIPification(true, false);

// Production with custom endpoint URLs.
configureIPification(true, true);
```

Call only the configuration appropriate for your environment before starting SDK requests.

## Verify your configuration

Before testing or releasing your application, confirm that:

1. The selected SDK environment matches the intended staging or production environment.
2. Custom URLs are enabled only when you intend to override the SDK’s default endpoints.
3. When custom URLs are enabled, both endpoints contain complete HTTPS URLs with the correct hostnames and paths.
4. The client ID and redirect URI are registered for the selected environment.
5. All configuration values are set before the first coverage or authorization request.
6. Production builds use production endpoints and credentials.

Manage endpoint URLs through your build configuration or another controlled configuration source. Do not allow end users to supply arbitrary endpoint URLs.
