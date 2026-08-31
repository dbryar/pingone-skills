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
