---
name: terraform
description: Manage PingOne and DaVinci with Terraform or OpenTofu - provider resource shapes, using plan output as a drift detector to reconcile hand-made or CLI-made changes back into HCL, adopting existing resources, and the provider failure modes that produce orphans, tainted resources or silently stale content. Use whenever a task involves .tf files targeting PingOne, reading a plan, importing a resource, or diagnosing an apply error. Assumes pingone:core for authentication and pingone:davinci for what the flow attributes mean.
---

# PingOne and DaVinci as code

Two things this skill exists for. The first is the resource shapes, which the provider documentation covers incompletely and the schema dump covers misleadingly. The second is the working method: **treating the plan as a drift detector**, so that a change made by hand in the console or through `pingcli` can be reconciled back into HCL deliberately rather than discovered later as an accident.

Companion skills: `pingone:core` for tenant operations and authentication, `pingone:davinci` for what the flow attributes actually mean at runtime.

## The plan as a drift detector

An empty plan is an assertion: the code and the live environment agree. That makes the plan the primary instrument for a workflow where humans change things in the console and agents change things through the CLI, and both have to end up in the code.

### The loop

1. **Start from an empty plan.** If the plan is not empty before you change anything, you do not have a baseline and every count you read afterwards is a mixture.
2. **Make the change by hand**, in the console or through `pingcli`. Do it deliberately and note what you did.
3. **Run `plan` and read the diff, not the count.** The count tells you how much reconciling there is. The diff tells you what the platform actually did, which is frequently more than you asked for: defaults it filled in, fields it normalised, objects it created alongside yours.
4. **Write HCL until the plan is empty again.** For a new object, that means either declaring it and importing it, or declaring it and letting it be created if the hand-made one was throwaway. For a changed attribute, it means changing the code to match.
5. **An empty plan is the pass condition.** Not "the remaining diff looks harmless".

### Reading a non-empty plan

Sort every line into one of four buckets before deciding anything:

| Bucket | Meaning | Action |
| --- | --- | --- |
| Drift you just caused | You changed it by hand and the code has not caught up | Reconcile into HCL |
| Drift someone else caused | Live state you did not create and cannot explain | Investigate before touching. Confirm through `pingcli` whether it is real live state or stale bookkeeping. Do not bundle it into an unrelated apply |
| Defaults the platform filled in | The API supplied a value your code left unset | Usually set it explicitly, so it stops being invisible |
| Genuine code change | What you actually intended | Apply |

Bundling the second bucket into an apply for the fourth is how unrelated live configuration gets destroyed by someone who was doing something else. When an unscoped plan surfaces drift you cannot explain, scope the apply instead:

```bash
tofu plan
tofu apply -target=module.x.resource.y -target=... -refresh=false
```

`-refresh=false` keeps the apply from folding in refreshed state for resources you are deliberately not touching. Use both together, use them sparingly, and record what you scoped out and why.

### Adopting what already exists

Prefer `import` blocks or `tofu import` over letting Terraform create a second copy. This matters more with PingOne than with most providers because several of its objects are singletons the platform creates for you, and several more get created as a side effect of using the console.

The trap: an object created as a side effect already exists by the time your code declares it. Adding it to an existing `for_each` set creates a duplicate rather than adopting the original. **Check for an existing instance before adding any resource that the console or Studio may already have created for you.** DaVinci connector instances are the standard case: Studio creates one the moment a node using that connector is first authored.

**Checking once, before you write the resource, is not enough.** The side effect recurs. Studio creates its own connector instance the first time a connector is used on a canvas whether or not a Terraform-managed instance for that same connector already exists, so duplicates appear days or weeks *after* a clean apply, whenever someone next opens the flow. Nothing surfaces this: the extra instance was never in state, so `plan` stays empty indefinitely, and a flow keeps working because it binds by instance ID. The cost is paid later, when the same duplication is carried into the next environment and it stops being obvious which of two instances a flow should reference.

Audit for it on a schedule rather than trusting the plan. Console-created DaVinci connector instances carry a `customerId` field that Terraform-created ones do not, which separates the two populations in one pass over the connector-instances list. To reconcile a duplicate: `state rm` the Terraform-managed copy, `import` the console-created one to the same address, apply so every dependent flow re-resolves to the surviving ID, and only then delete the orphan. Deleting before the dependent flows have been re-applied leaves them pointing at an instance that no longer exists.

### Restructuring without destroying

Use `moved` blocks, not `terraform state mv`. A `moved` block is reviewable, lands in the same commit as the restructure, and lets `plan` prove the move is a pure relocation. Confirm the plan shows the resources moved with zero attribute diff and zero destroyed before applying.

**`path.module`-relative `file()` calls do not survive a module move.** A path with three `../` becomes four when the module goes one level deeper. Check every one explicitly on any move; it fails at plan time with a nonexistent-path error, which is at least loud.

### Removing a resource safely

Code and state must move in the same change. Removing code before state updates destroys the live object. Removing state before code updates creates a duplicate on the next apply. Confirm with `state list`, remove from state and delete the resource block in the same commit, read the plan, then apply.

### Non-negotiables

- **Read the plan before every apply.** "The pipeline would have caught it" is not a substitute.
- **Never auto-apply to any environment, including non-production.** Keep the apply stage manually gated.
- If local development uses OpenTofu and CI applies with Terraform, author to the OpenTofu feature set. Terraform CLI is a strict superset, so anything `tofu` accepts, `terraform` accepts. The reverse does not hold.

## Which provider are you on

Two different providers manage DaVinci, with incompatible models. Establish which before reading any example.

| | `pingidentity/pingone` | The separate `davinci` provider |
| --- | --- | --- |
| Flow resource | `pingone_davinci_flow` | `davinci_flow` |
| Flow content | Structured attributes: `graph_data`, `settings`, `trigger`, `input_schema`, `output_schema` | One opaque `flow_json` string |
| Connector references | Computed from the graph | Manual `connection_link` / `subflow_link` remapping blocks |
| Connector instances | `pingone_davinci_connector_instance` | `davinci_connection` |

Guidance written for one is actively wrong for the other. `davinci_connection` is not a resource type in the `pingone` provider at all.

**Verify resource and attribute names against the pinned provider's own schema before writing HCL**, rather than against memory or a blog post:

```bash
tofu providers schema -json | jq '.provider_schemas | keys'
tofu providers schema -json | jq '.provider_schemas["registry.opentofu.org/pingidentity/pingone"].resource_schemas.pingone_davinci_flow'
```

When the schema dump and the provider's published registry documentation disagree about whether something is required, **believe the documentation and copy its examples verbatim**. The schema marks a great many things optional that the API requires in practice, and the gap only surfaces as an API error at apply time.

## `pingone_davinci_flow`

The shapes below were each established by an apply failing. They are not derivable from the schema.

- **`graph_data.elements.nodes` and `.edges` are maps keyed by ID, not lists.** The failure is `attribute "edges": map of object required`.
- **Every node and edge needs the full Cytoscape boilerplate**: `group`, `removed`, `selected`, `selectable`, `locked`, `grabbable`, `pannable`. The schema marks most of these optional. The API rejects their absence at `CreateFlow`, not at validate or plan. Edges additionally need `position = { x = 0, y = 0 }`, which is meaningless for an edge and still required. Copy the boilerplate from the provider's own documented example.
- **`connection_id` is not computed. Set it per node** from a real `pingone_davinci_connector_instance.<x>.id`. `connector_id` is a static string naming the connector *type*; `connection_id` is the instance. Omitting it passes validate and produces a flow referencing no real connection. The working pattern: have the generator discover every distinct `connector_id` the graph uses, create one instance per distinct value with `for_each`, and inject `connection_id` into each decoded node keyed on `connector_id` after `jsondecode()`. Do not hand-maintain the connector list; derive it from the graph so the two cannot disagree.
- **`connectors` *is* computed.** The provider resolves it from the graph against instances that must already exist in the environment. This is what replaces the older provider's manual remapping blocks.
- **`color` must be set explicitly.** The API defaults an unset colour server-side, the provider does not mark the attribute computed, and Terraform's post-apply consistency check then fails and taints the resource.
- **`trigger.type` accepts only `AUTHENTICATION` or `SCHEDULE`.** `validate` rejects anything else before an API call.
- A subflow is the same resource, with `trigger = null` and `input_schema` / `output_schema` populated. `output_schema.output.properties` is itself a JSON-encoded string.

### Subflow references need a substitution, not a reference

`connection_id` resolution does not extend to subflows. A subflow target lives inside `properties.subFlowId`, and `properties` is a plain **string** in the provider schema, so there is no structured attribute for the provider to compute against.

The mechanism is a textual substitution:

1. The generator emits a deterministic token, for example `__SUBFLOW_ID__<flow-name>__`, into that string.
2. The HCL runs `replace()` over the **whole file text before `jsondecode()`**, not over individual nodes after it. One substitution, and no reaching inside an opaque string to find a token.
3. Because the replacement reads `pingone_davinci_flow.<subflow>.id` directly, a destroyed and recreated subflow is picked up in the same plan as its caller. No remote state read, no hardcoded ID map.

**`replace()` fails silently.** If the generator's token and the HCL's ever disagree, `replace()` returns the string unchanged, the flow applies cleanly carrying a literal placeholder, and it surfaces only at runtime as a subflow call that does not resolve. Neither `validate` nor `plan` can see it, because both sides are just strings.

A `lifecycle.precondition` asserting no placeholder survived substitution is the only thing standing between a typo and a broken flow. It is not redundant with a generator-side check: the generator can only verify that a token it emits names a flow it also generated, which is a different failure from the two sides disagreeing.

Ordering needs no `depends_on`. The caller's substitution references the subflow's `id`, and it must also depend on the subflow's `pingone_davinci_flow_deploy`, since `subFlowVersionId: -1` resolves to the latest **deployed** version.

## Deploy and flow policy

`pingone_davinci_flow_deploy` is a separate resource, not an attribute of the flow. It takes `flow_id` and an optional `deploy_trigger_values` map; any changed value forces a redeploy. That redeploy shows in a plan as a destroy plus create. **This is the resource's designed behaviour, not an unintended loss**, and it is worth recognising on sight so nobody aborts a correct apply over it.

`pingone_davinci_application_flow_policy` binds a flow to a `pingone_davinci_application`, which is a DaVinci-native application registration distinct from the `pingone_application` an end user authenticates against. Two things about it:

- **`flow_distributions[].version` does not follow the flow's `current_version` automatically.** Updating a flow and its deploy leaves the policy pinned to the old version, and the environment carries on serving the old content with no error anywhere. This needs its own separate apply after the flow's. It is the easiest silent failure in the whole provider.
- **`trigger.configuration` is load-bearing, not decoration.** A policy declaring only `trigger.type` declares no session window, and the flow's `checkSession` node then reports "no session" for every request against a genuinely live session. The fix is a `configuration` block whose authenticator name matches the flow's own `checkSessionAuthenticator`. Nothing validates one against the other, and the mismatch produces a clean, plausible, wrong answer rather than an error.

Expose the flow policy ID as a module output and consume it as a plain reference. A hardcoded flow-policy ID copied between states is the single most reliably damaging pattern in a PingOne estate: destroy and recreate the policy and every application pointing at the old ID fails at runtime, never at plan time.

Compose modules in one state tree per environment rather than chaining repositories through `terraform_remote_state`, so a cross-module ID is an output-to-input reference resolved at plan time.

## Apply failure modes

Three distinct outcomes, which need three distinct responses:

| Symptom | What happened | Response |
| --- | --- | --- |
| `UNIQUENESS_VIOLATION` on retry after a failed create | The API created the object server-side before erroring. There is a real named object with no state entry | Check live (`pingcli pingone davinci flows list`) and delete the orphan before retrying. Do not assume a failed apply left nothing behind |
| "Provider produced inconsistent result after apply" | The API's response did not match what was configured. The resource **does** land in state, tainted | Plain re-apply. It converges with the configured value intact. Do not redesign around a rejection that did not happen |
| `ACCESS_FAILED` | Usually the environment's service list, not the role | See `pingone:core` |

The consistency-check case is worth knowing well because the instinct is to treat the error as the API refusing your configuration. Two confirmed instances: an unset flow `color`, and a subflow's `input_schema` where the API echoed back the standard authentication trigger schema instead of the configured property. In both, the stored value was correct and the create-time response was not.

### Never use `ignore_changes` to silence a provider bug

`lifecycle { ignore_changes = [properties] }` added as a workaround for an inconsistent-result error on a sensitive attribute discards every subsequent correction to that attribute before it reaches the server. A configuration can then be fixed repeatedly, correctly, and never take effect, while the state file shows exactly what you intended.

The consistency-check bug on these attributes fires on in-place *update*, not on fresh *creation*. So the remediation for a change to such a value is delete, `tofu state rm`, re-apply. Not `ignore_changes`.

## Resource notes

| Resource | Note |
| --- | --- |
| `pingone_mfa_device_policy` vs `pingone_mfa_device_policy_default` | The platform auto-creates a singleton default policy when the MFA service is enabled, and that singleton governs pairing. A separately created policy applies successfully and governs nothing. Use the `_default` resource, which manages the singleton in place and requires a one-time import |
| `pingone_image` | Accepts PNG, GIF and JPG only. Not SVG. Confirm with `tofu providers schema -json` rather than assuming |
| `pingone_application_sign_on_policy_assignment` | `sign_on_policy_id` forces replacement, despite the schema description not flagging it as immutable. Expect replace, not update in place |
| `pingone_phone_delivery_settings` | PingOne's own `${to}` / `${otp}` / `${message}` template variables need Terraform's `$$` escape (`$${to}`) to survive interpolation |
| `pingone_notification_settings_email` | Custom webhook provider and SMTP relay are two modes of one resource. Webhook mode requires `protocol = "HTTP"` alongside `custom_provider_name`; the schema rejects the combination without it |
| `pingone_mfa_device_policy` | `sms`, `email`, `totp`, `mobile` and `voice` are all required blocks. An unused channel must be present and disabled, not omitted |
| `pingone_davinci_variable` | Covers both `company` and `flowInstance` contexts. A `flow = {id}` block is only valid for `context = "flow"` and is rejected on the others |
| `pingone_key_rotation_policy` | Referenced from `oidc_options.signing.key_rotation_policy_id`. Leaving it unset silently accepts PingOne's implicit 90-day default |
| Secrets in state | A local state file holds application secrets in plaintext. Keep it out of version control |

## Guarding generated content

Where HCL consumes generated JSON, keep the HCL shallow: decode, assign, substitute the values only Terraform knows, and nothing else. Environment-specific patching belongs in the generator, so the generated artefact is already environment-correct.

The checks worth having, in order of how quietly they fail without them:

1. A `lifecycle.precondition` that no substitution placeholder survived. Nothing else can see this one.
2. Structural validation of generated JSON in CI, before plan. `tofu plan` is not a validity check on flow content.
3. An acceptance push against a disposable environment, confirming the API accepts the JSON, before the same JSON is applied anywhere real.

`tofu plan` succeeding is not evidence that a flow is correct, only that the HCL is well formed.

## Correcting this skill

This file is expected to be wrong eventually. Provider versions change resource shapes, and several statements here were confirmed against one pinned version.

When you find an instruction here that does not match observed behaviour, or you confirm a behaviour this file does not cover, run `/pingone:learn` and describe the finding. Do not silently work around a wrong instruction, and do not edit this file from memory.

Record the provider and version a finding was confirmed against. A resource shape without a version is not a durable fact, and this is the skill where that matters most.
