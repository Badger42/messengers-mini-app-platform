# MAX Mini App: Restricted Test Access Implementation Plan

## 1. Goal

Set up a private-by-authorization test environment for the MAX Mini App without requiring Cisco VPN on the mobile device.

The frontend and backend are already hosted. The frontend must remain reachable over public HTTPS so that MAX can load it, but protected application functionality and data must only be available to approved MAX users.

The access decision must be made by the ASP.NET Core backend. Vue Router guards are for navigation and user experience only; they are not a security boundary.

## 2. Target Architecture

Use a dedicated test environment:

- **Test Mini App frontend:** `https://app-test.example.com`
- **Test API:** preferably exposed under the same origin, for example `https://app-test.example.com/api`
- **Test database:** separate from production
- **Test MAX bot:** preferably a separate bot with its own bot token and Mini App configuration

Request flow:

1. A tester opens the Mini App from MAX on a phone. No VPN is required on the phone.
2. MAX Bridge exposes the signed launch data as `window.WebApp.initData`.
3. Vue sends the original `initData` string to the test API.
4. ASP.NET Core validates the MAX signature and launch-data timestamp.
5. ASP.NET Core extracts the verified MAX user ID and checks the test allowlist.
6. If the user is allowed, the backend creates the application's existing server-side/HttpOnly-cookie session.
7. Vue Router allows navigation to protected routes.
8. Every protected API endpoint independently checks the authenticated session.

**Network exposure:** the frontend's static files may be publicly downloadable. Do not put secrets or sensitive data in the frontend bundle. Protect all user-specific data and operations at the API boundary.

## 3. Environment and Configuration

### 3.1 Separate test from production

- Deploy the test frontend and backend separately from production.
- Use a dedicated test database and test integrations where possible.
- Configure a dedicated MAX test bot, if practical, to isolate bot settings and credentials.
- Keep test and production bot tokens separate.
- Do not reuse production credentials in the test deployment.

### 3.2 Backend configuration

Configure the following values on the server, preferably through environment variables or a secret manager:

- `MaxMiniApp:BotToken` — the test bot token. Never expose it to Vue or commit it to source control.
- `MaxMiniApp:AllowedUserIds` or an equivalent database-backed allowlist.
- `MaxMiniApp:MaxInitDataAge` — initially one hour, following the MAX Bridge recommendation.
- `MaxMiniApp:AllowedFutureClockSkew` — a small tolerance for clock differences, for example two minutes.

The allowlist should contain numeric MAX user IDs, not usernames or display names. Usernames can change; the MAX ID is the identifier used for the access decision.

For a first iteration, configuration-based IDs are acceptable. A database table is preferable if administrators need to add or remove testers without redeploying.

## 4. Frontend Implementation (Vue 3)

### 4.1 Obtain the original launch data

Ensure the MAX Bridge script is loaded before the Vue application attempts to read the global object:

```html
<script src="https://st.max.ru/js/max-web-app.js"></script>
```

Read the original signed string:

```js
const initData = window.WebApp?.initData;
```

Use `initData`, not `initDataUnsafe`, for authentication. `initDataUnsafe` is a parsed convenience object and must not be treated as proof of identity.

Do not send a separately supplied `userId` from the frontend. The backend must extract the user ID only after validating the signature.

### 4.2 Create a MAX authentication service

Implement a small frontend service, for example `maxAuthService.js`, that:

- Reads `window.WebApp.initData`.
- Fails closed if the value is absent or empty.
- Sends the exact original string to `POST /api/auth/max`.
- Uses the application's configured Axios instance and `withCredentials: true` if required by the existing cookie setup.
- Returns an explicit authorization result.
- Caches the in-flight Promise/result for the current app instance so route changes do not repeat the authentication request.
- Clears cached state after logout or when the backend reports an expired/invalid session and a new authentication attempt is required.

Do not cache the result in local storage as a substitute for server authorization. A page reload should re-establish the session through the backend flow.

### 4.3 Add Vue Router guards

Mark routes that require tester access, for example:

```js
{
  path: '/documents',
  name: 'documents',
  component: () => import('@/views/Documents.vue'),
  meta: { requiresMaxAccess: true }
}
```

Add an asynchronous `router.beforeEach` guard:

- Public routes such as an access-denied page can be excluded.
- For protected routes, call the authentication service.
- Allow navigation only after the backend returns success.
- Redirect unauthorized users to a dedicated access-denied page.
- Handle network/server errors separately from an explicit `403`, so a temporary outage is not presented as a definitive allowlist denial.

The route guard improves navigation flow only. It must never be the only access control.

## 5. Backend Implementation (ASP.NET Core)

Implement the validation and allowlist check as one authentication operation.

### 5.1 Create a validator service

Introduce a service such as `IMaxInitDataValidator` / `MaxInitDataValidator`.

Input:

- The original string received from `window.WebApp.initData`.
- The bot token loaded from server-side configuration.

Output:

- A validation result containing the verified MAX user ID and `auth_date`, or a validation failure.

The validator must follow the MAX documentation's `WebAppData` algorithm:

1. Parse the `initData` string into key/value pairs. Split each pair at the **first** `=` so values containing `=` are not truncated.
2. Reject malformed pairs and duplicate parameter names. Require exactly one `hash` parameter.
3. URL-decode each parameter value exactly once using query/form decoding semantics from the official Java `URLDecoder.decode` and Go `url.QueryUnescape` examples: a raw `+` becomes a space, while `%2B` becomes a literal `+`. In C#, use `System.Net.WebUtility.UrlDecode`.
4. Save the received `hash` and exclude it from the signed parameter set.
5. Sort the remaining parameters by key in ordinal/alphabetical order.
6. Build `launch_params` as `key=value` lines separated by `\n` (LF, `0x0A`), with no extra spaces.
7. Derive the secret key:

   ```text
   secret_key = HMAC-SHA256(key = UTF8("WebAppData"), data = UTF8(BOT_TOKEN))
   ```

8. Calculate the signature:

   ```text
   calculated_hash = HMAC-SHA256(key = secret_key, data = UTF8(launch_params))
   ```

9. Convert the calculated digest to lowercase hexadecimal and compare it with the received hash using a constant-time comparison.
10. Only after signature validation, parse `auth_date` and the `user` JSON object.
11. Require a valid numeric `user.id` and a valid Unix timestamp in seconds.
12. Reject launch data older than the configured maximum age (initially one hour) and data that is unreasonably far in the future (allow only a small clock-skew tolerance).

Implementation notes:

- Keep the token exclusively on the server.
- Use UTF-8 for all cryptographic inputs.
- Use `CryptographicOperations.FixedTimeEquals` for digest comparison.
- Validate malformed percent-encoding and malformed JSON as failures.
- Do not trust `user.id` until the HMAC check has succeeded.
- Do not log the raw `initData`, bot token, or full user payload.

### 5.2 Add an allowlist service

Introduce an abstraction such as `IAllowedMaxUsers` with a method similar to:

```csharp
Task<bool> IsAllowedAsync(long maxUserId, CancellationToken cancellationToken);
```

The implementation can initially read IDs from configuration. It can later be replaced with a database-backed implementation without changing the validator or controller.

The allowlist check must use the verified ID returned by `MaxInitDataValidator`; it must not accept a user ID supplied separately by the browser.

### 5.3 Add the authentication endpoint

Create:

```http
POST /api/auth/max
Content-Type: application/json
```

Request body:

```json
{
  "initData": "<original-window.WebApp.initData-string>"
}
```

Endpoint flow:

1. Validate the request shape and size.
2. Validate the MAX signature and `auth_date`.
3. Extract the verified MAX user ID.
4. Check the allowlist.
5. If allowed, create the application's existing authentication session and issue its HttpOnly cookie.
6. Return success without exposing any bot secret or unnecessary personal data.

Suggested response semantics:

- `400 Bad Request` — malformed request body.
- `401 Unauthorized` — missing, invalid, tampered, or expired MAX launch data.
- `403 Forbidden` — valid MAX identity, but the user is not on the tester allowlist.
- `204 No Content` — authentication/session creation succeeded.

Adapt status codes to existing API conventions if necessary, but keep invalid authentication distinct from a valid user who is not authorized for testing.

### 5.4 Reuse the existing HttpOnly-cookie authentication

Do not introduce a second, unrelated JWT mechanism if the application already uses HttpOnly cookies.

After successful MAX validation and allowlist approval:

- Create the normal application session using the existing authentication service.
- Store the verified MAX user ID in the authenticated principal/session.
- Set the cookie with `Secure` and `HttpOnly`.
- Prefer a same-origin frontend/API arrangement and a host-only cookie (omit `Domain` unless there is a specific reason to share it).
- Choose `SameSite` according to the actual deployment topology and MAX WebView behavior. If frontend and API are exposed through one origin, `Lax` or `Strict` may be possible; test the chosen policy in MAX. Cross-site deployments may require `SameSite=None; Secure` and careful CORS configuration.
- Add or retain CSRF protection for state-changing cookie-authenticated requests according to the application's existing security model.

Every protected API endpoint must validate the session independently. The API must not trust the Vue route state or a client-side `isAuthorized` flag.

## 6. Hosting and HTTP Routing

Prefer a same-origin public URL for the Mini App and its API:

```text
https://app-test.example.com/       -> test Vue frontend
https://app-test.example.com/api/*  -> test ASP.NET Core API
```

This can be implemented through the existing hosting reverse proxy or web server.

Advantages:

- Avoids cross-origin browser requests for the normal frontend/API flow.
- Simplifies cookie behavior.
- Avoids the need for credentialed CORS between different origins.

If the API must remain on a separate origin:

- Configure CORS with the exact frontend origin, not `*`.
- Enable credentials only for the required origin.
- Configure cookie attributes for the actual site relationship.
- Test cookie persistence and API calls inside MAX on both Android and iOS where available.

The initial HTML and static frontend assets must be reachable by MAX without Cisco VPN. Protected API data and operations remain inaccessible until the backend issues a valid session.

## 7. Security Requirements

- Never include the MAX bot token in frontend code, source maps, public configuration, or browser requests.
- Never authenticate using `initDataUnsafe`, a frontend-provided MAX user ID, username, or display name.
- Validate the signature on every new authentication attempt.
- Enforce the recommended `auth_date` freshness window (initially one hour).
- Use constant-time hash comparison.
- Reject duplicate parameters and malformed input.
- Apply request-size limits and rate limiting to the authentication endpoint.
- Do not log raw `initData`, bot tokens, cookies, or session secrets.
- Keep production and test bot tokens, databases, and session settings separate.
- Protect all data-bearing API endpoints with backend authorization.
- Consider one-time/replay protection using a validated launch identifier such as `query_id` if the threat model requires it. Define retention and refresh behavior before enabling this, because legitimate reloads/retries may reuse launch data.
- Treat a public frontend as a delivery mechanism, not as a security boundary.

## 8. Testing Plan

### 8.1 Validator unit tests

Cover at least:

- Valid MAX sample data produces the expected verified user ID.
- Changing `user.id` without recomputing the signature is rejected.
- Changing `auth_date` without recomputing the signature is rejected.
- Missing `hash` is rejected.
- Duplicate `hash` is rejected.
- Duplicate non-hash parameters are rejected.
- Malformed key/value pairs are rejected.
- Malformed percent-encoding is rejected.
- A raw `+` in a value is decoded as a space.
- An encoded `%2B` in a value is decoded as a literal `+`; `%20` is decoded as a space.
- Values containing encoded `&`, `=`, and Unicode are handled correctly after one decoding pass.
- Incorrect hash is rejected.
- Expired data is rejected.
- Excessively future-dated data is rejected.
- Missing/malformed `user` JSON or missing/non-numeric `user.id` is rejected.

Use an official MAX sample or a locally generated deterministic test vector. Do not use a real production bot token in tests.

### 8.2 API integration tests

- Valid launch data for an allowlisted user returns success and sets the expected HttpOnly cookie.
- Valid launch data for a non-allowlisted user returns `403` and does not create a session.
- Invalid or expired launch data returns `401` and does not create a session.
- Protected endpoints reject requests without a valid session.
- Protected endpoints accept requests with the issued session.
- Removing a user from the allowlist has the intended effect. Decide whether existing sessions are allowed to remain valid until expiry or must be revoked immediately.

### 8.3 MAX client tests

- Open the test Mini App from MAX on a phone without Cisco VPN.
- Confirm `window.WebApp.initData` is available.
- Confirm the authentication request reaches the test API.
- Confirm the cookie is stored and sent on subsequent API calls inside the MAX WebView.
- Confirm an allowlisted tester can use protected routes and API operations.
- Confirm a non-allowlisted user receives the access-denied experience.
- Confirm opening the frontend outside MAX does not grant access to protected functionality.
- Test the cookie and navigation behavior on Android and iOS if both platforms are in scope.

## 9. Acceptance Criteria

The implementation is complete when:

- The test Mini App opens in MAX without Cisco VPN on the phone.
- The browser sends the original `window.WebApp.initData` string to the test backend.
- The backend validates the signature exactly according to the current MAX documentation.
- The backend rejects invalid, modified, duplicated, malformed, and expired launch data.
- Only verified MAX user IDs present in the test allowlist receive an authenticated session.
- The existing HttpOnly-cookie authentication mechanism is reused.
- Vue Router blocks protected navigation for users who have not passed backend authorization.
- Protected API endpoints independently enforce authentication/authorization.
- Production data and credentials remain isolated from the test environment.

## 10. Official MAX References

- [MAX Mini App data validation](https://dev.max.ru/docs/webapps/validation)
- [MAX Bridge — initialization data (`initData` and `initDataUnsafe`)](https://dev.max.ru/docs/webapps/bridge/)

The MAX Bridge documentation describes `window.WebApp.initData` as the URL-encoded launch-data string intended for server-side validation. It explicitly warns that `initDataUnsafe` must not be used for validation. It also describes `auth_date` as the timestamp used to determine data invalidation and recommends an interval of one hour.

## 11. End-to-End Code Examples

The following examples turn the plan into a concrete implementation. They use ASP.NET Core and Vue 3 with Vue Router 4. Adapt namespaces, dependency injection, API conventions, and the existing authentication service to the application.

> **Important:** `window.WebApp.initData` is the URL-encoded launch-data string intended for server-side validation. Pass that exact string to the API. Do not send the whole browser URL, `initDataUnsafe`, or a separately supplied `userId`. The `WebAppPlatform` and `WebAppVersion` wrapper parameters are not part of the signed `initData` payload.

### 11.1. Backend configuration

Example configuration:

```json
{
  "MaxMiniApp": {
    "MaxInitDataAgeSeconds": 3600,
    "AllowedFutureClockSkewSeconds": 120,
    "AllowedUserIds": [
      123456789,
      987654321
    ]
  }
}
```

Do not store `BotToken` in a committed configuration file. Supply it through an environment variable or secret manager, for example:

```text
MaxMiniApp__BotToken=YOUR_TEST_BOT_TOKEN
```

The numeric IDs above are illustrative only. Replace them with the MAX user IDs of the approved testers.

Create an options class:

```csharp
using System;

public sealed class MaxMiniAppOptions
{
    public string BotToken { get; set; } = string.Empty;

    public int MaxInitDataAgeSeconds { get; set; } = 3600;

    public int AllowedFutureClockSkewSeconds { get; set; } = 120;

    public long[] AllowedUserIds { get; set; } = Array.Empty<long>();
}
```

`MaxInitDataAgeSeconds` controls how old the signed launch data may be. MAX recommends a one-hour validity interval. The future clock-skew allowance accounts for a small difference between the MAX and server clocks; it is not an extension of the normal validity period.

### 11.2. Define the validation result and service contract

```csharp
using System;

public sealed record MaxLaunchData(
    long UserId,
    DateTimeOffset AuthDate);

public interface IMaxInitDataValidator
{
    bool TryValidate(
        string? initData,
        out MaxLaunchData? launchData);
}
```

The validator returns the user ID only after the signature has been verified. This prevents the controller from accidentally using an untrusted ID supplied by the browser.

### 11.3. Implement the MAX HMAC validation service

`MaxInitDataValidator.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;
using System.Net;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using Microsoft.Extensions.Options;

public sealed class MaxInitDataValidator : IMaxInitDataValidator
{
    private readonly MaxMiniAppOptions _options;

    public MaxInitDataValidator(IOptions<MaxMiniAppOptions> options)
    {
        _options = options.Value;
    }

    public bool TryValidate(
        string? initData,
        out MaxLaunchData? launchData)
    {
        launchData = null;

        if (string.IsNullOrWhiteSpace(initData) ||
            string.IsNullOrWhiteSpace(_options.BotToken))
        {
            return false;
        }

        // Prevent unexpectedly large payloads from consuming resources.
        if (initData.Length > 16_384)
            return false;

        // MAX provides initData as a URL-encoded query string.
        // Split at the first '=' because a value may contain '=' characters.
        var parameters = new Dictionary<string, string>(StringComparer.Ordinal);

        foreach (var part in initData.Split('&'))
        {
            var separator = part.IndexOf('=');

            if (separator <= 0)
                return false;

            var key = part[..separator];
            var encodedValue = part[(separator + 1)..];

            if (string.IsNullOrEmpty(key) ||
                !HasValidPercentEncoding(encodedValue))
            {
                return false;
            }

            // Match the URLDecoder / QueryUnescape behavior used in MAX's
            // Java and Go examples: '+' becomes a space and '%2B' becomes '+'.
            var value = WebUtility.UrlDecode(encodedValue);

            // Reject duplicate names rather than allowing ambiguous input.
            if (!parameters.TryAdd(key, value))
                return false;
        }

        if (!parameters.TryGetValue("hash", out var receivedHash))
            return false;

        // Build launch_params from every parameter except 'hash'.
        // Use ordinal key ordering and LF ('\n') separators exactly as MAX specifies.
        var launchParams = string.Join(
            "\n",
            parameters
                .Where(parameter => parameter.Key != "hash")
                .OrderBy(parameter => parameter.Key, StringComparer.Ordinal)
                .Select(parameter => $"{parameter.Key}={parameter.Value}"));

        // First HMAC:
        // secret_key = HMAC-SHA256(key: "WebAppData", data: BOT_TOKEN)
        var secretKey = ComputeHmac(
            Encoding.UTF8.GetBytes("WebAppData"),
            Encoding.UTF8.GetBytes(_options.BotToken));

        // Second HMAC:
        // calculated_hash = HMAC-SHA256(key: secret_key, data: launch_params)
        var calculatedHash = ComputeHmac(
            secretKey,
            Encoding.UTF8.GetBytes(launchParams));

        byte[] receivedHashBytes;

        try
        {
            receivedHashBytes = Convert.FromHexString(receivedHash);
        }
        catch (FormatException)
        {
            return false;
        }

        // SHA-256 produces exactly 32 bytes. Compare digest bytes in constant time.
        if (receivedHashBytes.Length != 32 ||
            !CryptographicOperations.FixedTimeEquals(
                calculatedHash,
                receivedHashBytes))
        {
            return false;
        }

        // Do not read identity fields until the signature has been verified.
        if (!parameters.TryGetValue("auth_date", out var authDateValue) ||
            !long.TryParse(
                authDateValue,
                NumberStyles.None,
                CultureInfo.InvariantCulture,
                out var authTimestamp))
        {
            return false;
        }

        DateTimeOffset authDate;

        try
        {
            authDate = DateTimeOffset.FromUnixTimeSeconds(authTimestamp);
        }
        catch (ArgumentOutOfRangeException)
        {
            return false;
        }

        var now = DateTimeOffset.UtcNow;
        var maxAge = TimeSpan.FromSeconds(_options.MaxInitDataAgeSeconds);
        var allowedFutureSkew = TimeSpan.FromSeconds(
            _options.AllowedFutureClockSkewSeconds);

        if (_options.MaxInitDataAgeSeconds <= 0 ||
            _options.AllowedFutureClockSkewSeconds < 0 ||
            authDate > now.Add(allowedFutureSkew) ||
            now - authDate > maxAge)
        {
            return false;
        }

        if (!parameters.TryGetValue("user", out var userJson))
            return false;

        try
        {
            using var document = JsonDocument.Parse(userJson);
            var root = document.RootElement;

            if (root.ValueKind != JsonValueKind.Object ||
                !root.TryGetProperty("id", out var idElement) ||
                !idElement.TryGetInt64(out var userId) ||
                userId <= 0)
            {
                return false;
            }

            launchData = new MaxLaunchData(userId, authDate);
            return true;
        }
        catch (JsonException)
        {
            return false;
        }
    }

    private static byte[] ComputeHmac(byte[] key, byte[] data)
    {
        using var hmac = new HMACSHA256(key);
        return hmac.ComputeHash(data);
    }

    private static bool HasValidPercentEncoding(string value)
    {
        for (var index = 0; index < value.Length; index++)
        {
            if (value[index] != '%')
                continue;

            if (index + 2 >= value.Length ||
                !Uri.IsHexDigit(value[index + 1]) ||
                !Uri.IsHexDigit(value[index + 2]))
            {
                return false;
            }

            index += 2;
        }

        return true;
    }
}
```

Implementation notes:

- The first HMAC uses `WebAppData` as the key and the bot token as the message. The second HMAC uses the derived secret key and the canonical `launch_params` string.
- Decode each parameter value exactly once with `WebUtility.UrlDecode` (UTF-8, query/form semantics), matching the official Java and Go examples. Do not use `Uri.UnescapeDataString`, which leaves a raw `+` unchanged.
- Reject duplicate parameters, including duplicate `hash` values, to avoid ambiguous input.
- Verify the signature before parsing `user.id` or using it for authorization.
- Never log `initData`, the bot token, or the complete user payload.
- Keep this implementation covered by test vectors from the current MAX documentation. Do not rely only on manually constructed payloads.

### 11.4. Implement the tester allowlist

Start with an interface so configuration storage can later be replaced by a database without changing the controller:

```csharp
public interface IAllowedMaxUsers
{
    Task<bool> IsAllowedAsync(
        long maxUserId,
        CancellationToken cancellationToken);
}
```

A simple configuration-backed implementation:

```csharp
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.Options;

public sealed class ConfigurationAllowedMaxUsers : IAllowedMaxUsers
{
    private readonly IOptions<MaxMiniAppOptions> _options;

    public ConfigurationAllowedMaxUsers(
        IOptions<MaxMiniAppOptions> options)
    {
        _options = options;
    }

    public Task<bool> IsAllowedAsync(
        long maxUserId,
        CancellationToken cancellationToken)
    {
        cancellationToken.ThrowIfCancellationRequested();

        var isAllowed = _options.Value.AllowedUserIds.Contains(maxUserId);
        return Task.FromResult(isAllowed);
    }
}
```

For a production-quality administration workflow, replace this implementation with a database-backed service. The controller should continue to receive only the verified MAX user ID from the validator.

### 11.5. Register options and services

In `Program.cs`:

```csharp
builder.Services
    .AddOptions<MaxMiniAppOptions>()
    .Bind(builder.Configuration.GetSection("MaxMiniApp"))
    .Validate(options => !string.IsNullOrWhiteSpace(options.BotToken),
        "MaxMiniApp:BotToken must be configured.")
    .Validate(options => options.MaxInitDataAgeSeconds > 0,
        "MaxInitDataAgeSeconds must be positive.")
    .Validate(options => options.AllowedFutureClockSkewSeconds >= 0,
        "AllowedFutureClockSkewSeconds cannot be negative.")
    .ValidateOnStart();

builder.Services.AddSingleton<
    IMaxInitDataValidator,
    MaxInitDataValidator>();

builder.Services.AddScoped<
    IAllowedMaxUsers,
    ConfigurationAllowedMaxUsers>();
```

The validator is stateless and reads immutable options, so it can be registered as a singleton. The allowlist service is registered as scoped here so its implementation can later use a scoped database context.

### 11.6. Create the MAX authentication endpoint

Request model:

```csharp
public sealed record MaxAuthRequest(string InitData);
```

Controller example:

```csharp
using System.Globalization;
using System.Security.Claims;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/auth")]
public sealed class MaxAuthController : ControllerBase
{
    private readonly IMaxInitDataValidator _validator;
    private readonly IAllowedMaxUsers _allowedUsers;

    public MaxAuthController(
        IMaxInitDataValidator validator,
        IAllowedMaxUsers allowedUsers)
    {
        _validator = validator;
        _allowedUsers = allowedUsers;
    }

    [AllowAnonymous]
    [HttpPost("max")]
    public async Task<IActionResult> Authenticate(
        [FromBody] MaxAuthRequest request,
        CancellationToken cancellationToken)
    {
        if (request is null ||
            string.IsNullOrWhiteSpace(request.InitData) ||
            request.InitData.Length > 16_384)
        {
            return BadRequest();
        }

        if (!_validator.TryValidate(request.InitData, out var launchData) ||
            launchData is null)
        {
            return Unauthorized();
        }

        var isAllowed = await _allowedUsers.IsAllowedAsync(
            launchData.UserId,
            cancellationToken);

        if (!isAllowed)
            return StatusCode(StatusCodes.Status403Forbidden);

        // Example only: if the application already has an authentication
        // service, call that service instead of creating a parallel identity flow.
        var claims = new[]
        {
            new Claim(
                ClaimTypes.NameIdentifier,
                launchData.UserId.ToString(CultureInfo.InvariantCulture)),
            new Claim(
                "max_user_id",
                launchData.UserId.ToString(CultureInfo.InvariantCulture))
        };

        var identity = new ClaimsIdentity(
            claims,
            CookieAuthenticationDefaults.AuthenticationScheme);

        var principal = new ClaimsPrincipal(identity);

        await HttpContext.SignInAsync(
            CookieAuthenticationDefaults.AuthenticationScheme,
            principal,
            new AuthenticationProperties
            {
                IsPersistent = false,
                ExpiresUtc = DateTimeOffset.UtcNow.AddHours(8)
            });

        return NoContent();
    }
}
```


If the application maps MAX accounts to its own internal user records, perform that mapping after validation and allowlist approval. Use the internal application user ID as the principal's `NameIdentifier`, and keep the verified MAX ID in a separate claim if needed. Prefer calling the existing authentication/session service rather than maintaining a second cookie format.

The endpoint is `[AllowAnonymous]` because the user has no application session yet. It must still validate the signed MAX data and allowlist before issuing a cookie. Return API status codes instead of redirecting to an HTML login page.

### 11.7. Implement the frontend MAX authentication service

`src/services/maxAuthService.js`:

```javascript
import api from '@/api/client';

let authPromise = null;

export function ensureMaxAccess() {
  if (!authPromise) {
    authPromise = authenticate().then(result => {
      // A transient network/server failure should be retryable.
      if (result.status === 'error') {
        authPromise = null;
      }

      return result;
    });
  }

  return authPromise;
}

export function resetMaxAccess() {
  authPromise = null;
}

async function authenticate() {
  const initData = window.WebApp?.initData;

  if (!initData) {
    return { status: 'missing-init-data' };
  }

  try {
    await api.post('/auth/max', { initData });
    return { status: 'allowed' };
  } catch (error) {
    const statusCode = error.response?.status;

    if (statusCode === 401) {
      return { status: 'invalid-init-data' };
    }

    if (statusCode === 403) {
      return { status: 'not-allowed' };
    }

    return { status: 'error' };
  }
}
```

Explanation:

- The service sends the exact `window.WebApp.initData` string. It does not send `initDataUnsafe.user.id`.
- `authPromise` prevents duplicate authentication requests when multiple protected routes are entered during the same app session.
- A successful result is reused while the current Vue application instance remains active.
- A `401` means the launch data could not be authenticated; a `403` means the identity was validated but is not on the tester allowlist.
- Network and server failures are reported separately and are retryable. Provide a retry action in the error screen that calls `resetMaxAccess()` before attempting authentication again.
- Do not store an `isAuthorized` flag in local storage. The backend session remains the source of truth.

If the backend session expires while the Mini App remains open, handle a `401` from protected API calls by resetting the frontend auth state and running the authentication flow again when the current `initData` is still valid. If the launch data has expired, ask the user to close and reopen the Mini App to obtain fresh launch data.

### 11.10. Add a Vue Router navigation guard

Mark protected routes:

```javascript
{
  path: '/documents',
  name: 'documents',
  component: () => import('@/views/Documents.vue'),
  meta: { requiresMaxAccess: true }
}
```

Create public routes for `access-denied` and `auth-error`, then add the guard:

```javascript
import { ensureMaxAccess } from '@/services/maxAuthService';

router.beforeEach(async to => {
  if (!to.meta.requiresMaxAccess) {
    return true;
  }

  const result = await ensureMaxAccess();

  if (result.status === 'allowed') {
    return true;
  }

  if (
    result.status === 'not-allowed' ||
    result.status === 'invalid-init-data' ||
    result.status === 'missing-init-data'
  ) {
    return {
      name: 'access-denied',
      replace: true
    };
  }

  return {
    name: 'auth-error',
    replace: true
  };
});
```

This guard prevents unauthorized users from navigating to protected UI routes and distinguishes an access denial from a temporary API/network failure. It is not a security boundary: every protected API endpoint must still enforce the authenticated backend session and any required application permissions.

### 11.11. Authentication and authorization flow

```text
MAX opens the Mini App
        |
        v
window.WebApp.initData
        |
        v
Vue Router beforeEach
        |
        v
POST /api/auth/max { initData }
        |
        v
ASP.NET Core validates HMAC and auth_date
        |
        v
Extract verified MAX user ID
        |
        v
Check tester allowlist
        |
   +----+----+
   |         |
 allowed   not allowed
   |         |
   v         v
Issue      Return 403
HttpOnly      |
 session      v
   |       Access-denied page
   v
Allow protected route
   |
   v
Protected API endpoints validate the session on every request
```

The frontend route guard controls navigation and user experience. The backend's signature validation, allowlist check, session issuance, and authorization on protected endpoints provide the actual security controls.
