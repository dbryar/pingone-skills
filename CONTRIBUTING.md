# Contributing

The value of these skills is that their contents were observed rather than reasoned to. Everything below protects that.

## The bar

A finding goes into a `SKILL.md` only once it is confirmed. Confirmed means one of:

- **Observed directly** against a live tenant, flow or apply, and you can state the command or request that produced it.
- **Reproduced twice**, or once with a control case that isolates the cause. A control matters most for silent failures: "the silent request fails" means nothing until the interactive request through the same path is shown to work.
- **Read from an authoritative primary source**: the provider's registry documentation, a `tofu providers schema -json` dump, PingOne's published documentation, or a role definition read back from the API.

Not confirmed:

- A reading of the schema that has not been applied. The schema marks a great deal optional that the API requires.
- Behaviour inferred from an error message. Several errors in this domain name the wrong cause, which is itself recorded in the skills.
- Something that worked once immediately after several other changes.

Unconfirmed findings are welcome in `LEARNINGS.md`, marked as such. They do not go into a skill.

## The easy path

From a clone of this repository, with the plugin installed:

```
/pingone:learn <describe what you found>
```

It applies the procedure below, including the version bump and the journal entry, and stops before pushing.

## By hand

1. **Pick the file.**

   | Finding | File |
   | --- | --- |
   | An existing instruction is wrong | Correct that line in its `SKILL.md` |
   | A behaviour no skill covers, that changes what someone should do | The relevant `SKILL.md` |
   | Detail supporting an existing rule without changing it | That skill's `reference/*.md` |
   | A one-off quirk, or anything unconfirmed | `LEARNINGS.md` only |

   `core` is the tenant, `davinci` is flow content, `terraform` is the providers. A finding spanning two goes in the one whose reader will hit it, with a one-line pointer from the other. Do not duplicate the full text; duplicated rules drift apart and then contradict each other.

2. **Write it in the surrounding voice.** Terse, declarative, consequence first. Say what to do, not how the platform is designed.

3. **Say how loudly it fails.** A silent failure is worth far more than one that errors, and readers triage on that. Say which it is.

4. **Version anything volatile.** A resource shape without a provider version is not a durable fact.

5. **Correct in place.** Rewrite the wrong line. Do not leave it standing with a note beneath saying it was wrong. `LEARNINGS.md` is where history lives, so a reader following a skill never has to work out which of two competing statements is current.

6. **Keep it general.** No environment IDs, client IDs, secrets, hostnames, organisation names, brand names or internal project names. This repository is public. If a finding cannot be stated without an internal specific, it does not go in.

7. **Add a `LEARNINGS.md` entry**, newest first:

   ```markdown
   ## YYYY-MM-DD - one-line summary

   **Skill:** core | davinci | terraform (or none)
   **Confirmed by:** what was actually run or read
   **Versions:** pingcli x.y.z / provider x.y.z / n/a

   What was believed before, if anything, and what is true instead. What the failure looks like
   and whether it is silent. What changed in the skill files, or why nothing did.
   ```

8. **Bump the version** in both `plugins/pingone/.claude-plugin/plugin.json` and the plugin's entry in `.claude-plugin/marketplace.json`. They must match, and users only receive updates when the version changes. Patch for a correction or addition, minor for a new reference file or a restructure, major for a change that breaks how the skills are invoked.

9. **Add a `CHANGELOG.md` line.**

## Validating

```bash
claude plugin validate ./plugins/pingone
```

To test a change before publishing, add your clone as a local marketplace:

```
/plugin marketplace add ~/pingone-skills
/plugin install pingone@pingone-skills
```

Note that installing copies the plugin into a cache. Edit the clone and re-run `/plugin marketplace update pingone-skills`, not the cached copy.

## What does not belong here

- Product documentation that is accurate and easy to find. These skills are for what the documentation does not say, or says wrongly.
- PingFederate, PingAccess, PingDirectory, PingID, or PingOne Advanced Identity Cloud.
- Anything tenant-specific, per point 6.
