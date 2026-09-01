# Learnings journal

Newest first. One entry per finding.

This file is the history; the `SKILL.md` files are the current state. When a skill is corrected, the wrong instruction is rewritten in place and the fact that it was wrong is recorded here. That split is deliberate: a reader following a skill should not have to work out which of two competing statements is current, and a maintainer deciding whether to trust a rule should be able to see how it was established.

Entries are added by `/pingone:learn`, which will not write one until the finding is confirmed against a real system. Unconfirmed material is welcome here, marked as such, and does not go into a skill.

## Format

```markdown
## YYYY-MM-DD - one-line summary

**Skill:** core | davinci | terraform (or none)
**Confirmed by:** what was actually run or read
**Versions:** pingcli x.y.z / provider x.y.z / n/a

What was believed before, if anything, and what is true instead. What the failure looks like
and whether it is silent. What changed in the skill files, or why nothing did.
```

---

## 2026-09-01 - A node's outcomes each need their own EVAL, not a shared one

**Skill:** davinci
**Confirmed by:** Two deploys of one subflow against a live tenant, driven through a browser. A form node with a submit button and two flow buttons declared three outcomes, each claimed by its own edge with `multi_value_source_id` set. With all three edges pointing at one shared EVAL, the submit exit routed and both flow-button exits returned `400 requestTimedOut`. Adding two more EVALs so each edge had its own, and changing nothing else — same outcome ids, same `multi_value_source_id` values, same result strings, same form — made all three exits route. The working submit exit is the control: it behaved identically across both deploys.
**Versions:** n/a (flow engine behaviour, not provider or CLI)

The skill already carried the rule that a declared outcome needs its edge to claim it through `multi_value_source_id`. That rule is correct and unchanged, but it was the only documented cause of the `requestTimedOut` signature, so a flow that satisfied it and still timed out looked like evidence that something else entirely was wrong. In this case it produced a confident and completely wrong conclusion — that a flow button's outcome `result` must not be its field key, since declaring the keys as results "did not work". The result strings had been right the whole time; the evaluator fan-out was the fault.

Both faults present identically: the node logs success in milliseconds, no downstream node is logged at all, and the client sees `400` with `requestTimedOut` after the flow's HTTP timeout. Neither is visible in the flow JSON without knowing to look. Added the sibling rule immediately after the existing one, with the pairing called out in both directions, because the diagnostic difficulty is entirely in telling them apart rather than in fixing either.

## 2026-08-31 - A node's declared outcome routes nowhere unless its edge claims it

**Skill:** davinci
**Confirmed by:** Four deployed versions of one flow in a live sandbox, driven end to end by `@forgerock/davinci-client` in a real browser, with DaVinci's own flow-execution logs read from Studio for each. v5 (no outcome, plain edge), v6 (outcome added, plain edge) and v7 (outcome plus corrected downstream bindings) all failed identically; v8 (outcome plus the edge carrying `multi_value_source_id`) completed and issued a real authorization code. Edge and node schemas read from a `tofu providers schema -json` dump.
**Versions:** pingone provider 1.21.0 / @forgerock/davinci-client 2.1.1 / com.pingidentity.sdks:davinci 2.1.0

Previously the skill said an edge is "a link between node IDs and nothing more" and that "there is no edge-level label anywhere". Both are wrong. `edge.data` carries `multi_value_source_id`, which names which exit of the source node the edge leaves by, and it is **required** on every edge leaving a node with more than one named exit — a multi-select's options, or any node declaring `outcomes`.

The failure is silent and actively misleading. The connector executes and the flow log records it succeeding in a few milliseconds, reporting its result; then **no downstream node is logged at all**. The interaction runs out `flowHttpTimeoutSeconds` and the client receives `400` with `code: "requestTimedOut"` and a `CONSTRAINT_VIOLATION` saying "a capability took longer than allowed or the flow was misconfigured". Nothing timed out and no capability was slow — the result had nowhere to go. A connector log reporting success with no subsequent node entry, plus a client-side timeout, is the signature.

Two hypotheses were tested and disconfirmed first, both about the node rather than the edge: a missing `outcomes` declaration (real defect, worth fixing, not the cause) and a downstream function binding a whole node payload object rather than named fields (also worth fixing, also not the cause). Nothing in the symptom pointed at the edge, which is why the lint rule is now stated explicitly.

Recorded alongside, from the same session:

- `isResponseCompatibleWithMobileAndWebSdks` appears **only on the completed response**, never on a screen response, and no implementation file in the JavaScript client reads it — one occurrence, in a type declaration. It is not a renderability gate. Both the JavaScript and Android clients build collectors from a top-level `form.components.fields` and nothing else.
- The embedded SDK surface needs a **CORS allow-list on the OIDC client**, because the SDK calls `/as/authorize` via `fetch` from the page origin. A client created for a redirect journey has none.
- Driving the SDK from a **server-side JavaScript runtime fails identically against a known-working flow**, so a failure there says nothing about the flow. Debug embedded surfaces in a browser.
- A form field keyed `user.username` is authored and submitted **dotted** and returned to the flow **nested** (`output.formData.user.username`).
- `pingOneFormsConnector` is a **`core`-category** connector (`metadata.type`), so `properties = null` and no environment credential, despite the name.
- A Studio export spells the edge field `multiValueSourceId`; the provider spells it `multi_value_source_id`. Carrying the camelCase spelling into HCL makes Terraform drop it silently.

## 2026-08-27 - Forms are separate PingOne objects, and three field vocabularies disagree

**Skill:** terraform (resource shape, form-ID substitution), davinci (showForm node, field vocabularies)
**Confirmed by:** `tofu providers schema -json` for the `pingone_form` resource and its field `type` enum; a targeted `tofu plan` for a newly declared form; a real exported form and a real exported flow using seven `showForm` nodes; the JavaScript SDK's own field-to-collector switch; the Android SDK's `classes.jar` class list and the README bundled in its sources jar.
**Versions:** `pingidentity/pingone` provider (schema read 27-08-2026) / `@forgerock/davinci-client` 2.1.1 / `com.pingidentity.sdks:davinci` 1.2.0

Neither skill covered forms at all. A form turns out not to be flow content: it is a first-class
PingOne object with its own ID and lifecycle, and a `showForm` node holds nothing but a reference
to it in `properties.form.value`. Because `properties` is an opaque string, that reference needs
the same token-and-`replace()` substitution the skill already documents for `subFlowId`, and it
fails the same silent way, applying cleanly with a literal placeholder that neither `validate` nor
`plan` can see.

The finding with the most consequence is a three-way vocabulary mismatch. The form builder authors
24 field types, the JavaScript SDK collects 21, and the Android SDK collects 14. A field outside
the consuming SDK's set is **dropped from the node's collector list rather than raising**, so the
screen renders looking complete, cannot be submitted, and gives no indication why. A browser-based
review does not catch it, because the web vocabulary is the wider one. This is why the rule is to
author to the narrowest consuming SDK at authoring time.

Also recorded: a client SDK's published types are not a safe source for its vocabulary. The
JavaScript SDK's `StandardField` union declares `BUTTON` and `SINGLE_SELECT`, and neither has a
case in the code that builds collectors. The implementation is the authority.

Deliberately not recorded: whether `SLATE_TEXTBLOB` and `ERROR_DISPLAY`, both present in PingOne's
stock sign-on form and absent from both SDKs, are rewritten by DaVinci before a client sees them.
`LABEL` is collectable by both SDKs and is not a form field type, which makes a `SLATE_TEXTBLOB` to
`LABEL` rewrite the obvious explanation, but that is a hypothesis and no `showForm` response has
been captured to settle it. Nothing about apply-time behaviour is recorded either; the form was
planned, not applied.

## 2026-08-27 - a node with no inbound edge is an entry point, and an unwired terminal returns on load

**Skill:** davinci
**Confirmed by:** A subflow generated with 12 of its edges missing (91 nodes / 71 edges) returned an empty payload to its caller and terminated with no screen rendered; the same flow with those edges present (91 / 83) rendered its screen and completed. Reproduced for two different users from a cleared browser state, and the corrected flow verified end to end afterwards.
**Versions:** n/a - graph semantics, not a provider or CLI behaviour

The skills covered `startNode` needing no inbound edge, which reads as if being unwired is a property of that node type. It is not: DaVinci treats *every* node with no inbound edge as a place the flow starts, and `startNode` is simply the one where that is intended.

The consequence is worst for terminals. An unwired `createSuccessResponse` fires on entry and returns before anything has run, so a caller gets a blank claim and every downstream comparison silently takes its default branch. Nothing errors, the graph is valid, and `terraform plan` and `apply` both succeed. The flow ends on a real, named outcome that has nothing to do with the actual failure, so the redirect points the reader at whichever router defaulted rather than at the disconnected node.

Added the rule to "The graph model" in `davinci/SKILL.md`, next to the existing edge-level rules, with a pointer from the `startNode` line in "Teleports" so the two do not contradict each other. Deliberately left out: the specific generator bug that dropped the edges, which was local tooling rather than anything about DaVinci.

## 2026-08-25 - Custom claims on a success terminal, and which token they belong in

**Skill:** davinci
**Confirmed by:** reading a live, enabled flow export (`flowStatus: enabled`, current version 7) from a real tenant, and the source of the OIDC client library consuming it
**Versions:** n/a (DaVinci flow content)

The skills covered `returnSuccessResponseRedirect` as a terminal but said nothing about the claims it emits, which is most of what anyone configures on it.

Confirmed from the export:

- `accessTokenClaims` and `idTokenClaims` are two independent `properties` entries. The terminal read carried seventeen rows on one and a single row on the other, which is a real configuration rather than a partial one.
- Row shape is `name`, `value` (a Slate JSON string), `type`, `key` (a float, unique only within its own list), plus `label` and `nameDefault`. The last two drift from `name` as rows are edited and are never reconciled: the export contained a row labelled as an email address whose binding pointed at an unrelated field.
- A claim block pasted from another flow keeps that flow's node IDs. The export held thirty-six `{{local.<nodeId>...}}` bindings naming a node absent from its own graph, on a flow the platform had accepted and was serving. Nothing validates a claim binding against the graph at import, at save, or at deploy, so this is silent in the strongest sense: there is no error surface at all, only a claim built from nothing.
- A terminal reached from several branches only sees nodes that ran on the branch taken. Sourcing its claims from a lookup that runs on one branch leaves them empty on every other, which again reports nothing.

Recorded as guidance rather than as observed platform behaviour, and marked as such in the skill:

- Overriding `sub` in the id token. OpenID Connect Core §5.3.2 requires a relying party to reject a `/userinfo` response whose `sub` disagrees with the id token's, and the client library read here implements that check and throws. Whether DaVinci permits the override was not tested; the advice not to attempt it stands either way.
- Preferring identity claims in the id token, per the id-token-as-assertion / access-token-as-credential split and RFC 9068 §6 on identity detail in a forwarded access token. Deliberately non-blocking: resource servers that cannot call `/userinfo` or introspect are common in an older estate, and reading the access token directly is a constraint to record rather than an error to refuse. The skill says to keep the set minimal and not to propagate the arrangement by copying the terminal.

One consequence worth its own line, from the client library rather than the platform: a client that refreshes tokens and rebuilds its user from the new id token instead of re-calling `/userinfo` will fall back to bare `sub` if the identity claims live only on the access token. Moving claims between the two lists is therefore not a neutral relocation.

Not written down, because it is unconfirmed: whether a whole-field plain-text `{{...}}` Slate child substitutes in a claim value. The export used that form for one claim and the `link`-typed form for the rest, and the flow was live, but no observation isolates the plain-text row.

## 2026-08-14 - Initial skill set

**Skill:** core, davinci, terraform
**Confirmed by:** findings accumulated from building and operating a PingOne estate: live tenant API calls through pingcli, real `tofu apply` runs against a licensed environment, and deployed DaVinci flows driven through a real browser
**Versions:** pingcli 1.3.0 / pingidentity/pingone ~> 1.21

First publication. Every rule in the three skills was observed rather than derived, with the exception of a small number of statements marked as unconfirmed in place.

Findings that changed a previously held belief, and are recorded because the wrong version looked correct:

- A DaVinci subflow does not see the caller's `flowInstance` variables. Its input is reached as `{{global.parameters.<name>}}`. The wrong binding does not error; it compares against nothing and takes the mismatch branch on every request, so the flow appears to work and simply never takes the matching path.
- `checkSession`'s `authenticationMethodLastUsedIn` is in minutes, not seconds, while `cookieConnector`'s `cookieExpiresInSeconds` is in seconds. The same number in one flow meant two different things, and a value intended as twelve hours was asking for thirty days.
- A PingOne-category connector instance's environment property key is `envId`, not `environmentId`. The write API does not validate property keys against the connector schema, so the wrong key was stored and echoed back correctly on every read. The configuration looked confirmed for two rounds of debugging and had never worked.
- A `lifecycle { ignore_changes = [properties] }` block, added to work around a provider consistency-check bug, silently discarded every subsequent correction to that attribute. The shape being debugged had been correct the whole time.
- `pingone_mfa_device_policy` applies successfully and governs nothing. The platform auto-creates a singleton default policy when the MFA service is enabled, and only that singleton controls device pairing.
- A flow policy's `flow_distributions[].version` does not follow the flow's `current_version`. Updating a flow leaves the environment serving the old content, with no error at plan or apply.
- An `errorConnector` / `customErrorMessage` node displays by returning to the last-rendered screen, so it is fatal when reached before any screen has rendered. The resulting dead flow is reported to the relying party as `login_required` with the same generic description a legitimately refused silent authentication produces, which caused a wrong conclusion to be drawn from byte-identical error text.
- `ACCESS_FAILED: You do not have permissions or are not licensed` on an MFA policy call was the environment lacking the `MFA` service, not the credential lacking a role.
- Creating an environment with a licence attached needs `Organization Admin`, not `Environment Admin` at organisation scope.
- `Identity Data Admin` declares `applicableTo: ["ENVIRONMENT", "POPULATION"]`. It cannot be held organisation-wide by any identity in any tenant, which is a role-definition constraint rather than an entitlement gap.
- pingcli's client-credentials login rejects `CLIENT_SECRET_POST` with `invalid_client: Unsupported authentication method`. PingOne accepts the application configuration without complaint. This is documented nowhere.
