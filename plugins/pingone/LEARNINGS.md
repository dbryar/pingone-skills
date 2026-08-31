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
