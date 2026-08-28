# Learnings journal

Newest first. One entry per finding.

This file is the history; the `SKILL.md` files are the current state. When a skill is corrected, the wrong instruction is rewritten in place and the fact that it was wrong is recorded here. That split is deliberate: a reader following a skill should not have to work out which of two competing statements is current, and a maintainer deciding whether to trust a rule should be able to see how it was established.

Entries are added by `/pingone:learn`, which will not write one until the finding is confirmed against a real system. Unconfirmed material is welcome here, marked as such, and does not go into a skill.

## 2026-08-28 - a screen is painted once, and `disabled` on an skbutton is the widget's

**Skill:** davinci
**Confirmed by:** A hosted screen whose flow looped back to the same `customHTMLTemplate` node. The loop demonstrably re-executed the node — its connector call ran and re-issued a one-time code, and the new code validated where the previous one no longer did — while an element marked with a `data-` attribute before the loop still carried that mark 45 seconds later. Counters written into the DOM from the screen's own script survived the round trip with their values continuing from the first execution rather than restarting, so the script did not run a second time. In the same trace, the script's click handler fired, set `disabled` on an skbutton, and the button read enabled nine seconds later with no path in the script having cleared it; the same script's first arming of that state, before any submission, held correctly for its full 60 seconds. Company-variable substitution was confirmed by a control in the opposite direction: the screen offered a control only when a company-scoped limit read greater than zero, and the control was offered.
**Versions:** n/a - widget and flow-runtime behaviour, not a provider or CLI

The skills said the error re-render patches only the error region and does not re-run `customScript`. That was true and too narrow: it is not a property of error re-renders. **Any** re-entry to the same screen node leaves the DOM alone and the script un-run, so a screen gets exactly one execution for its whole life. Rewrote that line to point at the general rule and added the rule to "The HTML/CSS/JS surface".

The practical cost of the narrow version is that anything armed at render — a countdown, a disabled control, a rate limit — covers the first pass and no later one, while looking correct on the pass anyone tests first. Nothing errors and the screen keeps showing what it showed.

Also added: a script-set `disabled` on an element carrying `data-skcomponent="skbutton"` does not survive a submission. The widget owns that property on its own components and restores it, so a control gated that way looks gated in the DOM and refuses nothing. Gate with a class on your own container plus a `document`-captured click listener instead, and drive the visuals from the same class. `pointer-events: none` is worth having and is not the gate — it stops a pointer, not a keyboard and not `.click()`. What the widget does not touch is your own container class, your `data-` attributes, and a `hidden` your script cleared.

Third, smaller: `{{global.company.variables.<name>}}` resolves inside a subflow, including inside a screen's `customScript`. The skill's subflow section warned only about `global.variables`, which left it open whether the company namespace was reachable from a subflow at all. Added one line next to that warning.

Deliberately left out: the mechanism behind the `disabled` restoration. That the widget re-enables its buttons when a submission's response lands is the obvious reading and was not observed directly — what was observed is that a script-set value does not survive the round trip, which is what a reader needs either way.

## 2026-08-27 - a node with no inbound edge is an entry point, and an unwired terminal returns on load

**Skill:** davinci
**Confirmed by:** A subflow generated with 12 of its edges missing (91 nodes / 71 edges) returned an empty payload to its caller and terminated with no screen rendered; the same flow with those edges present (91 / 83) rendered its screen and completed. Reproduced for two different users from a cleared browser state, and the corrected flow verified end to end afterwards.
**Versions:** n/a - graph semantics, not a provider or CLI behaviour

The skills covered `startNode` needing no inbound edge, which reads as if being unwired is a property of that node type. It is not: DaVinci treats *every* node with no inbound edge as a place the flow starts, and `startNode` is simply the one where that is intended.

The consequence is worst for terminals. An unwired `createSuccessResponse` fires on entry and returns before anything has run, so a caller gets a blank claim and every downstream comparison silently takes its default branch. Nothing errors, the graph is valid, and `terraform plan` and `apply` both succeed. The flow ends on a real, named outcome that has nothing to do with the actual failure, so the redirect points the reader at whichever router defaulted rather than at the disconnected node.

Added the rule to "The graph model" in `davinci/SKILL.md`, next to the existing edge-level rules, with a pointer from the `startNode` line in "Teleports" so the two do not contradict each other. Deliberately left out: the specific generator bug that dropped the edges, which was local tooling rather than anything about DaVinci.

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
