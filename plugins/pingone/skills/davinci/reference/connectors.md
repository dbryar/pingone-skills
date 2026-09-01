# Connector and capability reference

Capabilities here are confirmed in use rather than transcribed from a catalogue: one absent is not necessarily unavailable, only not confirmed. The name and category table below is the exception, and is read straight from the catalogue endpoint.

## Two categories

**PingOne-category connectors** (`pingOneSSOConnector`, `pingOneAuthenticationConnector`, `pingOneMfaConnector`, `pingOneRiskConnector`, `notificationsConnector`) call back into their own PingOne environment and need a credential to do it. That credential comes from the auto-provisioned "PingOne DaVinci Connection" application, or from one you provision yourself if the environment was created with the `DAVINCI_MINIMAL` tag.

Their `properties` shape is flat, and **the environment key is `envId`, not `environmentId`**:

```json
{ "envId": "<environment-id>", "clientId": "...", "clientSecret": "...", "region": "..." }
```

The connector instance write API does not validate property keys against the connector's schema. A wrong key is accepted, stored, and echoed back on every subsequent read, so the instance appears correctly configured and every capability call fails with an opaque `unexpectedError`. Verify keys, not just values.

**Core-category connectors** (`httpConnector`, `functionsConnector`, `variablesConnector`, `codeSnippetConnector`, `errorConnector`, `nodeConnector`, `flowConnector`, `cookieConnector`, `userPolicyConnector`, `annotationConnector`) need no credential. Confirmed empirically for `httpConnector`; holds for the family.

## Default instance names

The name each connector's instance takes by default, and the name Studio gives the one it creates for you. Read from `GET /v1/environments/{envId}/connectors`, which returns `name` and `metadata.type` per connector and is the authority for any connector not listed here. `metadata.type` is the same field that decides which `properties` shape the instance needs, so one read answers both questions.

Name a managed instance anything else and it becomes indistinguishable, in a list showing both, from a duplicate Studio made.

| Connector | Default name | `metadata.type` |
| --- | --- | --- |
| `pingOneSSOConnector` | PingOne | `ping` |
| `pingOneAuthenticationConnector` | PingOne Authentication | `ping` |
| `pingOneMfaConnector` | PingOne MFA | `ping` |
| `pingOneRiskConnector` | PingOne Protect | `ping` |
| `notificationsConnector` | PingOne Notifications | `ping` |
| `pingOneFormsConnector` | Form | `core` |
| `httpConnector` | HTTP | `core` |
| `functionsConnector` | Functions | `core` |
| `variablesConnector` | Variable | `core` |
| `codeSnippetConnector` | Code Snippet | `core` |
| `errorConnector` | Error Message | `core` |
| `nodeConnector` | Teleport | `core` |
| `flowConnector` | Flow Conductor | `core` |
| `cookieConnector` | Cookie | `core` |
| `userPolicyConnector` | User Policy | `core` |
| `analyticsConnector` | Flow Analytics | `core` |
| `annotationConnector` | Annotation | `core` |
| `challengeConnector` | Challenge | `core` |
| `skUserPool` | User Pool | `core` |

Four of these are worth reading twice, because the connector id does not predict the name and a plausible guess is wrong: `nodeConnector` is **Teleport**, `flowConnector` is **Flow Conductor**, `errorConnector` is **Error Message**, and `pingOneFormsConnector` is **Form** — `core`, despite the PingOne prefix, because its theming is applied PingOne-side at render rather than fetched on a credential of its own.

If a build generates flow JSON, derive the name from the connector id through this table rather than spelling it at each node. Nothing at runtime binds on it — a node binds to its instance by `connectionId` — so a wrong name never fails, and left as free text per node it drifts without resistance.

## Capabilities in use

| Connector | Capability | Notes |
| --- | --- | --- |
| `httpConnector` | `customHTMLTemplate` | Renders a screen. Content is a fragment. Form posts to `/davinci/connections/<connectionId>/capabilities/customHTMLTemplate` |
| `httpConnector` | `createSuccessResponse` | A **subflow's** return. Wrong as a main flow's terminal |
| `pingOneAuthenticationConnector` | `returnSuccessResponseRedirect` | A main flow's terminal. Establishes the session and issues the authorization code. Carries `properties.idleTimeout`, and the two independent claim lists `accessTokenClaims` and `idTokenClaims` |
| `pingOneAuthenticationConnector` | `returnErrorResponseRedirect` | Refuses the request to the relying party. `errorCode`, `customErrorFlag: true` |
| `pingOneAuthenticationConnector` | `checkSession` | Needs `checkSessionAuthenticator` plus a matching flow policy `trigger.configuration` window. `authenticationMethodLastUsedIn` is in **minutes** |
| `pingOneSSOConnector` | `userLookup` | Supports `scimFilter` with `useCustomSCIMFilter`. A population-scoped filter combines an identifier attribute with `population.id eq "..."` |
| `pingOneSSOConnector` | `checkPassword` | Credential verification |
| `pingOneSSOConnector` | `readUser` | |
| `variablesConnector` | `saveValue` | Needs **both** `name` and `variableId` when targeting a pre-declared variable |
| `functionsConnector` | `customFunction` | Throwing is a real branching signal, routed through the following EVAL's `anyTriggersFalse` branch |
| `functionsConnector` | `AEqualsB` | Branches natively through a following EVAL keyed `anyTriggersFalse`, without needing to throw |
| `errorConnector` | `customErrorMessage` | Displays by returning to the last-rendered screen. A correct dead end with no outgoing edge, and **fatal if reached before any screen has rendered** |
| `nodeConnector` | `goToNode` | Carries only `nodeInstanceId`. No outgoing edge |
| `nodeConnector` | `startNode` | `type: "trigger"` re-entry point. Needs no inbound edge |
| `flowConnector` | `startSubFlow` / `startUiSubFlow` | Target named in `properties.subFlowId`. `subFlowVersionId: -1` means latest **deployed** version |
| `cookieConnector` | `checkSessionCookieWithoutUser` | Reads a named cookie. `enforceFlowIdMatch: false` lets one flow read a cookie written by another |
| `cookieConnector` | (write capabilities) | `hmacSigningKey` is a connector-instance property. `cookieExpiresInSeconds` is in **seconds** |

## Cookie connector properties

The HMAC signing key lives on the connector instance as `hmacSigningKey`, not as a flow variable. Source it from outside your infrastructure code and treat it as a credential: a Terraform-managed random value sits inside the blast radius of the delete-and-recreate remediation these instances sometimes need, and regenerating it silently invalidates every cookie previously issued.

## Native error display

`errorConnector`'s `errorMessage` property is what populates `data-skcomponent="skerror"` on the returned screen. Not `errorDescription`. Leave the `skerror` and `skerrormessage` elements empty in your markup; the widget fills them.

## Choosing a risk source

`pingOneRiskConnector` exists. If a separate risk product supplies the sign-in decision, do not provision it: an unused connector instance is one more thing that must be kept configured, and its presence implies a decision that was not made.

## `pingOneFormsConnector` — Form

Renders a **PingOne form** as a screen, using the environment's own branding and theme. It is a `core`-category connector: `properties = null`, no environment credential, despite the name. Check `metadata.type` on `GET /v1/environments/{envId}/connectors/{connectorId}` before provisioning any connector — `core` needs no credential block, `ping` usually does.

This is the connector an embedded SDK client can render; a `customHTMLTemplate` screen is an HTML document with no field structure and produces no collectors.

### `showForm`

```json
{
  "capabilityName": "showForm",
  "connectorId": "pingOneFormsConnector",
  "capabilityClass": "render",
  "properties": {
    "form":     { "value": "<form id>" },
    "formData": { "value": [ { "key": "user.username", "value": "" },
                             { "key": "user.password", "value": "" } ] },
    "nodeTitle": { "value": "Sign On Form" },
    "dynamicText": { "value": [] }
  },
  "outcomes": [ { "id": "<edge id>", "label": "Log in", "result": "submit" } ]
}
```

- **The form is a PingOne resource, not flow content.** The node holds only its ID, which is environment-specific — generate it in rather than committing a literal.
- **`formData` keys must exist as field keys on the referenced form.** A key that does not match binds to nothing and the flow JSON does not say so.
- **One `submit` outcome per form**, claimed by its edge's `multi_value_source_id`. Branch after the form on what was submitted, not by the form offering several exits.
- **The submitted value comes back nested.** A field keyed `user.username` returns as `output.formData.user.username`, though it is authored and submitted dotted.
