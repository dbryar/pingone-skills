# PingOne Forms as the client contract

Read alongside `SKILL.md`. This file is the authoring side: what a form may contain if an SDK is going to render it. `pingone:terraform` owns the `pingone_form` resource shape; `pingone:davinci` owns the flow that shows the form.

## The field types, and who can collect them

**Ping publishes a maintained cross-SDK matrix, and it is the authority: <https://developer.pingidentity.com/orchsdks/davinci/compatibility.html>.** It gives, per PingOne Forms field, the collector class name and the SDK version it arrived in, for Android, iOS and JavaScript side by side. Read it rather than a transcription: a copy here would be a snapshot of a matrix that gains a row every minor release, and a stale vocabulary is the specific failure this file exists to prevent.

What the matrix does not tell you is below.

### It lists what is supported, not what happens when something is not

An unsupported field does not error. It is missing from the collector list, so the screen renders looking complete and cannot be submitted. The matrix reads as a feature table; it is really a list of the fields that will not silently vanish.

### The collector class names diverge across platforms, so grep for the wrong one and find nothing

| Field | Android and iOS | JavaScript |
| --- | --- | --- |
| Agreement | `ReadOnlyTextCollector` | `ReadOnlyCollector` |
| QR Code | `QRCodeCollector` | `QrCodeCollector` |
| Translatable Rich Text | `LabelCollector` | `RichTextCollector` |

### Three fields are web-only, and one of them is a password field

`Image` (`ImageCollector`) and `New Password` (`ValidatedPasswordCollector`) are JavaScript only, unsupported on both native platforms. A registration or reset form built around `New Password` therefore works in a browser and cannot be completed in an app, which is the exact shape of failure that passes a browser-based review.

### `SKPolling` cannot be processed at all

Ping's own wording: "SKPolling components cannot be processed by the DaVinci Client and should not be included in flows." Anything built on it, Magic Link authentication included, is incompatible with every SDK client rather than with one of them. This is a flow-level exclusion, not a field-level one, so it will not show up as a missing collector: the flow simply cannot be driven.

Images embedded in a Custom HTML Template, backgrounds and logos included, are likewise not processed. Brand imagery belongs in the client, which is where the rest of the branding already lives.

### Two field types appear in PingOne's stock sign-on form and in no SDK

`SLATE_TEXTBLOB` and `ERROR_DISPLAY` appear nowhere in either SDK's source, in any file, and both are in PingOne's own stock sign-on form. **A stock form therefore cannot be adopted as authored.**

The matrix narrows the first of these considerably. It lists a `Translatable Rich Text` field collected as `LabelCollector` on Android and iOS and `RichTextCollector` on the web, with link support added in 2.1.0. `LABEL` is collectable by every client and is not itself a PingOne form field type, so DaVinci rewriting a rich-text blob to `LABEL` on the way out is now the documented behaviour rather than a guess. What is still unconfirmed is the identifier: the matrix names fields the way the form builder labels them, not by their `type` enum value, so whether `Translatable Rich Text` is `SLATE_TEXTBLOB` wants one captured `showForm` response to settle. Nothing should be built on the assumption until then.

`ERROR_DISPLAY` has no such candidate. On the web an error is `error: string | null` on every collector rather than a collector of its own, which is consistent with the field never producing one.

## Authoring rules

- **Author to the narrowest client that will render the form.** If a native app is one of the consumers, the Android vocabulary is the constraint, not the web one. A form authored to the web vocabulary passes a browser-based review and collects nothing in the app.
- **Enforce the vocabulary at generation time.** It cannot be caught at runtime: the failure is a missing field, not an error.
- **A key is a branch target.** The flow matches on a `FLOW_BUTTON`'s `key`, not its label, so renaming a key is a flow change. Route on which button submitted, rather than inspecting a field's value to infer it.
- **Validate in the form where the format is known.** A six-digit OTP validated with a `CUSTOM` regex on the field is rejected at the client rather than costing a flow round trip and a generic failure.
- **A device field's `options` are method types, not devices.** `DEVICE_AUTHENTICATION` and `DEVICE_REGISTRATION` declare which of `SMS`, `EMAIL`, `TOTP`, `FIDO2` the screen offers; PingOne substitutes the user's actual devices at runtime. What the collector's options contain is not what the author wrote.
- **A polling field carries the field, not the cadence.** `enablePolling`, `pollInterval` and `pollRetries` are the flow node's, and the `POLLING` field is what the client receives as the collector that drives it.

## What to assert against a real response

The only useful renderability check is the SDK's own field path. A response is renderable when it carries a top-level `form.components.fields`, because that is the only path either client builds collectors from.

Do not assert `isResponseCompatibleWithMobileAndWebSdks`. It appears on the completed response, never on a screen response, and no implementation reads it.

Prove a form from the web client before the native one. The same DaVinci response drives both, with no Gradle build in the loop, so a wrong flow is wrong in both and finding out in a test run is cheaper. It also keeps the flow's shape and the app's SDK wiring as two separately diagnosable unknowns.
