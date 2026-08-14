---
description: Record a confirmed PingOne, DaVinci or Terraform finding into the pingone skills, correcting a wrong instruction or adding a new one.
argument-hint: [what you found, or leave blank to work from the session]
---

# Record a finding into the pingone skills

The user has found something about PingOne, DaVinci or the Terraform providers that the skills either get wrong or do not cover. Your job is to establish whether it is real, decide where it belongs, and write it down in a form the next reader can act on.

The finding: $ARGUMENTS

If that is empty, work from what happened in this session, and say what you believe the finding is before going further.

## 1. Establish that it is real

**Do not write anything down until the finding is confirmed against a real system.** These skills are worth using only because their contents were observed rather than reasoned to, and one plausible-but-wrong entry costs more than ten missing ones. Most of what is already in them records something that looked correct and was not.

Confirmed means one of:

- Observed directly against a live tenant, flow or apply, and you can state the command or request that produced it.
- Reproduced twice, or once with a control case that isolates the cause. A control matters most for silent failures: "the silent request fails" means nothing until the interactive request through the same path is shown to work.
- Read from an authoritative primary source: the provider's own registry documentation, a provider schema dump, PingOne's published documentation, or a role definition read back from the API.

Not confirmed, and not to be written down as fact:

- A reading of the schema that has not been applied.
- Behaviour inferred from an error message. Several errors in this domain say something different from what is wrong, which is itself recorded in the skills.
- Something that worked once immediately after several other changes.

If it is plausible but unconfirmed, say so and offer to record it in `LEARNINGS.md` marked unconfirmed, without touching any `SKILL.md`. That is a legitimate outcome, not a failure.

## 2. Find the working clone

Plugins are copied into a cache when installed, so editing the installed copy achieves nothing: it is not the source, and it is discarded on the next update. Locate the source repository instead, in this order:

1. `$PINGONE_SKILLS_REPO`, if set.
2. `~/pingone-skills`.
3. Ask the user where their clone is.

Confirm it is a git repository with the expected layout (`plugins/pingone/skills/`) before editing. If no clone exists, offer to clone it from the plugin manifest's `repository` URL, and if the user declines, output the proposed change as a patch they can apply themselves rather than editing the cache.

## 3. Decide where it belongs

| Kind of finding | Goes to |
| --- | --- |
| An instruction in a `SKILL.md` is wrong | Correct that line, and add a `LEARNINGS.md` entry recording that it was wrong and what it is now |
| A behaviour no skill covers, that changes what someone should do | The relevant `SKILL.md`, plus a `LEARNINGS.md` entry |
| Detail that supports an existing rule without changing it | The relevant `reference/*.md` file |
| A one-off environment quirk, or something unconfirmed | `LEARNINGS.md` only |

Which skill:

- `core` for the tenant: environments, services, roles and scopes, users, populations, MFA devices, licences, pingcli, the Management API.
- `davinci` for flow content: the graph, nodes, connectors, variables, bindings, subflows, branching, the hosted page's HTML/CSS/JS.
- `terraform` for the providers: resource shapes, plan and drift reconciliation, import and adoption, apply failure modes.

A finding that spans two skills goes in the one whose reader will hit it, with a one-line pointer from the other. Do not duplicate the full text; duplicated rules drift apart and then contradict each other.

## 4. Write it

Match the surrounding voice. These files are terse and declarative, they lead with the consequence rather than the mechanism, and they say what to do rather than describing the platform.

Requirements:

- **Nothing tenant-specific.** No environment IDs, client IDs, secrets, hostnames, organisation names, brand names, or internal project names. This repository is public. Generalise the finding; if it cannot be stated without an internal specific, it does not go in.
- **Say what fails and how loudly.** A finding that fails silently is worth far more than one that errors, and readers triage on that. Say which it is.
- **Version the volatile things.** A provider resource shape without a provider version is not a durable fact. A pingcli command without a version is only slightly better.
- **Correct in place, do not append a correction.** If a line is wrong, rewrite that line. Do not leave the wrong text with a note beneath it saying it was wrong. `LEARNINGS.md` is where the history lives.
- **No session narration.** The skill files say what is true. They never say what an earlier draft said, or that something was corrected, or how the finding was arrived at during a conversation. That belongs in `LEARNINGS.md` and in your reply.

Then prepend an entry to `LEARNINGS.md`, newest first:

```markdown
## YYYY-MM-DD - one-line summary

**Skill:** core | davinci | terraform (or none)
**Confirmed by:** what was actually run or read
**Versions:** pingcli x.y.z / provider x.y.z / n/a

What was believed before, if anything, and what is true instead. What the failure looks like
and whether it is silent. What changed in the skill files, or why nothing did.
```

## 5. Finish

- Bump the patch version in both `plugins/pingone/.claude-plugin/plugin.json` and the plugin's entry in `.claude-plugin/marketplace.json`. They must match, and users only receive updates when the version changes.
- Run `claude plugin validate ./plugins/pingone` from the repository root if the CLI is available.
- Commit on a branch. Show the user the diff and the proposed commit message.

**Do not push, and do not open a pull request, without the user explicitly approving that specific action.** Publishing to a shared repository is theirs to authorise, not yours to infer from having been asked to record a finding.

Then tell the user, briefly: what was confirmed and how, which files changed, and what you deliberately left out.
