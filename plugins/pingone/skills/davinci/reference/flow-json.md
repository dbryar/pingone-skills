# DaVinci flow JSON reference

## Top-level keys

An exported flow is a single JSON document. The runtime-significant keys are `settings`, `graphData`, `variables` and `trigger`; the rest is identity and bookkeeping.

```json
{
  "companyId": "...",
  "customerId": "...",
  "flowId": "...",
  "name": "Sign in",
  "description": "Sign in",
  "flowStatus": "enabled",
  "flowColor": "#E3F0FF",
  "currentVersion": -1,
  "publishedVersion": -1,
  "versionId": -1,
  "createdDate": 1760000000000,
  "updatedDate": 1760000000000,
  "deployedDate": 1760000000000,
  "savedDate": 1760000000000,
  "isOutputSchemaSaved": false,
  "authTokenExpireIds": [],
  "connectorIds": {},
  "settings": {},
  "timeouts": {},
  "trigger": {},
  "graphData": {},
  "variables": [],
  "forms": [],
  "connections": []
}
```

`forms` and `connections` appear even when empty.

## graphData

```json
{
  "elements": { "nodes": [], "edges": [] },
  "data": {},
  "zoomingEnabled": true,
  "userZoomingEnabled": true,
  "panningEnabled": true,
  "userPanningEnabled": true,
  "boxSelectionEnabled": true,
  "renderer": {}
}
```

The Cytoscape boilerplate is not optional in practice, even where a schema marks it so. Missing fields surface as API errors at flow-creation time (`no value given for required property boxSelectionEnabled`, `grabbable`), not at validation time.

## Node

```json
{
  "data": {
    "id": "2xcb4zcir9",
    "nodeType": "CONNECTION",
    "connectorId": "flowConnector",
    "connectionId": "<connector-instance-id>",
    "capabilityName": "startUiSubFlow",
    "name": "Flow Connector",
    "label": "Flow Connector",
    "type": "trigger",
    "status": "configured",
    "isDisabled": false,
    "properties": {
      "nodeTitle": { "value": "Session restore" },
      "subFlowId": {
        "value": { "label": "Session restore subflow", "value": "<subflow-id>" }
      },
      "subFlowVersionId": { "value": -1 }
    }
  },
  "position": { "x": 830.5, "y": 830.5 },
  "group": "nodes"
}
```

`nodeType` is `CONNECTION` for a connector call and `EVAL` for a decision node with no connector. Some nodes carry blank connector fields; these are evaluator or visual helpers and still participate in edge routing.

### Outcomes

A node with more than one named exit declares them, and each must be claimed by an outgoing edge:

```json
"outcomes": [ { "id": "<edge id>", "label": "Log in", "result": "submit" } ]
```

All three fields are required by the `pingone` provider (`~> 1.21`). `id` is the **edge** the outcome leaves by, not the node — pair it with that edge's `multi_value_source_id`, below. An unclaimed outcome routes nowhere and the flow stops without an error.

## Edge

```json
{ "data": { "id": "arset2nic3", "source": "p3wcu2oywm", "target": "pohtik6pdl" },
  "group": "edges" }
```

An edge carries no conditions. It carries one routing field, and only when the source node needs it:

```json
{ "data": { "id": "…", "source": "…", "target": "…", "multi_value_source_id": "<outcome or option id>" },
  "group": "edges" }
```

`multi_value_source_id` names **which exit of the source node** this edge leaves by. It is required on every edge leaving a node that has more than one named exit — a multi-select's options, or any node declaring `outcomes`. Omitting it routes the result nowhere, silently; see the SKILL's "Branching".

**Spelling differs between an export and the Terraform provider.** A Studio export spells it `multiValueSourceId`; the `pingone` provider's edge schema spells it `multi_value_source_id`, like every other attribute it exposes. Carry the camelCase spelling into HCL and Terraform drops the field without comment, and every branch then leaves by the same exit.

**The provider's edge has no `id` attribute.** `graph_data.elements.edges` is a map keyed by edge ID, and `edge.data` exposes `source`, `target` and `multi_value_source_id` only — so the map key *is* the identity, and an `outcomes` entry pointing at an edge must name that key.

## Variables

```json
{
  "name": "featureFlagConfig##SK##company",
  "context": "company",
  "type": "property",
  "visibility": "private",
  "fields": {
    "type": "object",
    "displayName": "featureFlagConfig",
    "value": "{\"someFlag\":true}",
    "min": 0,
    "max": 2000
  }
}
```

Object-typed variables hold a JSON-encoded string in `fields.value`. Both `company` and `flowInstance` contexts support object typing.

## Settings

Fields observed in real exports:

| Field | Purpose |
| --- | --- |
| `css`, `useCustomCSS`, `disableDvCss` | Page-level stylesheet, served at `/{envId}/davinci/flows/{flowId}/css` |
| `useCustomScript` | Enables the flow's `customScript` |
| `jsLinks` | External script URLs. Commonly rewritten per environment at deploy time |
| `csp`, `enforcedCsp` | Content security policy for the hosted page |
| `intermediateLoadingScreenHTML`, `intermediateLoadingScreenCSS` | Between-screen loading state |
| `customTimeoutErrorScreenMessage`, `customTimeoutErrorScreenHTML`, `customTimeoutErrorScreenCSS` | Timeout screen |
| `logLevel` | Flow log verbosity |

## Parsing rules for tooling

- `graphData.elements.nodes[]` is the source of executable behaviour. `edges[]` is control flow only, not the decision model.
- Read `data.connectorId` and `data.capabilityName` together before interpreting `data.properties`.
- Property values are either `{ "value": ... }` wrappers or raw strings. Handle both.
- Treat `connectionId` and subflow IDs in an export as environment-local. They do not carry to another environment.
- Preserve unknown fields on any round trip. DaVinci uses visual and internal metadata that has no meaning to your tooling and is still required.
- Assert that every EVAL branch key names a node with a real outgoing edge. Studio leaves stale keys behind when a branch is repointed.
- Assert that every `{{local.<nodeId>...}}` binding names a node present in the same graph. Claim rows and node properties pasted from another flow keep the source flow's node IDs, and nothing rejects them.
- Assert that no placeholder token survives whatever substitution your pipeline does. A textual `replace()` that misses returns the string unchanged, applies cleanly, and fails at runtime.

## Subflow schemas

A subflow is the same object as a main flow, distinguished by `trigger` being null and `input_schema` / `output_schema` being populated.

- `input_schema` is a list of objects: `property_name`, `preferred_data_type`, `preferred_control_type`, `required`, `description`, `is_expanded`.
- `output_schema` is a single nested `output` object whose `properties` attribute is itself a **JSON-encoded string**, not a structured value.

A subflow's declared input is read inside the subflow as `{{global.parameters.<property_name>}}`, and passed from the caller as a top-level property on the calling node named `<property_name>`.
