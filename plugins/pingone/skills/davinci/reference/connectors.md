# Connector and capability reference

Confirmed in use rather than transcribed from a catalogue. A capability absent here is not necessarily unavailable; it is only not confirmed.

## Two categories

**PingOne-category connectors** (`pingOneSSOConnector`, `pingOneAuthenticationConnector`, `pingOneMfaConnector`, `pingOneRiskConnector`, `notificationsConnector`) call back into their own PingOne environment and need a credential to do it. That credential comes from the auto-provisioned "PingOne DaVinci Connection" application, or from one you provision yourself if the environment was created with the `DAVINCI_MINIMAL` tag.

Their `properties` shape is flat, and **the environment key is `envId`, not `environmentId`**:

```json
{ "envId": "<environment-id>", "clientId": "...", "clientSecret": "...", "region": "..." }
```

The connector instance write API does not validate property keys against the connector's schema. A wrong key is accepted, stored, and echoed back on every subsequent read, so the instance appears correctly configured and every capability call fails with an opaque `unexpectedError`. Verify keys, not just values.

**Core-category connectors** (`httpConnector`, `functionsConnector`, `variablesConnector`, `codeSnippetConnector`, `errorConnector`, `nodeConnector`, `flowConnector`, `cookieConnector`, `userPolicyConnector`, `annotationConnector`) need no credential. Confirmed empirically for `httpConnector`; holds for the family.

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
