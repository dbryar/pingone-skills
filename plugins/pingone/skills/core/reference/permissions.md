# Roles, scopes and worker applications

## The scope model

Every role assignment has three parts: the identity holding it, the role, and the scope it is held at. Scope is one of `ORGANIZATION`, `ENVIRONMENT` or `POPULATION`.

Two constraints govern what is actually grantable:

1. **A role can only be held at scopes its own definition allows.** Read `applicableTo` from `pingcli pingone roles get --role-id <id>`. If `ORGANIZATION` is absent, no identity in any PingOne tenant can hold that role organisation-wide, and no support ticket will change that.
2. **An identity can only grant a role at a scope it already holds that role at.** This is why grants made by a partially privileged administrator keep coming out narrower than asked for.

## Roles seen to matter, and what each actually unlocks

| Role | Unlocks | Notes |
| --- | --- | --- |
| `Organization Admin` | Creating an environment *with a licence attached* | `Environment Admin` at organisation scope is not sufficient for this. The failure is `INSUFFICIENT_PERMISSIONS` on `CreateEnvironmentActiveLicense` |
| `Environment Admin` | Most resource administration inside an environment | Does not cover directory data operations |
| `Identity Data Admin` | Creating and reading users; managing group role assignments | Cannot be held at `ORGANIZATION` scope. Managing a group's role assignments requires this role scoped to the environment *hosting the group* |
| `DaVinci Admin` | Administering DaVinci flows, connectors, applications | Sufficient together with `Environment Admin` for full flow administration through pingcli. Confirmed, not assumed |

Add roles one at a time against a specific need and re-test after each. A blanket grant works, and leaves nobody able to say which capability was actually required when the next environment is built.

## Worker application shape

For a machine credential that pingcli and Terraform will both use:

- Type `WORKER`
- Grant `CLIENT_CREDENTIALS`
- **`token_endpoint_auth_method = "CLIENT_SECRET_BASIC"`.** `CLIENT_SECRET_POST` is accepted by PingOne and rejected by pingcli's login with `invalid_client: Unsupported authentication method`
- Roles assigned with `scope_environment_id`, not `scope_organization_id`, unless the credential genuinely needs to act across environments
- An explicit key rotation policy, so the implicit 90-day default is not what is signing your tokens

## The bootstrap problem

An environment-scoped worker application cannot exist before its environment does, so creating the first environment needs an organisation-scoped credential created by hand in the console. That credential is a permanent manual object outside any Terraform state.

Two consequences:

- **Document it.** Client ID, application type, grants, and which roles it holds. It is otherwise undiscoverable by reading code, and the next engineer will assume it does not exist.
- **Stop using it once the environment exists.** Create an environment-scoped operational worker inside the new environment and point everything ongoing at that instead. The bootstrap credential's job is one call.

## The administrators environment

Never manage it with Terraform, and never manage it with a script that runs unattended. A mistake in an ordinary environment costs that environment. A mistake in the administrators environment costs administrative access to the organisation.

Changes there are made by hand, one at a time, by a person reading the result. Record what was changed in a document, because none of it is visible in state.

## Human interactive access

A worker application authenticates as itself. It cannot answer "can a person on the team see this environment's users". That needs either a role granted to a group the person belongs to, or a separate interactive OIDC client:

- Grants: `Authorization Code` (plus `Implicit` if the tooling wants it)
- Public client, no token endpoint authentication
- Redirect `http://127.0.0.1:<port>/callback`

`localhost` and `127.0.0.1` are valid PingOne redirect URIs. PingOne does not require HTTPS for loopback.

Grant the role to the *group*, not to individuals, so membership rather than a repeated manual step controls who can see what. One grant is needed per environment, since the roles that matter here cannot be held organisation-wide.
