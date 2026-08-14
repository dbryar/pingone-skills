---
name: core
description: Operate a PingOne tenant through pingcli and the Management API - environments, services, worker applications, roles and scopes, users, populations, MFA devices, and licences. Use whenever a task touches a PingOne environment directly rather than through Terraform, when diagnosing a PingOne API error (ACCESS_FAILED, INVALID_DATA, UNIQUENESS_VIOLATION), or when setting up authentication for pingcli. Load this before pingone:davinci or pingone:terraform, since both assume its authentication and permission model.
---

# PingOne platform operations

This skill covers the PingOne tenant itself: how to authenticate against it, what gates each API, and which of its behaviours are undocumented or actively misleading. It is written from findings confirmed against live tenants, not from the product documentation alone. Where the two disagree, the file says so.

Companion skills: `pingone:davinci` for flow authoring, `pingone:terraform` for managing either through HCL.

## The one rule that saves the most time

**When a PingOne API call fails with a permission error, check the environment's enabled services before you touch roles.**

PingOne gates whole API families on the environment having the relevant *service* enabled, and it reports that gate as a permission failure. An environment without the `MFA` service returns `ACCESS_FAILED: You do not have permissions or are not licensed to make this request` on `CreateDeviceAuthenticationPolicies`, no matter what roles the calling identity holds. The message names permissions and licensing; the actual cause is neither.

Order of investigation for any `ACCESS_FAILED`:

1. Is the service enabled on the environment (`SSO`, `MFA`, `DaVinci`, `Risk`, `Verify`, `Credentials`, `Authorize`)?
2. Does the calling identity hold the role?
3. Is the role held at the right *scope* (organisation, environment, population)?
4. Is the role capable of being held at that scope at all? See "Roles are not uniformly scopable" below.

Only after all four should you consider it a licensing problem.

## Authenticating pingcli

Every pingcli configuration key is settable by environment variable as `PINGCLI_<PATH_IN_UPPER_SNAKE_CASE>`, with shorter `PINGONE_*` aliases honoured for common settings. Prefer environment variables over a `config profiles` file for anything a machine runs, so nothing depends on a persisted file that CI will not have.

```bash
export PINGCLI_PINGONE_ENABLED=true
export PINGCLI_PINGONE_AUTHENTICATION_TYPE=client_credentials
export PINGCLI_PINGONE_CLIENT_CREDENTIALS_CLIENT_ID="$CLIENT_ID"
export PINGCLI_PINGONE_CLIENT_CREDENTIALS_CLIENT_SECRET="$CLIENT_SECRET"
export PINGCLI_PINGONE_ENVIRONMENT_ID="$ENVIRONMENT_ID"
export PINGCLI_PINGONE_ROOT_DOMAIN="pingone.com"   # domain form, see below

pingcli auth login
pingcli auth status
```

Two things reliably go wrong here.

**The worker application must use `CLIENT_SECRET_BASIC`.** A worker application configured with `token_endpoint_auth_method = "CLIENT_SECRET_POST"` is accepted by PingOne without complaint and works for direct API calls, but pingcli's client-credentials flow rejects it with `invalid_client: Unsupported authentication method`. This is not documented in either PingOne's or Ping CLI's published material. Set `CLIENT_SECRET_BASIC` explicitly.

**Root domain and region code are different vocabularies for the same fact.** pingcli's `ROOT_DOMAIN` wants the domain form (`pingone.com`, `pingone.eu`, `pingone.ca`, `pingone.asia`, `pingone.com.au`, `pingone.sg`). The Terraform provider's `region_code` wants an enum (`NA`, `EU`, `CA`, `AP`, `AU`, `SG`). They do not match string-for-string and neither accepts the other's form. If a script handles both, translate explicitly rather than passing one through to the other.

For a human authenticating as themselves rather than as a machine, a named `pingcli` config profile against an interactive OIDC client (authorization code, public client, `http://127.0.0.1:<port>/callback`) is the right shape. `localhost` and `127.0.0.1` redirect URIs are valid in PingOne; it does not require HTTPS for them.

## Roles, scopes, and worker applications

Grant roles one at a time, each against a documented need, and re-test after each. Blanket administrator grants hide which capability was actually required, and you will need to know that the next time you build an environment.

Findings worth knowing before you start:

- **Creating an environment with a licence attached needs `Organization Admin`, not just `Environment Admin` at organisation scope.** The failure is `ACCESS_FAILED` / `INSUFFICIENT_PERMISSIONS` on `CreateEnvironmentActiveLicense`, which reads like a licensing problem and is not.
- **Roles are not uniformly scopable.** Each role definition declares its own `applicableTo` set. `Identity Data Admin`, for instance, declares `["ENVIRONMENT", "POPULATION"]` with `ORGANIZATION` absent, so no identity in any PingOne tenant can hold it organisation-wide. Check before assuming a scope is available:
  ```bash
  pingcli pingone roles list --environment-id <any-env-id> \
    -O json --query "data[?name=='Identity Data Admin'].id | [0]"
  pingcli pingone roles get --role-id <role-id>
  ```
  Roles are organisation-wide objects. The `--environment-id` flag on `roles list` is required by the command but does not scope the result.
- **An identity can only grant a role at a scope it holds that role at itself.** If a grant keeps coming out environment-scoped when you asked for something broader, this is usually why.
- **Managing group role assignments is gated by `Identity Data Admin` scoped to the environment that hosts the group**, not by `Environment Admin` or `Organization Admin`. Holding both of those and still getting `403` on `groups role-assignments list` is the signature.
- **`--environment-id` on `groups role-assignments create` is where the group lives. The target of the grant is `scope.id` inside the body.** These are frequently different environments, and swapping them produces a grant that looks plausible and does nothing you wanted.

  ```bash
  cat <<JSON | pingcli pingone groups role-assignments create \
    --environment-id <environment-hosting-the-group> \
    --group-id <group-id> --from-file -
  { "role": { "id": "<role-id>" },
    "scope": { "id": "<target-environment-id>", "type": "ENVIRONMENT" } }
  JSON
  ```

**Never manage the administrators environment with Terraform.** A mistake there breaks administrative access to the whole organisation rather than to one environment. Bootstrap credentials and administrator access belong to hand-run, one-change-at-a-time `pingcli` calls, recorded in a document, because they cannot be discovered by reading HCL.

## Creating environments

- **Multiple `ACTIVE` licences on an organisation is normal.** Do not index into the first result. Fail loudly and require an explicit licence ID when more than one comes back. When choosing, the licence with the largest environment allowance and highest user count is usually the general-purpose one; a lighter tier is usually reserved for something else.
- **Service tags applied at creation cannot be changed later.** The DaVinci service accepts a `DAVINCI_MINIMAL` tag that suppresses the demo flows and connectors a fresh environment otherwise inherits. It also suppresses the auto-provisioned "PingOne DaVinci Connection" application that every PingOne-category DaVinci connector instance needs in order to call back into its own environment. That application cannot be added retroactively by flipping the tag. If you use `DAVINCI_MINIMAL`, plan to provision that application yourself. See `pingone:davinci`.
- **Enable every service the environment will need up front**, for the reason in "The one rule" above. Adding a service later is an in-place update, not a replacement, so this is cheap to correct, but only once you know that is what you are looking at.
- **Set an explicit key rotation policy on applications.** PingOne silently attaches a default 90-day rotation policy to any application that does not name one. An explicit policy is a value you can see and rotate deliberately. PingOne is making this mandatory from 02-03-2027.

## Field validation traps

**Description fields reject nine specific characters.** `$ + < = > ^ \` | ~` produce `INVALID_DATA` with the message "must contain only Unicode letters, marks, numbers, spaces, or punctuation except ...". Confirmed on `pingone_population.description` and `pingone_application.description`; assume it applies to description fields generally. Parentheses, full stops, commas and hyphens are fine. This bites hardest when a description is generated from a filter expression or a code snippet, because `=` is the natural character to use there.

## MFA and device pairing

**PingOne auto-creates a singleton default MFA device policy the moment the MFA service is enabled, and that singleton is what actually governs device pairing.** A separately created policy sits alongside it doing nothing. The symptom is a pairing attempt failing with "Pairing device type TOTP is not allowed" while the policy you created plainly shows `totp.enabled = true`. Manage the singleton in place; see `pingone:terraform` for the resource that does this.

**MFA only fires because a sign-on policy explicitly invokes it.** A device policy plus a paired device does nothing on its own. The sign-on policy needs an `mfa` action after the `login` action.

Device pairing through the Management API has three undocumented shapes:

| Device | Request | Trap |
| --- | --- | --- |
| SMS | `POST /users/{id}/devices` with `{"type": "SMS", "phone": "+61..."}` | `phone` is a bare string. A nested `{"phone": {"number": ...}}` is rejected. The user also needs a real `mobilePhone` on their profile first. |
| Email | `POST /users/{id}/devices` with `{"type": "EMAIL", "email": "..."}` | The endpoint does not infer the address from the user's profile. Omitting `email` fails with "email must not be blank". |
| TOTP | `POST /users/{id}/devices` with `{"type": "TOTP"}`, then activate with an RFC 6238 code | Activation requires `Content-Type: application/vnd.pingidentity.device.activate+json` or the API returns `415`. |

App-bound authenticators (push, biometric) cannot be paired by API call. They need a real registered mobile application and a real device.

## Driving authentication headlessly

`response_mode=pi.flow` on `/as/authorize` returns the authentication flow as JSON instead of redirecting, which is how you script a login end to end without a browser.

Useful states and their meanings, all confirmed live:

- A `usernamePassword.check` submission for a user whose sign-on policy invokes MFA but who has no paired device returns `status: "FAILED"`, `error.code: "NO_USABLE_DEVICES"`.
- The same submission for a user with an `ACTIVE` paired device reaches `status: "OTP_REQUIRED"` with the device identified.
- A user whose policy has no `mfa` action completes single-factor regardless of what devices they hold.

Keep a cookie jar across the whole sequence. Ad hoc `curl` calls that each start fresh fail at the OTP step with a session mismatch that looks like a configuration defect and is not.

**Respect the lockout policy when probing.** Environments commonly lock at five failed attempts per fifteen minutes. Debugging a credential problem by repeated login attempts turns a diagnosable problem into a locked account plus a wait.

## Error vocabulary

| Code | Usual meaning | First thing to check |
| --- | --- | --- |
| `ACCESS_FAILED` | Service not enabled, or role missing, or role at wrong scope | Environment's `services` list, before roles |
| `INSUFFICIENT_PERMISSIONS` | Genuine role gap | Which scope the role is held at |
| `INVALID_DATA` | Field validation | Restricted characters in a description; wrong body shape |
| `UNIQUENESS_VIOLATION` | An object with that name already exists | An orphan left by a failed create. See `pingone:terraform` |
| `unexpectedError` (DaVinci) | Opaque server-side failure | Almost never diagnosable from the message. See `pingone:davinci` |

## Environment limits

- **100 flows per environment**, counting main flows and subflows together. An applied but undeployed flow still counts. Test harnesses that create a flow per case exhaust this fast.
- Custom risk predictors are gated on a Risk/Protect entitlement. An environment already carrying the built-in predictors will refuse a custom one with `ACCESS_FAILED: The request exceeded your license limit`.

## Reference

- [`reference/pingcli.md`](reference/pingcli.md) - command catalogue, output shaping with `-O json --query`, raw API passthrough.
- [`reference/permissions.md`](reference/permissions.md) - role catalogue, scope model, and the worker application shapes that work.

## Correcting this skill

This file is expected to be wrong eventually. PingOne changes, and several statements here were true only of the tenant they were confirmed against.

When you find an instruction here that does not match observed behaviour, or you confirm a behaviour this file does not cover, run `/pingone:learn` and describe the finding. Do not silently work around a wrong instruction, and do not edit this file from memory. The command's procedure requires the finding to be reproducible before it is written down, which is the property that makes this file worth trusting.

Two things specifically do not belong in this file: anything tenant-specific (environment IDs, client IDs, domains, organisation names), and anything you have reasoned to but not observed. Mark unconfirmed material as unconfirmed or leave it out.
