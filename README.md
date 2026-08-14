# PingOne skills for Claude Code

Three skills that give a coding agent working knowledge of PingOne, PingOne DaVinci, and the Terraform/OpenTofu providers that manage both.

The contents are not a summary of the product documentation. They are the things that had to be found the hard way: the API errors that name the wrong cause, the resource attributes the schema marks optional and the API requires, and the several DaVinci behaviours that apply cleanly, report success, and do the wrong thing.

## Install

```
/plugin marketplace add dbryar/pingone-skills
/plugin install pingone@pingone-skills
```

If the install summary says `Run /reload-plugins to activate.`, run that.

Once installed, three skills are available and Claude will load whichever fits the task:

| Skill | Covers |
| --- | --- |
| `pingone:core` | The tenant. Environments and services, worker applications, roles and scopes, users and populations, MFA device pairing, licences, pingcli, the Management API |
| `pingone:davinci` | Flow content. The graph model, node and connector shapes, variable contexts and bindings, subflow contracts, branching, error handling, and the HTML/CSS/JS of hosted login pages |
| `pingone:terraform` | Both, as code. Provider resource shapes, using plan output to reconcile hand-made changes back into HCL, adopting existing resources, and the apply failure modes that leave orphans or silently stale content |

You can also invoke one directly, for instance `/pingone:davinci`, when you want it loaded before describing the task.

## What each skill is for

**`pingone:core`** answers "why did this API call fail". Its central rule is that PingOne reports a missing *service* on an environment as a permission error, so investigating roles first is usually a wasted hour. It also covers the pingcli authentication shape that works, which is not the one the documentation implies, and the field validation and device pairing quirks that have no published description.

**`pingone:davinci`** answers "why does this flow behave differently from how it reads". Almost every entry is a silent failure: a subflow input bound the way it obviously should be, which compares against nothing forever; a session window in minutes sitting next to a cookie expiry in seconds; an error node that is a correct dead end in one position and kills the whole authorisation request in another. It also covers the working method these were found by, which matters more than any single entry.

**`pingone:terraform`** answers two questions. What shape does this resource actually take, and how do I get the code and the live environment back into agreement.

The second is the one worth reading even if you know the provider. A plan with zero changes is an assertion that the code matches reality, which makes the plan the instrument for a workflow where people change things in the console, agents change things through the CLI, and everything has to end up in the code. The skill sets out that loop: baseline, change, read the diff rather than the count, sort the diff into drift you caused, drift you did not, defaults the platform filled in, and changes you intended, then write HCL until the plan is empty again.

## Keeping the skills correct

These skills go out of date. PingOne changes, provider versions change resource shapes, and some of what is recorded was true of one tenant.

The repository carries a mechanism for that rather than an intention to remember:

- **Every skill ends with a "Correcting this skill" section** telling the reader not to work around a wrong instruction silently.
- **`/pingone:learn`** turns a finding into an edit. It insists the finding be confirmed against a real system before anything is written down, finds your working clone rather than the read-only installed copy, decides whether the finding belongs in a skill or only in the journal, makes the edit in the skill's own voice, records the history, and bumps the version. It stops short of pushing.
- **[`plugins/pingone/LEARNINGS.md`](plugins/pingone/LEARNINGS.md)** is the journal. Skills hold the current state; the journal holds what was believed before and how each rule was established.

```
/pingone:learn the flow policy version pin does follow current_version now, as of provider 1.24
```

The bar for writing something down is deliberately high. Most entries in these skills record something that looked correct and was not, so one plausible-but-unconfirmed addition costs more than ten missing ones. `/pingone:learn` will decline to touch a skill for an unconfirmed finding and will offer the journal instead.

To contribute a finding back, see [CONTRIBUTING.md](CONTRIBUTING.md).

## Repository layout

```
pingone-skills/
├── .claude-plugin/
│   └── marketplace.json          marketplace catalogue
└── plugins/
    └── pingone/
        ├── .claude-plugin/
        │   └── plugin.json       plugin manifest
        ├── skills/
        │   ├── core/
        │   │   ├── SKILL.md
        │   │   └── reference/    pingcli commands, roles and scopes
        │   ├── davinci/
        │   │   ├── SKILL.md
        │   │   └── reference/    flow JSON shapes, connector catalogue
        │   └── terraform/
        │       ├── SKILL.md
        │       └── reference/    drift reconciliation commands
        ├── commands/
        │   └── learn.md          /pingone:learn
        ├── LEARNINGS.md
        └── CHANGELOG.md
```

Each `SKILL.md` is the decision layer: what to do, and what will bite you. The `reference/` files carry the command catalogues and shape tables, and are read only when the skill points at them.

## Scope

Deliberately general. Nothing here is specific to one tenant, organisation or estate, and findings are generalised before they are recorded. No environment IDs, client IDs, hostnames or organisation names appear anywhere in the repository, and `/pingone:learn` will not add them.

Deliberately excluded: PingFederate, PingAccess, PingDirectory, PingID, and PingOne Advanced Identity Cloud. These skills cover PingOne and DaVinci only.

## Versions

Findings were confirmed against pingcli 1.3.0 and the `pingidentity/pingone` Terraform provider `~> 1.21`, except where an entry says otherwise. The DaVinci sections also describe the older, separate `davinci` provider where the two differ, because guidance written for one is actively wrong for the other.

## Licence

MIT. See [LICENSE](LICENSE).
