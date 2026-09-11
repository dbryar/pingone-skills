# Learnings journal

Newest first. One entry per finding.

This file is the history; the `SKILL.md` files are the current state. When a skill is corrected, the wrong instruction is rewritten in place and the fact that it was wrong is recorded here. That split is deliberate: a reader following a skill should not have to work out which of two competing statements is current, and a maintainer deciding whether to trust a rule should be able to see how it was established.

Entries are added by `/pingone:learn`, which will not write one until the finding is confirmed against a real system. Unconfirmed material is welcome here, marked as such, and does not go into a skill.

## Format

```markdown
## YYYY-MM-DD - one-line summary

**Skill:** core | davinci | terraform | sdk | qa (or none)
**Confirmed by:** what was actually run or read
**Versions:** pingcli x.y.z / provider x.y.z / SDK x.y.z / n/a

What was believed before, if anything, and what is true instead. What the failure looks like
and whether it is silent. What changed in the skill files, or why nothing did.
```

---

## 2026-09-11 - A read-only credential reads execution logs; `pingone:qa` added

**Skill:** qa
**Confirmed by:** a worker holding only `DaVinci Admin Read Only` at environment scope: `200` on a flow's execution listing and on an execution's events; `403` on `DELETE` of a nonexistent flow and on `POST` to `variables`, where a worker holding DaVinci Admin got `404` and `400` on the same calls. The role's permission list read back from the API: 28 permissions, all reads, including `davinci:read:interactionEvents` and `davinci:read:dvFlows`. A journey test asserting milestones from the log passed on a real run and failed, with the expected report, when the same run was checked against the enrolment path. The harness traps are each recorded in the test harness that found them, where a broken flow had passed: a click helper that matched nothing, an OTP read that returned the previous run's code (twice), a leftover session routing a first login as a returning one.
**Versions:** n/a

`pingone:qa` is new. It holds testing method rather than platform behaviour, so the platform facts it depends on stay in `core` and `davinci` and it points at them. What it states as fact: the success response's `interactionId` is the execution's `id`; a comparison node's false branch logs `success: "false"`; a node can log two `Send Response` events and a form one per render; and the read-only role above. Nothing was corrected.

## 2026-09-11 - Flow execution logs are readable per event, and subflows log under their own interaction

**Skill:** core, with pointers from davinci
**Confirmed by:** one embedded sign-in with an SMS challenge, driven end to end, then its executions read back both through the PingOne remote MCP server (`listDavinciFlowExecutions`, `getDavinciFlowExecution`) and directly through `pingcli pingone api` against `GET /environments/{envId}/flows/{flowId}/interactions` and `.../interactions/{interactionId}/events`. The `interactionId` in the client's success response matched the listed execution. Subflow lookup by the parent's `interactionId` returned `404`; lookup by `transactionId eq` returned the subflow execution with `isSubFlow: true`; a made-up `transactionId` returned zero, as a control. Read with a worker holding Organization Admin, Environment Admin and DaVinci Admin, so the minimum role is not established.
**Versions:** pingcli 1.4.0 / n/a

The DaVinci skill told readers to "read the DaVinci flow log" in two places and nowhere said how. Core now carries the recipe. The trap is the subflow: its events are not under the parent's `interactionId`, and the `404` that returns reads as "no log was kept". The parent's `startUiSubFlow` event names the subflow actually run in `properties.subFlowId`, which in the confirming run was not the subflow the node title suggested. A `next` link was present on a page shorter than the requested limit, so the skill does not tell readers to page until it disappears.

## 2026-09-11 - The PingOne remote MCP server: regional host, environment-local sign-in, and a silent empty tool list

**Skill:** core (reference only)
**Confirmed by:** DNS resolution of the candidate hosts; the server's protected-resource metadata read directly; a Claude Code debug log showing the connection with `hasTools: true` and no tools; the same server listing 77 tools immediately after the environment was opted in to the PingOne Remote MCP Server early-access feature and MCP Access was enabled on its Settings > MCP Server page, with no client change. The permission model is the console's own statement on that page.
**Versions:** Claude Code 2.1.268 / n/a

The regional host takes the full root domain (`mcp.pingone.com.au`), not a short suffix; the short form does not resolve. The authorization server is the target environment's own, so sign-in needs a user in that environment rather than an administrator from the administrators environment. The empty tool list is the expensive one: every client-side signal says connected and authenticated, and reconnecting, re-authenticating and restarting change nothing. The fix is entirely in the admin console. Recorded in `core/reference/execution-logs.md`; no `SKILL.md` rule changed beyond a pointer.

## 2026-09-11 - `pingcli pingone api` takes the method as a flag, and unknown paths return 403

**Skill:** core
**Confirmed by:** `pingcli pingone api GET <path>` failing with "command accepts 1 arg(s), received 2"; `pingcli pingone api --help`; a nonexistent path under a working prefix returning `403` "Invalid key=value pair (missing equal-sign) in Authorization header" while a sibling real path with the same token succeeded.
**Versions:** pingcli 1.4.0

`reference/pingcli.md` showed the method as a positional argument and inline JSON in `--data`. At 1.4.0 the method is `--http-method`, `--data` takes a file, and inline JSON goes in `--data-raw`. The examples were rewritten in place. The `403` on an unknown path is loud but misleading: it names the Authorization header, so it reads as a credential or role problem. Added to core's error vocabulary.

## 2026-09-09 - Ping's own published SDK skills are a minor version behind, in the direction that matters

**Skill:** sdk
**Confirmed by:** `@forgerock/davinci-client` 2.1.1's `collector.types.d.ts` and its implementation switch, read 02-09-2026; `com.pingidentity.sdks:davinci` 2.1.0's `CollectorRegistry`, read 31-08-2026; both against Ping Identity's `ping-sdk-agent-skills` at commit `f210a5c`, 31-07-2026.
**Versions:** `@forgerock/davinci-client` 2.1.1 / `com.pingidentity.sdks:davinci` 2.1.0

`pingone:sdk` was seeded from Ping's published agent skills rather than written from scratch, so the question of how far that material can be trusted was settled first rather than left implicit.

Ping's skills state the JavaScript packages are at 2.0.0 and list fourteen collector types. At 2.1.1 the published catalogue is missing `BooleanCollector`, `ValidatedBooleanCollector`, `PollingCollector`, `RichTextCollector`, `ImageCollector`, `QrCodeCollector`, `SingleSelectObjectCollector`, `ObjectSelectCollector`, `PhoneNumberExtensionCollector` and `UnknownCollector`. On Android it lists the 1.2.0 registry of eleven collector categories; 2.1.0 registers fifteen, with `BOOLEAN`, `POLLING`, `QR_CODE` and `READ_ONLY_TEXT` added and none removed.

The direction of the error is what makes it worth recording. A stale catalogue does not cause a build failure, it causes a field to be classified as unrecognised or dropped, and a dropped collector is invisible in the DOM and therefore invisible to any test that reads the DOM. So the vendor's own catalogue, taken as complete, produces exactly the silent failure this platform specialises in.

Two further corrections to material inherited from the same source. Ping's catalogue names `CheckboxCollector` and `ErrorDisplayCollector`; neither exists at 2.1.1, where a checkbox is a `BooleanCollector` carrying an `appearance` and an error is `error: string | null` on every collector. And `BUTTON` and `SINGLE_SELECT` appear in the SDK's own `StandardField` type union with no case in the implementation, which is the general rule the skill now carries: read a vocabulary from the switch statement, never from the union.

The skill records the import and its licence attribution openly, and states that a live finding beats the inherited material.

The correction that followed is structural rather than a version bump. Ping publishes a maintained cross-SDK compatibility matrix at <https://developer.pingidentity.com/orchsdks/davinci/compatibility.html>, giving the collector class and originating SDK version per field for Android, iOS and JavaScript together. That is the authority for the vocabulary, and the skill links it instead of carrying a table, because any transcription is a snapshot of something that gains a row every minor release. What the skill keeps is what the matrix does not say: that an unsupported field is silently absent rather than an error, that collector class names diverge across platforms (`ReadOnlyTextCollector` against `ReadOnlyCollector`, `QRCodeCollector` against `QrCodeCollector`), and that a type union is not an implementation.

This supersedes the counts recorded on 27-08-2026, which were read from the SDKs then shipping: the JavaScript client collecting 21 field types and Android 14. Android 2.1.0 registers fifteen collector categories where 1.2.0 registered eleven, with none removed, so the fourteen-type list is 1.2.0-era and the seven-type gap it named has narrowed. The numbers were right when taken. That they went stale inside a fortnight is the argument for linking the matrix rather than restating it, and the `pingone:davinci` copy of the vocabulary was retired to a pointer for the same reason: the constraint on how a form may be authored stays there, the per-client list does not.

Two findings arrived with it. `SKPolling` cannot be processed by any DaVinci client and must not appear in a flow, which makes Magic Link authentication incompatible with every SDK surface rather than one of them; it is a flow-level exclusion, so it never presents as a missing collector. And the matrix lists a `Translatable Rich Text` field collected as `LabelCollector` on Android and iOS, which is strong corroboration for the standing hypothesis that DaVinci rewrites a rich-text blob to `LABEL` on the way out. It is not confirmation: the matrix names fields as the form builder labels them, not by `type` enum value, so whether that field is `SLATE_TEXTBLOB` still wants one captured response.

## 2026-09-01 - A node's outcomes each need their own EVAL, not a shared one

**Skill:** davinci
**Confirmed by:** Two deploys of one subflow against a live tenant, driven through a browser. A form node with a submit button and two flow buttons declared three outcomes, each claimed by its own edge with `multi_value_source_id` set. With all three edges pointing at one shared EVAL, the submit exit routed and both flow-button exits returned `400 requestTimedOut`. Adding two more EVALs so each edge had its own, and changing nothing else — same outcome ids, same `multi_value_source_id` values, same result strings, same form — made all three exits route. The working submit exit is the control: it behaved identically across both deploys.
**Versions:** n/a (flow engine behaviour, not provider or CLI)

The skill already carried the rule that a declared outcome needs its edge to claim it through `multi_value_source_id`. That rule is correct and unchanged, but it was the only documented cause of the `requestTimedOut` signature, so a flow that satisfied it and still timed out looked like evidence that something else entirely was wrong. In this case it produced a confident and completely wrong conclusion — that a flow button's outcome `result` must not be its field key, since declaring the keys as results "did not work". The result strings had been right the whole time; the evaluator fan-out was the fault.

Both faults present identically: the node logs success in milliseconds, no downstream node is logged at all, and the client sees `400` with `requestTimedOut` after the flow's HTTP timeout. Neither is visible in the flow JSON without knowing to look. Added the sibling rule immediately after the existing one, with the pairing called out in both directions, because the diagnostic difficulty is entirely in telling them apart rather than in fixing either.

## 2026-08-31 - A collection endpoint returning 500 means one poisoned row, not an outage

**Skill:** terraform
**Confirmed by:** deleting the single connector instance whose specific-ID `GET` had returned 500 for weeks, and watching `GET /connectorInstances` go from 500 to 200 with all 17 instances present, on the next request
**Versions:** pingidentity/pingone ~> 1.21

An environment's `GET /connectorInstances` list returned 500 continuously for roughly three weeks
while specific-ID reads of individual instances returned 200. This was recorded, including in the
first version of the entry below, as a PingOne-side outage of the list endpoint, and a workaround was
built around it: adopt by specific-ID read, accept that discovery is unavailable.

That was wrong. Exactly one instance in the environment was unreadable — its own specific-ID `GET`
returned 500 on every attempt across three weeks, while every other instance returned 200. The list
endpoint was failing because it could not render that one member. **PingOne collection endpoints
fail whole rather than skipping a member they cannot serialise.** Deleting the single bad record
restored the list on the next request.

The practical consequence is a diagnostic rather than a fix: a 500 from a collection endpoint is
evidence about one unidentified row, not about the collection. Walk the IDs you know individually;
the one that 500s while its neighbours return 200 is the cause.

Two properties of such a record, both confirmed and both counterintuitive. It **can be deleted**
despite never having been readable — `DELETE` returned 204 and the follow-up `GET` returned 404. And
it **cannot be imported**, because `tofu import` needs a read first, so a broken record can only be
replaced, never adopted. Ruled out by control along the way: it is not that `ping`-category
connectors with no `properties` are unserialisable, since a throwaway instance of the same connector
with the same null properties read back 200 twice and deleted cleanly.

---

## 2026-08-31 - The connector catalogue is the authority on default instance names

**Skill:** davinci
**Confirmed by:** `GET /v1/environments/{envId}/connectors` against a live environment (230 connectors), cross-checked against the names on instances Studio created unprompted in that same environment
**Versions:** pingidentity/pingone ~> 1.21

Follow-on from the duplication entry below. That one said to name a managed connector instance after
the connector's catalogue name without saying what those names are, which leaves the reader guessing
at exactly the point the guess is wrong: the connector id does not predict the name. `nodeConnector`
is `Teleport`, `flowConnector` is `Flow Conductor`, `errorConnector` is `Error Message`,
`pingOneFormsConnector` is `Form`. A reader inventing `FlowConnectorInstance` or `flowConnector`
lands back in the indistinguishable-from-a-duplicate state the rule exists to avoid.

The catalogue endpoint returns `name` and `metadata.type` together, so one read answers both the
naming question and the which-`properties`-shape question. `reference/connectors.md` gains the table
for the connectors in confirmed use, and the `SKILL.md` bullet now points at it rather than carrying
a partial second copy.

Worth noting for anyone building the same capture: capability names are **not** available from the
Management API, on the list or on a single connector. Only `name`, `metadata.type`, `version`,
`description` and timestamps are. Capabilities exist only in the DaVinci API's own connector
catalogue, which returns each capability's full property schema alongside them and runs to 27MB for
230 connectors against 24KB for the same set stripped to name/type/version.

---

## 2026-08-31 - Studio duplicates a connector instance after your apply, not only before it

**Skill:** terraform, davinci
**Confirmed by:** a connector-instances dump from a live environment, ordered by `createdDate`, cross-referenced against `tofu state list` and the connector catalogue from `GET /environments/{envId}/connectors`; specific-ID `GET /connectorInstances/{id}` on each duplicate pair
**Versions:** pingidentity/pingone ~> 1.21

Both skills said to check for an existing connector instance before adding a resource, which framed
the problem as a one-time ordering question: Studio got there first, so adopt instead of create. That
is only half of it. The creation-time ordering shows Studio also creates an instance for a connector
Terraform is *already* managing, twelve and fourteen days after the apply that created the managed
one, when someone next opened a flow using that connector on the canvas.

The failure is entirely silent and has no expiry. The duplicate is not in state, so `plan` is clean
forever; flows bind by instance ID, so nothing misbehaves; and the environment simply accumulates
pairs. It surfaces only when the duplication is promoted to another environment, or when a human
looks at the connector list and cannot tell which of two instances is the real one.

One reliable discriminator: instances created through the console carry a `customerId` field and
Terraform-created ones do not. That held across every instance in the dump examined, managed and
unmanaged.

Also observed: `GET /connectorInstances` (the list) returned HTTP 500 while specific-ID
`GET /connectorInstances/{id}` on the same environment returned 200. The reason for that split is
recorded in the entry above, which supersedes the reading given here — it was not an API-wide
condition to work around.

Skills updated: the terraform skill's "Adopting what already exists" now says the side effect recurs,
that `plan` cannot detect it, and gives the state-rm / import / apply / delete order, with the
ordering constraint that the orphan is deleted only after dependent flows have been re-applied. The
davinci skill's connector notes gained the naming convention and a pointer to that procedure.

**Not asserted, because it was not confirmed:** that naming a managed instance with the connector's
catalogue default name stops Studio creating a duplicate. It is the obvious hypothesis, but the
matching rule Studio actually uses was not established, and there is a data point against it: an
instance renamed *away* from its catalogue default did not subsequently acquire a Studio-created
twin despite that connector being authored on the canvas again. The naming convention is recorded as
a legibility measure, not a preventative.

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

## 2026-08-28 - a node's description is its height on the canvas

**Skill:** davinci
**Confirmed by:** Reading a published flow's canvas. Descriptions run from roughly 50 to 480 characters across one 115-node flow, and node height tracks that directly: the longest turn into tall columns of body text several times the height of a node carrying one sentence, to the point where the diagram stops being readable at normal zoom.
**Versions:** n/a - Studio canvas rendering

Nothing in the skills said where a `nodeDescription` ends up. It is rendered inside the node body, so a description written as a paragraph of reasoning becomes a tall box, and a flow full of them becomes a diagram nobody can take in.

The pull is real and worth naming: a description is the obvious place to record why a node is built the way it is, especially in a generated flow where the surrounding code comments are not visible to whoever opens Studio. The cost is not visible from the authoring side at all — the flow applies cleanly, nothing warns, and the damage only shows on a canvas someone else opens later.

Added the rule to "The graph model" beside the note on positions, since both are about a flow staying readable. Recommended capping the length in the generator, because the platform will not.

## 2026-08-28 - a screen is painted once, and `disabled` on an skbutton is the widget's

**Skill:** davinci
**Confirmed by:** A hosted screen whose flow looped back to the same `customHTMLTemplate` node. The loop demonstrably re-executed the node — its connector call ran and re-issued a one-time code, and the new code validated where the previous one no longer did — while an element marked with a `data-` attribute before the loop still carried that mark 45 seconds later. Counters written into the DOM from the screen's own script survived the round trip with their values continuing from the first execution rather than restarting, so the script did not run a second time. In the same trace, the script's click handler fired, set `disabled` on an skbutton, and the button read enabled nine seconds later with no path in the script having cleared it; the same script's first arming of that state, before any submission, held correctly for its full 60 seconds. Company-variable substitution was confirmed by a control in the opposite direction: the screen offered a control only when a company-scoped limit read greater than zero, and the control was offered.
**Versions:** n/a - widget and flow-runtime behaviour, not a provider or CLI

The skills said the error re-render patches only the error region and does not re-run `customScript`. That was true and too narrow: it is not a property of error re-renders. **Any** re-entry to the same screen node leaves the DOM alone and the script un-run, so a screen gets exactly one execution for its whole life. Rewrote that line to point at the general rule and added the rule to "The HTML/CSS/JS surface".

The practical cost of the narrow version is that anything armed at render — a countdown, a disabled control, a rate limit — covers the first pass and no later one, while looking correct on the pass anyone tests first. Nothing errors and the screen keeps showing what it showed.

Also added: a script-set `disabled` on an element carrying `data-skcomponent="skbutton"` does not survive a submission. The widget owns that property on its own components and restores it, so a control gated that way looks gated in the DOM and refuses nothing. Gate with a class on your own container plus a `document`-captured click listener instead, and drive the visuals from the same class. `pointer-events: none` is worth having and is not the gate — it stops a pointer, not a keyboard and not `.click()`. What the widget does not touch is your own container class, your `data-` attributes, and a `hidden` your script cleared.

Third, smaller: `{{global.company.variables.<name>}}` resolves inside a subflow, including inside a screen's `customScript`. The skill's subflow section warned only about `global.variables`, which left it open whether the company namespace was reachable from a subflow at all. Added one line next to that warning.

Deliberately left out: the mechanism behind the `disabled` restoration. That the widget re-enables its buttons when a submission's response lands is the obvious reading and was not observed directly — what was observed is that a script-set value does not survive the round trip, which is what a reader needs either way.

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
