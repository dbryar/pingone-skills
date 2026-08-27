---
name: davinci
description: Author, debug and deploy PingOne DaVinci flows - the flow JSON graph model, node and connector shapes, variable contexts and template bindings, subflow contracts, branching, error handling, and the HTML/CSS/JS surface of hosted login screens. Use whenever a task involves a DaVinci flow, a flow export, a login journey, or diagnosing why a deployed flow behaves differently from how it reads. Assumes pingone:core for authentication and permissions.
---

# PingOne DaVinci flows

DaVinci flows fail quietly. Most of the findings below are cases where the flow applied cleanly, the API reported success, and the behaviour was wrong. That is the characteristic failure mode of this platform and it shapes how you should work in it: verify against a running flow, not against a schema.

Companion skills: `pingone:core` for tenant operations, `pingone:terraform` for deploying flows as code.

## Before anything else: which render mechanism

DaVinci flows render two structurally different ways, and almost every UI decision depends on which one you are in.

| | Redirect | Embedded |
| --- | --- | --- |
| What happens | Browser navigates to a PingOne-hosted page | A client library renders inside the requesting application's own page |
| Who owns the DOM | PingOne's widget | Your application |
| Branding mechanism | Flow `settings.css`, `customHTML`, `customScript` | Your application's own stylesheets |
| Triggered by | A flow policy bound to an OIDC client | A client SDK call |

Confirm which one you are targeting before planning any authoring work. Inferring it from context is how a branding mechanism gets built for the wrong surface.

The redirect page is itself a client-rendered widget, not server-rendered HTML. Every screen renders into one light-DOM container (`#widgetContainer`), with no iframe and no shadow DOM, and screen transitions happen over XHR without a page reload. Two consequences follow directly, and both are load-bearing:

- **Anything injected into `document.head` on the first screen persists across every later screen.** This is what lets one flow serve several brands or locales at runtime rather than deploying a flow per brand.
- **Your HTML is inserted as a fragment.** A `<!DOCTYPE html>`, `<html>`, `<head>` or `<body>` wrapper in `customHTML` is ignored entirely, and a `<style>` block inside it has no effect. CSS belongs in the flow's `settings.css` with `use_custom_css` set, which is served as a page-level stylesheet at `/{envId}/davinci/flows/{flowId}/css`.

## The graph model

A flow is a Cytoscape graph under `graphData.elements`, with `nodes[]` and `edges[]`. The canvas fields (`position`, `selected`, `locked`, `grabbable`, `pannable`, `classes`) are not where behaviour lives. `node.data` and `edge.data` are.

The fields to read first on any node:

| Field | Meaning |
| --- | --- |
| `data.id` | Local node ID. Edges reference this |
| `data.connectorId` | Connector *type*, e.g. `httpConnector`, `pingOneSSOConnector` |
| `data.connectionId` | The specific connector *instance* in this environment. Not portable between environments |
| `data.capabilityName` | The operation, e.g. `userLookup`, `customFunction`, `customHTMLTemplate` |
| `data.nodeType` | `CONNECTION` for a connector call, `EVAL` for a decision node |
| `data.properties` | Operation configuration. Most business logic lives here |

**Edges do not carry branch conditions.** An edge is a link between node IDs and nothing more. Branching is expressed in the properties of the EVAL or connector node upstream of it. See "Branching" below.

**Never draw an edge directly from one `CONNECTION` node to another.** The target screen renders and its controls silently stop responding, with no error anywhere. Put an EVAL between them. Every transition in a working graph has one.

**Node positions are not decoration.** Spacing encodes readability of the control flow: a connector and its EVAL bound tightly together, gates sharing a column so a cascade reads as an if/else-if chain, a default path set visibly further away than a gate outcome. If you generate flows programmatically, hold the layout explicitly and make a missing layout entry an error rather than defaulting a node to another node's position or to `(0,0)`. Nodes stacked at the origin still produce valid JSON that applies cleanly.

## Property encoding

Most node properties wrap their value:

```json
{ "nodeTitle": { "value": "Normalise authorization request" },
  "code": { "value": "module.exports = async ({ params }) => { ... }" } }
```

Some are raw strings instead. Any tooling that reads flow JSON has to handle both.

Two encodings that are easy to get wrong, both of which round-trip through the API unchanged while doing nothing:

- **`nodeTitle.value` must be a plain string.** Give it the Slate rich-text encoding used elsewhere for bound values and the title renders as raw JSON on the canvas.
- **A binding embedded inside a larger literal needs a `link`-typed Slate child.** A plain-text child containing a literal `{{...}}` string is never substituted. This bites when one field of a JSON object being saved is a bound value: the object saves, carrying the template text verbatim.

## Variables and bindings

Variables carry their context in the name: `<displayName>##SK##<context>`. `SK` is a legacy namespace marker from Singular Key, the product Ping acquired and renamed to DaVinci. It carries no current meaning.

| Context | Use |
| --- | --- |
| `company` | Environment-wide configuration. Stable across executions |
| `flowInstance` | Per-run state. Mutable during the flow |
| `user` | Scoped to the authenticating user |
| `flow` | Flow-scoped. This is the only context that accepts a `flow = {id}` block; setting it on any other context is rejected |

Both `company` and `flowInstance` variables can be object-typed, not just flat strings and numbers.

Binding forms:

```text
{{global.company.variables.<name>}}      company-scoped variable
{{global.variables.<name>}}              flow instance variable
{{global.parameters.<name>}}             flow input: trigger parameters, and subflow inputs
{{local.<nodeId>.payload.output.<field>}} output of a named upstream node
```

**A `saveValue` entry targeting a pre-declared `flowInstance` variable needs both `name` and `variableId`.** With `name` alone, the node round-trips through the API unchanged, Studio shows its variable selector unset, and the write never happens. The target variable sits at its declared default forever and nothing reports an error.

**The authorisation request reaches the flow as `auth_mode`, not `prompt`.** PingOne derives it. A silent authentication request is `{{global.parameters.authorizationRequest.auth_mode}}` equal to `"silent"`. The string `prompt` does not appear in flows that handle this correctly.

`{{global.parameters.application}}` and `{{global.parameters.authorizationRequest.client_id}}` give you the calling client. `client_id` is a PingOne application UUID, so if you need it as a key for brand or configuration lookup you need an explicit mapping; the UUID is not meaningful on its own.

## Subflows

**A subflow does not see the caller's `flowInstance` variables.** Its declared input is reached as `{{global.parameters.<name>}}`, named for the property in its own `input_schema`.

Binding a subflow to a caller variable is the most expensive error on this list, because it fails silently in the worst possible way. The comparison does not error. It compares against nothing, takes the mismatch branch every time, and the flow looks like it is working correctly and simply never taking the matching path.

**A subflow input is passed as its own top-level property on the calling node**, named for the subflow's `input_schema` property. It is not a nested `inputSchema` list. Getting this wrong fails the subflow call outright, which at least is loud.

`subFlowVersionId: -1` resolves to the latest *deployed* version, so a subflow must be deployed, not merely applied, before a caller can reach it.

A subflow returns through `httpConnector` / `createSuccessResponse`. A main flow must not. See "Terminals" below.

## Branching

- **An EVAL names its branches by the target node's own ID**, with `value: "anyTriggersFalse"` marking the failure branch. There is no edge-level label anywhere.
- **A `functionsConnector` node that throws is a real branching signal**, routed through the following EVAL's `anyTriggersFalse` branch.
- **`AEqualsB` branches natively** through a following EVAL keyed the same way, so a computed condition does not have to throw in order to produce a graph branch.
- **Rewiring a branch leaves the old branch key behind.** Studio does not clean up `properties` when an edge is repointed, so an EVAL's `properties` is not a reliable description of its branches. Treat "every EVAL branch key corresponds to a real outgoing edge" as a lint rule over generated and exported flows alike.

### Teleports

`nodeConnector`'s `goToNode` and `startNode` pair transfers control by reference rather than by edge:

- `goToNode` carries only `nodeInstanceId` and has **no outgoing edge**.
- `startNode` is a `type: "trigger"` named re-entry point and **needs no inbound edge to be reachable**.

This is the right tool for a shared destination reached from several places, and for a failure branch that must reach a specific screen rather than falling back to the last-rendered one. Using it means a shared "Success", "Error" or "No session" node can exist exactly once with no long edges dragged across the canvas to it. Sharing a destination and sharing a jump node are separate decisions; one `goToNode` can serve several nearby callers.

## Forms, and the vocabulary that actually renders

A form is a PingOne object with its own ID, provisioned separately (see `pingone:terraform`'s `pingone_form`). A flow shows one through `pingOneFormsConnector`'s `showForm` capability, and the node carries only a reference:

```jsonc
{
  "capabilityName": "showForm",
  "connectorId": "pingOneFormsConnector",
  "capabilityClass": "render",
  "properties": {
    "form":     { "value": "<form id>" },
    "formData": { "value": [ { "key": "user.username", "value": "" } ] }
  },
  "outcomes": [ { "result": "submit", "label": "<the form's button text>" } ]
}
```

- **One `submit` outcome per form.** Branch after the form on what was submitted, not by the form offering several exits.
- **`formData` keys must exist on the referenced form.** A key that does not match binds to nothing, and nothing in the flow JSON says so.

**A form is the only screen a native or SDK-driven client can render.** A `customHTMLTemplate` screen is an HTML document with no field structure, and the client SDKs refuse it: DaVinci marks an SDK-renderable response with `isResponseCompatibleWithMobileAndWebSdks`, and a `customHTMLTemplate` response does not carry the flag. Check for that flag rather than inspecting generated JSON, because it is the platform's own verdict.

**Three vocabularies, and the narrowest one binds.** The form builder authors 24 field types. The JavaScript client SDK implements collectors for 21. The Android SDK implements 14: `TEXT`, `PASSWORD`, `PASSWORD_VERIFY`, `SUBMIT_BUTTON`, `FLOW_BUTTON`, `FLOW_LINK`, `LABEL`, `COMBOBOX`, `CHECKBOX`, `DROPDOWN`, `RADIO`, `DEVICE_REGISTRATION`, `DEVICE_AUTHENTICATION`, `PHONE_NUMBER`. The seven it lacks are `SOCIAL_LOGIN_BUTTON`, `PROTECT`, `POLLING`, `AGREEMENT`, `IMAGE`, `REGISTER` and `SINGLE_CHECKBOX`.

**The failure is silent and looks like success.** A field the SDK has no collector for is omitted from the node's collector list rather than raising, so the screen renders looking complete, cannot be submitted, and says nothing about why. A browser-based review of the same form passes, because the web SDK's vocabulary is wider. Author every shared form to the narrowest consuming SDK and check it at authoring time; there is nothing to catch it later.

Read a client SDK's vocabulary from its implementation rather than its published types. The JavaScript SDK's own `StandardField` type union declares `BUTTON` and `SINGLE_SELECT`, and neither has a case in the code that maps fields to collectors.

## Terminals

| Situation | Correct terminal |
| --- | --- |
| Main flow, success | `pingOneAuthenticationConnector` / `returnSuccessResponseRedirect` |
| Main flow, refused silent authentication | `pingOneAuthenticationConnector` / `returnErrorResponseRedirect` with `errorCode: "login_required"`, `customErrorFlag: true` |
| Subflow, success | `httpConnector` / `createSuccessResponse` |

A main flow ending on `createSuccessResponse` returns plain JSON and never establishes a PingOne authentication session or issues an authorization code. It looks like it completed.

## Error handling, and a trap worth an entire debugging session

**A `customErrorMessage` node displays by returning control to the last-rendered screen.** It has no screen of its own. That gives it two properties:

1. It is a legitimate dead end with no outgoing edge. DaVinci's runtime returns to the previous screen and populates its native error display on its own. You do not need a router back to the form, and adding one is what breaks the retry.
2. **It is fatal if reached before any screen has rendered.** There is nowhere to return to, and the whole authorisation request dies.

Treat "no `customErrorMessage` node is reachable before a screen has rendered" as a precondition on any flow that calls a subflow or checks a session before its first screen. A branch in that position must rejoin the routing decision through a waypoint instead of terminating.

**When a flow dies, PingOne reports `login_required` to the relying party, with the same generic `error_description` a legitimately refused silent authentication produces.** Two completely different causes produce byte-identical text at the relying party. Never conclude anything from that text. Read DaVinci's own flow log, where a dead flow shows as `id: "subflowFailed"` with an `err.code`.

### The native error display

Error text reaches the screen through empty `data-skcomponent="skerror"` and `data-skcomponent="skerrormessage"` elements that DaVinci's widget populates itself when the loop returns to that screen with an active error. No binding and no script is needed. Hand-built bindings like `{{local.errorScreen.payload.output.errorDescription}}` into a custom div are not how this works.

The content comes from `errorConnector`'s `errorMessage` property, not `errorDescription`.

**The error re-render patches only the error region.** The screen's inline `customScript` does not re-run. Anything the script was doing, such as swapping labels for a brand or locale, is not reapplied. The tell is that other script-driven changes correctly do *not* revert on that same re-render, which only makes sense if the DOM patch is partial.

## The HTML/CSS/JS surface

- **`customHTML` is a fragment.** Document wrapper tags are ignored; inline `<style>` does nothing. Use `settings.css` with `use_custom_css`.
- **Literal `onclick="..."` attributes are stripped** by DaVinci's server-side sanitisation before the markup is served. Give the element a stable `id` and attach the handler from `customScript`. Markup injected client-side by `customScript` via `innerHTML` survives, because sanitisation has already run by then.
- **A form on a `customHTMLTemplate` screen submits to** `POST /davinci/connections/<connectionId>/capabilities/customHTMLTemplate`.
- Your content sits several wrapper `div`s deep inside the widget's own markup. A `body { display: flex; justify-content: center }` reaches only `body`'s direct child, which is not your card. Elements must lay themselves out (`margin: 0 auto`, `max-width`, `box-sizing: border-box`) rather than depending on an ancestor's layout context that does not reach them.
- Watch stylesheet concatenation order. A `:root` fallback in a shared base stylesheet that is concatenated *after* a brand's token file silently overrides every brand.
- `settings.css` in real production flows runs to hundreds of kilobytes. Do not assume it is small.

## Session checking

- **`checkSession` needs `checkSessionAuthenticator` set, and a matching `trigger.configuration.<authenticator>` window on the flow policy.** With either missing it reports "no session" for every request while the Management API plainly shows a live session for that user. Nothing errors. The two values live in different places, nothing validates one against the other, and a mismatch produces a clean, plausible, wrong answer. See `pingone:terraform` for the flow policy side.
- `anyAuthenticationMethod` is usually the right authenticator, since the capability has no notion of relative strength and naming a specific one refuses sessions established by any other.
- **Unit trap.** `checkSession`'s `authenticationMethodLastUsedIn` is in **minutes**. `cookieConnector`'s `cookieExpiresInSeconds` is in **seconds**. The same number in the same flow means two different things, and getting it wrong is silent: nothing errors, the boundary is just wrong by a factor of sixty.
- `authenticationMethodLastUsedIn` measures time since last *authentication*, not last *use*. A session passing through without re-authenticating does not reset it, so it cannot express a "last active within N minutes" rule at all.
- A `cookieConnector` cookie is an **opaque handle**, not a self-contained claim set. Its value decodes to a connector instance ID and a DaVinci-side reference, so claims are not browser-inspectable and can only be confirmed by reading them back in-flow.
- Cookies intended to survive a cross-site top-level navigation cannot be `SameSite=Strict`, which is not sent on that navigation at all. PingOne's own session cookie on the same domain is `SameSite=None`, with a variant for browsers that mishandle `None`.

## Connectors

- **A connector instance is created the moment a node using it is first authored in Studio.** Anything you author by hand exists before your infrastructure code hears about it. Check for an existing instance before creating one, or you get a duplicate rather than adoption.
- **PingOne-category connectors need a credential to call back into their own environment**, supplied by the auto-provisioned "PingOne DaVinci Connection" application. An environment created with the `DAVINCI_MINIMAL` service tag does not have one. Studio shows this as "This connector is missing required configuration"; the API shows it as an opaque `unexpectedError` on every capability call.
- **The connector instance property key for the environment is `envId`, not `environmentId`**, in a flat shape with no wrapper. The write API does not validate property keys against the connector's schema, so a wrong key is stored and echoed back correctly on every `get`. The configuration reads as confirmed-correct and has never worked. If a PingOne-category connector fails opaquely, verify the property *keys* before anything else.

## Working method

The findings above were not derived from documentation. They came from deploying flows and watching them run. Adopt the same method:

1. **Ground new node shapes against a real working export** before authoring them. A capability invented from a plausible reading of the schema will apply cleanly and do nothing.
2. **Verify against a real rendered page**, not against the API accepting the deploy. A green deploy proves nothing about behaviour, branding, or whether a button responds to a click.
3. **When something opaque fails, get evidence rather than reasoning further.** Inspect the live node through the API, dump the page's state through the browser's debug protocol, read the DaVinci flow log. Opaque connector and session failures here have a history of surviving two rounds of schema-based reasoning and yielding immediately to one direct observation.
4. **Keep a control case.** "Silent authentication does not work" means nothing until the interactive request through the same flow, same session, same client is shown to work.

## Anti-pattern: the Studio export as source of truth

Exported flow JSON commonly runs to hundreds of kilobytes with HTML, CSS and orchestration all inside one file. Kept as the authoring artefact, this has three costs: only one person can edit a flow at a time, since the next export overwrites the previous wholesale; a code review of a re-exported blob shows every line changed regardless of what was actually done; and orchestration, markup and design cannot be worked on in parallel by the people suited to each.

Author flows from separated sources and generate the JSON. Studio is then a review and inspection tool, which is what it is good at.

## Reference

- [`reference/flow-json.md`](reference/flow-json.md) - top-level keys, node and edge shapes, settings, parsing rules.
- [`reference/connectors.md`](reference/connectors.md) - connector and capability catalogue with the shapes confirmed in use.

## Correcting this skill

This file is expected to be wrong eventually, and DaVinci changes more often than the rest of the platform.

When you find an instruction here that does not match observed behaviour, or you confirm a behaviour this file does not cover, run `/pingone:learn` and describe the finding. Do not silently work around a wrong instruction, and do not edit this file from memory.

Hold a high bar here specifically. Most of this file records something that *looked* correct and was not, so a finding is worth writing down only once it has been observed on a running flow, ideally with a control case that isolates it. A plausible reading of the schema is exactly the kind of material this file exists to correct, and adding more of it makes the file worse.
