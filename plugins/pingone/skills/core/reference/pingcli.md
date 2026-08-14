# pingcli reference

Confirmed against pingcli v1.3.0. Command surfaces move between versions; check `--help` before trusting a subcommand path from this file.

## Configuration

Every configuration key is settable as `PINGCLI_<PATH_IN_UPPER_SNAKE_CASE>`. Shorter `PINGONE_*` aliases exist for common settings. The full key list is in `pingcli config --help`.

| Variable | Value |
| --- | --- |
| `PINGCLI_PINGONE_ENABLED` | `true` |
| `PINGCLI_PINGONE_AUTHENTICATION_TYPE` | `client_credentials` or `worker` |
| `PINGCLI_PINGONE_CLIENT_CREDENTIALS_CLIENT_ID` | Worker application client ID |
| `PINGCLI_PINGONE_CLIENT_CREDENTIALS_CLIENT_SECRET` | Worker application secret |
| `PINGCLI_PINGONE_ENVIRONMENT_ID` | Environment the credential is scoped to |
| `PINGCLI_PINGONE_ROOT_DOMAIN` | `pingone.com`, `pingone.eu`, `pingone.ca`, `pingone.asia`, `pingone.com.au`, `pingone.sg` |

Named profiles (`pingcli config profiles`) suit a human's interactive login. Environment variables suit anything a machine runs, including CI, because nothing then depends on a file existing.

## Output shaping

`-O json` plus `--query` (JMESPath) is how you get a single value out without piping through `jq`:

```bash
pingcli pingone roles list --environment-id "$ENV" \
  -O json --query "data[?name=='Identity Data Admin'].id | [0]"

pingcli pingone applications list --environment-id "$ENV" \
  -O json --query "data[].{name: name, id: id, type: type}"
```

## Raw Management API passthrough

`pingcli pingone api` issues an authenticated call against any Management API path. This is the escape hatch for everything the typed subcommands do not cover, and it is where most real diagnostic work happens.

```bash
pingcli pingone api GET "/environments/$ENV/users/$USER_ID/devices"

pingcli pingone api POST "/environments/$ENV/users/$USER_ID/devices" \
  --data '{"type": "SMS", "phone": "+61400000000"}'

pingcli pingone api PATCH "/environments/$ENV/users/$USER_ID" \
  --data '{"mobilePhone": "+61400000000"}'

pingcli pingone api DELETE "/environments/$ENV/deviceAuthenticationPolicies/$POLICY_ID"
```

Content-type sensitive calls need the header set explicitly. TOTP device activation is the common case:

```bash
pingcli pingone api POST "/environments/$ENV/users/$USER_ID/devices/$DEVICE_ID" \
  --header "Content-Type: application/vnd.pingidentity.device.activate+json" \
  --data '{"otp": "123456"}'
```

## Common commands

```bash
# Auth
pingcli auth login
pingcli auth status

# Environment inventory
pingcli pingone applications list --environment-id "$ENV"
pingcli pingone populations list --environment-id "$ENV"
pingcli pingone users list --environment-id "$ENV"

# Roles and assignments
pingcli pingone roles list --environment-id "$ENV"
pingcli pingone roles get --role-id "$ROLE_ID"
pingcli pingone groups role-assignments list --environment-id "$ADMIN_ENV" --group-id "$GROUP"
pingcli pingone groups role-assignments create --environment-id "$ADMIN_ENV" --group-id "$GROUP" --from-file -
pingcli pingone groups role-assignments delete --environment-id "$ADMIN_ENV" --group-id "$GROUP" --assignment-id "$ID"

# MFA
pingcli pingone mfa user-devices list --environment-id "$ENV" --user-id "$USER_ID"

# DaVinci
pingcli pingone davinci flows list --environment-id "$ENV"
pingcli pingone davinci flows delete --environment-id "$ENV" --flow-id "$FLOW_ID"
pingcli pingone davinci connector-instances list --environment-id "$ENV"
pingcli pingone davinci connector-instances get --environment-id "$ENV" --id "$INSTANCE_ID"
pingcli pingone davinci connector-instances delete --environment-id "$ENV" --id "$INSTANCE_ID"
```

`pingcli pingone davinci flows list` is the fastest check for orphaned flow objects left behind by a failed Terraform apply, and `flows delete` is how you clear them. See `pingone:terraform`.

## Export

`pingcli platform export` writes an environment's configuration out as Terraform HCL. It is useful for discovering what resource types and attribute shapes the provider actually uses for objects that already exist, which is often faster than reading the provider docs. Treat the output as a reference sample, not as code to commit: it is generated flat, carries no module structure, and does not import anything into state.

## Known limitations observed in practice

- The DaVinci `connector-instances` read endpoints (both `list` and by-ID `get`) have been observed returning sustained HTTP `500`s as a genuine platform-side outage. If every call path fails identically, confirm against a second environment before assuming your configuration is at fault.
- The connector instance write API does not validate property keys against the connector's own schema. A misspelled key is stored and echoed back on every subsequent `get`, so the configuration looks correct and has never worked. See `pingone:davinci`.
