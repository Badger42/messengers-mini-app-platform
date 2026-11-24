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
3. URL-decode parameter values once, following the equivalent of JavaScript `decodeURIComponent`. Do not accidentally convert literal `+` characters to spaces as form-style decoding can do.
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
- Values containing encoded `&`, `=`, Unicode, or literal `+` are handled correctly.
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
