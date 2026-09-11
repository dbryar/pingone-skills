# Flow execution logs and the PingOne remote MCP server

Supports "Observing flow executions" in [`../SKILL.md`](../SKILL.md). The Management API recipe there is the durable one; this file covers the MCP route to the same data and the shape of what comes back.

## The PingOne remote MCP server

PingOne hosts an MCP server per environment. Its tools cover platform reads and writes (applications, users, groups, populations, roles, resources, provisioning, audit activity) and DaVinci (flows, forms, connectors, variables, UI templates, and flow executions).

### Endpoint

```
https://mcp.<root domain>/admin/<environmentId>/mcp
```

`<root domain>` is the domain form pingcli uses for `ROOT_DOMAIN` (`pingone.com`, `pingone.eu`, `pingone.com.au`, ...), not a short regional suffix. `mcp.pingone.au` does not resolve; `mcp.pingone.com.au` does. A wrong host shows in the client as a DNS failure (`ENOTFOUND`), which at least is loud. Checked by DNS on 11-09-2026: `mcp.pingone.com`, `mcp.pingone.eu` and `mcp.pingone.com.au` resolve; `mcp.pingone.asia` and `mcp.pingone.au` do not.

Claude Code registration, with the client ID and callback port the server expects:

```bash
claude mcp add --transport http \
  --client-id pingone-mcp-server --callback-port 7474 \
  pingone-remote "https://mcp.<root domain>/admin/<environmentId>/mcp"
```

Register one server per environment. The environment is part of the URL, so a server registered against one environment cannot reach another.

### Who can sign in

**The authorization server is the target environment's own, not the administrators environment.** The server's protected-resource metadata (`/.well-known/oauth-protected-resource/admin/<environmentId>/mcp`) names `https://auth.<root domain>/<environmentId>/as`. An administrator whose identity lives in the administrators environment cannot sign in through it. Create a user in the target environment and grant that user roles scoped to the environment. The console states that AI clients inherit the signed-in administrator's permissions and never grant additional access, so the tools can do exactly what that user's roles allow. Environment Admin, DaVinci Admin and Identity Data Admin at environment scope were sufficient for the read tools.

The metadata advertises `scopes_supported: []` and the authorization request carries no `scope`. A client that requests no scope receives a working token, so there is nothing to add to the client configuration.

### Turning it on

The server is an early-access feature, enabled per environment in the admin console, in two steps:

1. **Settings > Environment Properties > Manage Opt-Ins**: opt in to **PingOne Remote MCP Server**.
2. **Settings > MCP Server**: enable **MCP Access**. The page lists the tool groups the server exposes and has its own Redirect URIs tab.

**Until it is on, the server can connect, authenticate, report `hasTools: true`, and list zero tools, with no error anywhere.** The client shows the server as connected; its debug log shows the connection and then nothing for that server's tool fetch. Re-authenticating, reconnecting and restarting the client change nothing. Once it is on, reconnect the server so the client re-reads the tool list; an already-running session keeps the empty list until then. 77 tools were listed on 11-09-2026.

With the opt-in enabled, the execution-log endpoints the MCP tools call are the same ones a worker with a client credentials token reads, which is the route for anything that cannot sign a person in. The worker route was confirmed on an environment with the opt-in enabled.

### Write access

The tool set includes `create*`, `update*` and `delete*` operations on flows, applications, users, connectors and more. On an environment managed by Terraform, a change made through the MCP server is drift that the next apply will revert or trip over. Restrict the client to the read tools on such environments (in Claude Code, `permissions.deny` entries for the write tools), and treat the server as an observation instrument.

The server's own instructions state that no tool accepts a secret value, and that secrets in responses are masked.

### Size of results

A single execution's event log is tens of kilobytes. MCP clients that cap tool output will spill it to a file; parse that file with a script rather than paging through it.

## Event log shape

`GET /environments/{envId}/flows/{flowId}/interactions` returns `_embedded.interactions[]`, each with `id` (the `interactionId`), `flow.id`, `flow.name`, `flow.version`, `timestamp`, `transactionId` and `isSubFlow`. The listing's own `_links.self` points at the wrong path; build URLs from the IDs.

`GET /environments/{envId}/flows/{flowId}/interactions/{interactionId}/events` returns `_embedded.events[]`. Useful fields:

| Field | Meaning |
| --- | --- |
| `message` | `Start Interaction`, `Receive Request`, `Send Response`, `Starting subflow` |
| `nodeTitle` | The node's title in the flow |
| `connector.id`, `capabilityName` | What the node ran, for example `pingOneSSOConnector` / `checkPassword` |
| `success` | `"true"` or `"false"` as a string, on `Send Response` events. A comparison node that took its false branch logs `"false"` without anything having gone wrong |
| `executionTime` | Milliseconds |
| `properties` | The node's resolved inputs and outputs. For `startUiSubFlow`, `subFlowId.value.value` is the subflow actually invoked |
| `requestContext` | User agent, request ID, and the request domain. Client IP is masked |

Each node that ran produces a `Receive Request` and a `Send Response` event.
