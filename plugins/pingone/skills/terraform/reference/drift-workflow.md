# Drift reconciliation workflow

The problem this solves: a solution gets built by a mixture of console clicks, `pingcli` calls and HCL, and nobody can say which of the three a given piece of live configuration came from. Terraform's plan is the instrument that answers that, if the loop is run deliberately.

Commands below use `tofu`; `terraform` is interchangeable for all of them.

## Establish a baseline

```bash
tofu plan -detailed-exitcode
```

Exit codes: `0` no changes, `1` error, `2` changes present. This makes "is the baseline clean" a scriptable question rather than a judgement about how the output looks.

If the baseline is not clean, resolve it before making any hand change. Every count afterwards is otherwise a mixture of your change and whatever was already outstanding.

## Capture the delta

Make the change by hand, then:

```bash
tofu plan -out=tfplan
tofu show -json tfplan | jq -r '
  .resource_changes[]
  | select(.change.actions != ["no-op"])
  | "\(.change.actions | join(",")) \(.address)"
'
```

That gives one line per changed resource with its action. For the attribute-level detail on a single resource:

```bash
tofu show -json tfplan | jq '
  .resource_changes[] | select(.address == "module.x.pingone_application.y") | .change
'
```

Read the attribute diff, not just the count. The platform routinely fills in defaults you did not set, and a plan showing "1 to change" on a resource you only touched one field of usually has more than one field in the diff.

## Reconcile

**Object you created by hand and want to keep.** Declare it, then adopt it.

```hcl
import {
  to = pingone_application.thing
  id = "<environment-id>/<application-id>"
}
```

`tofu plan` after adding the block shows what the declaration would change about the live object. Iterate on the HCL until that is empty, *then* apply. Once adopted, remove the `import` block. `tofu import` does the same thing imperatively if you prefer; the block form is reviewable and lands in the commit.

Import ID formats vary per resource and are documented per resource on the provider registry page. Most PingOne resources are `<environment-id>/<resource-id>`; some are deeper.

**Object you created by hand as a throwaway.** Delete it live, declare it in HCL, apply. Cheaper than importing, and it proves the HCL creates the right thing.

**Attribute changed by hand.** Change the code to match, confirm the plan is empty. If the value was one the platform filled in rather than one you chose, set it explicitly anyway. An unset attribute whose value comes from an API-side default is invisible, and it will change under you at some point.

**Drift you cannot explain.** Confirm through `pingcli` whether the object is really live before assuming the state file is stale, and vice versa. These are different problems with opposite fixes. Leave it alone and scope your apply around it until you know which it is.

## Scoping an apply around unexplained drift

```bash
tofu apply -refresh=false \
  -target=module.apps.pingone_application.a \
  -target=module.apps.pingone_application_resource_grant.a
```

Include every resource the change touches. A partially targeted apply leaves the configuration half-applied, which is worse than not applying.

Record what was scoped out and why. A targeted apply is a deliberate exception, and an undocumented one looks identical to a mistake six weeks later.

## Restructuring

```hcl
moved {
  from = pingone_application.thing
  to   = module.apps.pingone_application.thing
}
```

Confirm the plan reports the resources as moved with zero attribute diff and zero destroyed before applying. Check every `path.module`-relative `file()` call for a changed `../` depth in the same pass.

## Removing

```bash
tofu state list | grep <thing>
```

Then, in one commit: remove the resource block and remove its state entry. Read the plan. Apply. Splitting these across two changes destroys the live object or duplicates it, depending which half goes first.

## Verifying against reality rather than against state

A clean plan proves the code and the state file agree, and that a refresh found the objects it knows about unchanged. It does not prove nothing else exists.

For anything created outside Terraform, or for confirming a failed apply left no orphan:

```bash
pingcli pingone applications list --environment-id "$ENV" -O json --query "data[].name"
pingcli pingone davinci flows list --environment-id "$ENV" -O json --query "data[].name"
pingcli pingone davinci connector-instances list --environment-id "$ENV"
```

An orphaned object left behind by a failed create is invisible to `plan` and blocks the retry with `UNIQUENESS_VIOLATION`.
