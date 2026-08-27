# Learnings journal

Newest first. One entry per finding.

This file is the history; the `SKILL.md` files are the current state. When a skill is corrected, the wrong instruction is rewritten in place and the fact that it was wrong is recorded here. That split is deliberate: a reader following a skill should not have to work out which of two competing statements is current, and a maintainer deciding whether to trust a rule should be able to see how it was established.

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
