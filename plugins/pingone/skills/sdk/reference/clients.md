# Clients: packages, construction, lifecycle

Read alongside `SKILL.md`. Minimal construction only. There is no scaffolding here on purpose: these SDKs go into an application that already exists, and a generated sample app is a different task from wiring one in.

## Packages

| Purpose | Web | Android | iOS |
| --- | --- | --- | --- |
| DaVinci flow client | `@forgerock/davinci-client` | `com.pingidentity.sdks:davinci` | `Ping/ping-ios-sdk`, `PingDavinciPlugin` |
| OIDC tokens | `@forgerock/oidc-client` | the `Oidc` module, in-artefact | `PingOidc` |
| PingOne Protect | `@forgerock/protect` | `:protect` | `PingProtect` |
| Device binding and push | `@forgerock/device-client` | `:binding`, `:push` | platform equivalents |

The Android 2.x major absorbed the MFA platform as `:binding` and `:push`, removing the separate PingOne MFA Mobile SDK dependency.

Do not reach for `@forgerock/journey-client` or `com.pingidentity.sdks:journey`. Journey is callback-based authentication against PingAM or PingOne Advanced Identity Cloud, a different orchestration server with a different response shape. A Journey import in a PingOne DaVinci codebase is a mistake, not a variant.

## Web

```javascript
import { davinci } from '@forgerock/davinci-client'

const davinciClient = await davinci({
  config: {
    clientId: '...',
    serverConfig: {
      wellknown: 'https://auth.pingone.com/<envId>/as/.well-known/openid-configuration',
    },
    scope: 'openid profile email',
    redirectUri: 'https://your-host/callback',
  },
})
```

The `wellknown` URL is the whole server configuration: there is no `serverUrl`, `realm` or `cookie`, those are Journey fields. The `redirectUri` must match the value registered on the PingOne application exactly, and the application's CORS allow-list must contain the host page's origin or the first call fails before any flow starts.

| Call | Returns | Notes |
| --- | --- | --- |
| `davinciClient.start()` | node | begins the flow |
| `davinciClient.getNode()` | node state | the whole object |
| `davinciClient.getClient()` | `{ status, name, action, ... }` | current node summary |
| `davinciClient.getCollectors()` | `Collector[]` | for the current node |
| `davinciClient.getError()` | error or null | when status is `error` |
| `davinciClient.update(collector)` | updater | call it with the new value |
| `davinciClient.validate(collector)` | validator | validated collectors only; returns `string[]` |
| `davinciClient.next()` | node | submits the form |
| `davinciClient.flow({ action })` | function | call it to claim a named exit |

Status is `start`, `continue`, `success` or `error`. An `error` node still carries collectors: re-render them.

## Android

```kotlin
import com.pingidentity.davinci.DaVinci
import com.pingidentity.davinci.module.Oidc
import com.pingidentity.logger.Logger
import com.pingidentity.logger.STANDARD   // top-level extension, needs its own import

val daVinci = DaVinci {
    logger = Logger.STANDARD
    module(Oidc) {
        clientId          = BuildConfig.CLIENT_ID
        discoveryEndpoint = BuildConfig.DISCOVERY_ENDPOINT
        scopes            = mutableSetOf("openid", "profile", "email")
        redirectUri       = BuildConfig.REDIRECT_URI
    }
}
```

`daVinci.start()`, `continueNode.next()` and `daVinci.user()` are all suspend functions. `user?.accessToken()` returns a `Result<Token, OidcError>`; `user?.logout()` ends the session.

Several of these are top-level extensions rather than members, so an unresolved reference here usually means a missing explicit import rather than a wrong version. `minSdk` floor for DaVinci and OIDC is 28.

## iOS

Construction follows the same shape: a DaVinci client configured with a client id, a discovery endpoint, scopes and a redirect URI, driven through `start()` and `next()` over `ContinueNode` / `SuccessNode` / `FailureNode`. `FailureNode.cause` is non-optional. Give each client instance its own `account` string on `KeychainStorage<Token>` or two instances overwrite each other's tokens.

Unconfirmed here beyond that; see the caveat in `reference/collectors.md`.

## Token exchange, and what the client does with claims

A completed flow yields an authorization code that exchanges for a token set with no browser involved at any point. Where a client reads the user's profile from differs by moment, and getting this wrong produces a profile that is correct on sign-in and stale afterwards: the profile comes from `/userinfo` on sign-in and from the id token's claims on refresh. The access token is never decoded client-side.
