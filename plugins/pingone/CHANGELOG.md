# Changelog

## 0.2.1 - 2026-09-11

- `pingone:core` - "Observing flow executions". Every DaVinci execution is readable event by event from `GET /environments/{envId}/flows/{flowId}/interactions` and its `/events`, or through the PingOne remote MCP server. A subflow logs under its own `interactionId`, linked to the parent by `transactionId`; asking for it under the parent's returns `404`.
- `pingone:core` - `reference/execution-logs.md`. The PingOne remote MCP server: the regional host takes the full root domain, sign-in goes through the target environment's own authorization server, and the server can connect with an empty tool list and no error until a tenant-side setting is enabled. The event log's shape.
- `pingone:core` - `pingcli pingone api` examples corrected for 1.4.0: method as `--http-method`, inline JSON as `--data-raw`. A nonexistent path returns `403` naming the Authorization header; added to the error vocabulary.
- `pingone:davinci` - "read the DaVinci flow log" now points at the core recipe.

## 0.2.0 - 2026-09-09

- `pingone:sdk` - new skill. The client side of an embedded DaVinci flow: the Ping Orchestration SDKs for web, Android and iOS, the collector vocabulary each one implements, the PingOne Form field types that produce those collectors, and applying a brand and locale to a form the application draws itself.
  - The three-vocabulary rule. PingOne's form builder accepts 24 field types and each SDK collects a subset. An unsupported field does not error, it is absent from the collector list, so the screen renders looking complete and cannot be submitted. Author to the narrowest client that will render the form, and enforce it at generation time. Which fields each client collects is Ping's maintained compatibility matrix, linked rather than transcribed.
  - Three constraints that are not field-level and so never present as a missing collector: `SKPolling` cannot be processed by any DaVinci client and rules out Magic Link authentication; images embedded in a Custom HTML Template are not processed; and `SLATE_TEXTBLOB` and `ERROR_DISPLAY` are in PingOne's stock sign-on form and in no SDK source.
  - `Image` and `New Password` are web-only. A reset or registration form built on `New Password` completes in a browser and cannot be completed in an app.
  - Read a vocabulary from the SDK's implementation, never from its published type union. `BUTTON` and `SINGLE_SELECT` are declared by the JavaScript client and implemented by neither.
  - `FlowCollector` and `SubmitCollector` share `category: "ActionCollector"` and reach DaVinci by different calls (`flow({action})()` against `next()`). A dispatch keyed on category alone submits a named exit as a plain form submission, and the failure is silent because the re-rendered screen is identical either way.
  - An `error` node still carries collectors, which is what every retry lane depends on. An error is a field on every collector rather than a collector of its own; `CheckboxCollector` and `ErrorDisplayCollector` are 1.2.0 names that do not exist in 2.1.1.
  - Branding and localisation: resolve the brand once per client construction, make the partner input a schema-typed token file rather than a stylesheet so Compose and SwiftUI can consume it unchanged, reject per key and fall back per key, and treat a brand key with nothing published for it as the expected case.
- `pingone:davinci` - the client-side half of "Driving the embedded surface with a Ping SDK" moves to `pingone:sdk`. The two flow-authoring facts stay: the `form.components.fields` assertion, and the dotted-in/nested-out form field key round trip.
- `pingone:davinci` - the per-client field vocabularies added in 0.1.6 become a pointer. The constraint on how a form may be authored stays; the list of what each client collects moves to `pingone:sdk`, which links Ping's matrix. The Android list was 1.2.0-era and already stale at 2.1.0.
- `pingone:terraform` - `pingone_form`'s vocabulary note points at both skills: `pingone:davinci` for why the constraint binds form authoring, `pingone:sdk` for which fields each client collects.
- `/pingone:learn` - routes findings to `sdk` as a fourth destination, and records an SDK version alongside pingcli and provider versions.
## 0.1.6 - 2026-09-01

- `pingone:terraform` - `pingone_form`. A form is a first-class PingOne object with its own lifecycle, not flow content, so a `showForm` node's form reference needs the same token substitution as `subFlowId` and fails the same silent way. Field positions, the `type` enum, directory-attribute field keys, Slate labels with their own language bundle, and the stock forms a new environment already has.
- `pingone:davinci` - three field vocabularies disagree. The form builder authors 24 types, the JavaScript SDK collects 21, the Android SDK collects 14. A field outside the consuming SDK's set is dropped from the collector list rather than raising, so the screen renders looking complete and cannot be submitted.

## 0.1.5 - 2026-09-01

- `pingone:davinci` - a screen is painted once. Re-entering a `customHTMLTemplate` node does not replace the DOM or re-run `customScript`, so the error re-render's partial patch is one case of a general rule. `disabled` on an `skbutton` belongs to the widget and does not survive a submission round trip; gate with a class and a captured `click` listener instead.
- `pingone:davinci` - a node's description is rendered inside the node, so its length is the node's height on the canvas. Company variables (`{{global.company.variables.<name>}}`) resolve inside a subflow and inside a screen's `customScript`, unlike `{{global.variables.<name>}}`.

## 0.1.4 - 2026-09-01

- `pingone:davinci` - connector instances. Studio creates a duplicate the first time a connector is opened on a canvas, whether or not a managed instance already exists, so the check has to be periodic rather than once. Default instance names read from the connector catalogue, since the id does not predict the name.
- `pingone:terraform` - reconciling a duplicate instance, and telling console-created instances from Terraform-created ones by `customerId`. A PingOne collection endpoint returning 500 means one unserialisable row, not an outage.

## 0.1.3 - 2026-09-01

- `pingone:davinci` - outcome routing. A node's declared outcome routes nowhere unless the edge leaving by it carries `multi_value_source_id`, and each outcome needs its own evaluator rather than a shared one. Both faults present as the connector logging success, no downstream node logging at all, and the client seeing `400 requestTimedOut`.

## 0.1.2 - 2026-08-27

- `pingone:davinci` - a node with no inbound edge is an entry point, not a dead node. An unwired terminal fires on entry and returns before anything runs, which is valid JSON, applies cleanly, and reports nothing.

## 0.1.1 - 2026-08-25

- `pingone:davinci` - custom claims on `returnSuccessResponseRedirect`. `accessTokenClaims` and `idTokenClaims` are independent lists; claim row shape and which fields are cosmetic; pasted claim blocks keeping the source flow's node IDs, which nothing validates; branch reachability of the node a terminal sources claims from; why not to override `sub`; guidance on which token identity claims belong in, and when putting them in the access token is a reasonable constraint.

## 0.1.0 - 2026-08-14

Initial release.

- `pingone:core` - tenant operations. Service gating presenting as permission errors, pingcli authentication, the role scope model, environment creation, field validation limits, MFA device pairing, headless authentication via `pi.flow`, error vocabulary, environment limits.
- `pingone:davinci` - flow authoring. Render mechanisms, the graph model, property encoding, variable contexts and bindings, subflow contracts, branching and teleports, terminals, error handling, the hosted page surface, session checking, connectors.
- `pingone:terraform` - both as code. The plan as a drift detector, provider selection, `pingone_davinci_flow` shapes, subflow substitution, deploy and flow policy, apply failure modes, resource notes.
- `/pingone:learn` - records a confirmed finding into the skills and the journal.
