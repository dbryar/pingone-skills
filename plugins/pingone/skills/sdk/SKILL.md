---
name: sdk
description: Build the client side of an embedded DaVinci flow with a Ping Orchestration SDK - the web, Android and iOS clients, the collector vocabulary each one can actually render, the PingOne Form field types that produce those collectors, and applying a brand and locale to a login form the application draws itself. Use whenever a task involves @forgerock/davinci-client, com.pingidentity.sdks:davinci, ping-ios-sdk, a collector, a PingOne Form field, or a login screen rendered inside an application rather than on a PingOne-hosted page. Assumes pingone:core for authentication and pingone:davinci for what the flow does.
---

# Ping Orchestration SDKs and PingOne Forms

**The SDK renders nothing.** It interprets a server-driven flow and hands the application a list of collectors. Every control on the screen is drawn by code you write, in your own UI toolkit, styled by your own stylesheet or theme. Ping's own documentation puts it as "you are in charge of the experience your end users have", and almost every mistake in this file follows from forgetting it.

Companion skills: `pingone:core` for authentication and permissions, `pingone:davinci` for the flow the SDK is driving, `pingone:terraform` for deploying the forms and flows as code.

## Before anything else: are you actually on the embedded surface

`pingone:davinci` opens with the redirect-versus-embedded distinction, and this skill is only the second column. If the browser navigates to a PingOne-hosted page, none of this applies: that page is a PingOne widget branded through the flow's own `settings.css` and `customHTML`, and no SDK is involved. Confirm which surface you are on before planning any work here.

## Four things decide whether a flow starts at all, and three fail silently

- **The OIDC client needs a CORS allow-list.** The SDK calls `/as/authorize` with `response_mode=pi.flow` via `fetch`, from the host page's origin, so an origin absent from the application's CORS settings cannot start a flow. A client created for a redirect journey has no such setting and needs one added before it can serve an embedded one.
- **It only works in a browser.** Driving the JavaScript SDK from a server-side runtime fails identically against a known-good flow and a broken one, so a failure there says nothing about the flow. Debug in a real browser. A headless probe of `/as/authorize` reads the raw response and tells you less than it appears to.
- **Collectors are built from `form.components.fields` on the response, and from nothing else.** Both the JavaScript and Android clients read that one path. A response carrying its fields anywhere else produces zero collectors and an empty screen. `Form.parse` on Android reads `json["form"]["components"]["fields"]`, the same path.
- **`isResponseCompatibleWithMobileAndWebSdks` is not a renderability gate.** It appears on the completed response and never on a screen response, and no implementation file in the JavaScript client reads it: it occurs once, in a type declaration. Assert the field path above instead.

## The three vocabularies, and why the narrowest one wins

This is the rule that decides how a form may be authored, and it has no equivalent in Ping's published guidance.

| Vocabulary | Size | What it constrains |
| --- | --- | --- |
| PingOne form field types | 24 | What the form builder and the Terraform provider will accept |
| JavaScript SDK collectors | fewer | What a browser client can draw |
| Android SDK collectors | fewer still | What a native client can draw |

PingOne's own `type` enum for a form field carries 24 values, and the form builder accepts every one of them regardless of what will render it. That is the first place a form can go wrong.

**Which fields each SDK collects is published and maintained: <https://developer.pingidentity.com/orchsdks/davinci/compatibility.html>.** It gives the collector class and the SDK version it arrived in, per field, for all three platforms at once. Use it as the authority rather than any transcription: the matrix gains rows every minor release, and a stale vocabulary causes exactly the failure below.

**An unsupported field does not error. It is simply absent from the collector list.** The screen renders looking complete and cannot be submitted. That is why the vocabulary is a generation-time constraint on how a form is authored, not something to catch at runtime.

**Where a form is rendered by more than one client, author it to the narrowest one.** A form authored to the web vocabulary works in a browser and collects nothing in the app, and it passes a browser-based review on the way through. That is the worst failure shape available here.

**Three constraints do not appear in the matrix as missing collectors, because they are not field-level.** `SKPolling` components cannot be processed by any DaVinci client and must not appear in a flow at all, which rules out Magic Link authentication; images embedded in a Custom HTML Template are not processed either; and `SLATE_TEXTBLOB` and `ERROR_DISPLAY` are in PingOne's own stock sign-on form and in no SDK source, which is why a stock form cannot be adopted as authored. [`reference/forms.md`](reference/forms.md) has the detail, including which of those the matrix now half-answers.

**Where the matrix is silent, read the SDK's implementation and not its published types.** `BUTTON` and `SINGLE_SELECT` are declared in the JavaScript SDK's `StandardField` type union and have no case in its switch statement, so neither collects anything despite appearing in the types. A type union is a partial and partly aspirational view; the switch is the authority, and a captured response beats both.

The collector-side shapes are in [`reference/collectors.md`](reference/collectors.md) and the form-authoring side in [`reference/forms.md`](reference/forms.md).

## The form is the contract

A collector's type, key, label, options and validation are all decided in the PingOne Form, not in the client. Debugging a missing, mistyped or unlabelled collector means reading the form definition, which in a Terraform-managed estate is a `pingone_form` resource rather than something to inspect in the console.

- **A field's `key` is what the flow branches on, not its label.** Renaming a `FLOW_BUTTON`'s key is a flow change, not a copy change, and the flow will branch on the old key until it is updated too.
- **A form field key round-trips asymmetrically.** A field keyed `user.username` is authored dotted, submitted dotted by the SDK (`formData: {"user.username": ...}`), and returned to the flow nested (`output.formData.user.username`). Bind downstream nodes against the nested path. A dotted binding resolves to nothing, silently. This is a flow-authoring consequence and `pingone:davinci` carries it too.
- **A device selection field declares method types, not devices.** `DEVICE_AUTHENTICATION` and `DEVICE_REGISTRATION` fields carry an `options` list of method types (`SMS`, `EMAIL`, `TOTP`, `FIDO2`); PingOne fills in the user's actual devices at runtime. The field set is fixed and the collector's options are not what the form author wrote.

## Driving the flow

The loop is the same on all three platforms: start, read the node's status, draw its collectors, write values back into the collectors, advance. Minimal client construction and the full lifecycle API per platform are in [`reference/clients.md`](reference/clients.md).

Node status is `start`, `continue`, `success` or `error`. **An `error` node still carries collectors**, and re-rendering them is the mechanism an OTP retry lane depends on: an error connector reached from a form returns control to that form, the status goes to `error`, and the same collectors come back. Treating `error` as terminal breaks every retry path.

### `FlowCollector` is not `SubmitCollector`, and they reach DaVinci by different calls

The SDK submits a form with `next()` and claims a named exit with `flow({ action })()`. Both collectors present as a button and both carry `category: "ActionCollector"`, so a dispatch keyed on the category alone renders them identically **and submits them identically**, which is right for the first and wrong for the second.

Confirmed live, 02-09-2026: "Send a new code" and "Try another way" were submitting as plain form submissions, so the resend outcome was never claimed and no new code was ever requested. **Nothing errored.** A submitted wrong or empty code re-renders the same screen, which is indistinguishable in the DOM from a resend re-rendering it, so every assertion after the button press still held.

Key the dispatch on `type` first and `category` only as a fallback, and surface the two to the host application as two callbacks rather than one. A host handed a single callback has to re-derive the distinction, which is the derivation that was got wrong.

### Never drop a collector

Classify every collector into exactly one role, and give the unclassifiable ones a text input rather than nothing. **A dropped collector is invisible in the DOM and therefore invisible to any test that reads the DOM**, so a form that silently omits a field passes its own test suite.

The auto-advancing collectors (`ProtectCollector`, `PollingCollector`, `FidoRegistrationCollector`, `FidoAuthenticationCollector`) genuinely draw nothing, and that is not the same thing. Report them to the host as deferred so that "not rendered" stays distinguishable from "not seen".

### An error is a field, not a collector

In `@forgerock/davinci-client` 2.1.1, `error: string | null` is a property on *every* collector. There is no error collector: render a field's error beside the field it belongs to. `CheckboxCollector` and `ErrorDisplayCollector` are 1.2.0 names and do not occur in 2.1.1; a checkbox is a `BooleanCollector` whose `output` carries an `appearance`.

## Branding and localisation

Ping's SDK guidance says nothing about this, because from the SDK's point of view there is nothing to say: your application owns the DOM, so your application owns the styling. That is true and unhelpful, because the actual problem is serving several brands and locales from one flow. What follows is the shape that works.

**Do not fork the flow per brand.** One flow, one form, and the brand resolved on the client. A brand-forked flow doubles every subsequent flow change and the two copies drift on the first change that only lands in one.

**Resolve the brand once per client construction, not per screen.** On the redirect surface the equivalent trick is injecting into `document.head` on the first screen and relying on the widget's container surviving XHR screen transitions. There is no such hack here: the client object is a real addressable singleton for its own lifetime, so resolve the brand key, fetch its tokens and messages once at construction, and hold them.

**Make the partner-supplied input a schema-typed token file, not a stylesheet.** Two things follow, and the second is the one people miss:

- A value is either valid for its declared token's type or it is rejected for that key alone, so there is no arbitrary CSS for a validation pass to guard. Reject per key and fall back per key; never discard a whole file for one bad value, and never apply an unvalidated one.
- **A token file is values, so Compose and SwiftUI consume it unchanged.** Had the input stayed CSS it could only ever brand a browser, and the native surfaces would have needed a second, parallel branding mechanism that drifts on the first brand change.

**A brand key with nothing published for it is the expected case, not an error.** Keep the default tokens and the default message set, surface nothing to the user, and log the outcome distinguishably: fetch failed against fetch never attempted. A missing, broken or slow brand asset must never break or visibly delay a login.

**Messages merge over a complete default set, single level.** Load the full default English set first, then merge the resolved locale's partner file over it, so a partner supplies only the keys it changes. Resolve the locale from the OIDC `ui_locales` request parameter first, falling back to the browser or device locale.

**Labels come from the flow, and the client still needs a floor.** Most on-screen text arrives on the collector. But a button whose collector carries no label renders as an empty control, which no user can operate and no test can find, so keep a small set of client-owned fallbacks (action label, select placeholder, and the part labels a phone collector does not supply).

**Render into the light DOM, not a shadow root,** if the styling contract is that a design team restyles by editing class names. Custom properties pierce a shadow boundary; class names do not. The exposure that buys is real (host CSS can reach and break the library's layout) and the mitigation is a prefix on every class and no element selectors, not encapsulation.

## Reference

- [`reference/collectors.md`](reference/collectors.md) - the collector catalogue for all three SDKs, the role dispatch, and the per-platform property and method names that are wrong if guessed.
- [`reference/clients.md`](reference/clients.md) - packages, versions, minimal client construction and the lifecycle API per platform.
- [`reference/forms.md`](reference/forms.md) - PingOne Form field types, which are collectable by which client, and the authoring rules.

## Provenance

Parts of the collector and client-API material here were derived from Ping Identity's own published agent skills (`pingidentity/ping-sdk-agent-skills`, commit `f210a5c`, 31-07-2026), which are MIT licensed. That material has not been confirmed against a running system by this repository, and where it conflicts with a finding recorded here from live observation, the finding wins. It already does on two counts: Ping's skills state the SDK version numbers and the collector vocabulary one minor version behind what is published.

Entries added after this initial import go through the `/pingone:learn` bar in the usual way.

## Correcting this skill

When you find an instruction here that does not match observed behaviour, or you confirm a behaviour this file does not cover, run `/pingone:learn` and describe the finding. Do not silently work around a wrong instruction, and do not edit this file from memory.

The bar for this skill has one extra clause. **A collector vocabulary read from a published type union is not a finding.** Both SDKs declare types they do not implement, so a vocabulary claim is only worth recording once it has been read from the implementation, or better, seen in a captured response.
