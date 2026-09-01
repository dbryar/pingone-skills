# Changelog

## 0.1.6 - 2026-09-01

- `pingone:terraform` - `pingone_form`. A form is a first-class PingOne object with its own lifecycle, not flow content, so a `showForm` node's form reference needs the same token substitution as `subFlowId` and fails the same silent way. Field positions, the `type` enum, directory-attribute field keys, Slate labels with their own language bundle, and the stock forms a new environment already has.
- `pingone:davinci` - three field vocabularies disagree. The form builder authors 24 types, the JavaScript SDK collects 21, the Android SDK collects 14. A field outside the consuming SDK's set is dropped from the collector list rather than raising, so the screen renders looking complete and cannot be submitted.

## 0.1.4 - 2026-09-01

- `pingone:davinci` - connector instances. Studio creates a duplicate the first time a connector is opened on a canvas, whether or not a managed instance already exists, so the check has to be periodic rather than once. Default instance names read from the connector catalogue, since the id does not predict the name.
- `pingone:terraform` - reconciling a duplicate instance, and telling console-created instances from Terraform-created ones by `customerId`. A PingOne collection endpoint returning 500 means one unserialisable row, not an outage.

## 0.1.3 - 2026-09-01

- `pingone:davinci` - outcome routing. A node's declared outcome routes nowhere unless the edge leaving by it carries `multi_value_source_id`, and each outcome needs its own evaluator rather than a shared one. Both faults present as the connector logging success, no downstream node logging at all, and the client seeing `400 requestTimedOut`.

## 0.1.2 - 2026-08-27

- `pingone:davinci` - a node with no inbound edge is an entry point, not a dead node. An unwired terminal fires on entry and returns before anything runs, which is valid JSON, applies cleanly, and reports nothing.

## 0.1.1 - 2026-08-25

- `pingone:davinci` - custom claims on `returnSuccessResponseRedirect`. `accessTokenClaims` and `idTokenClaims` are independent lists; claim row shape and which fields are cosmetic; pasted claim blocks keeping the source flow's node IDs, which nothing validates; branch reachability of the node a terminal sources claims from; why not to override `sub`; guidance on which token identity claims belong in, and when putting them in the access token is a reasonable constraint.

## 0.1.0 - 2026-08-14

Initial release.

- `pingone:core` - tenant operations. Service gating presenting as permission errors, pingcli authentication, the role scope model, environment creation, field validation limits, MFA device pairing, headless authentication via `pi.flow`, error vocabulary, environment limits.
- `pingone:davinci` - flow authoring. Render mechanisms, the graph model, property encoding, variable contexts and bindings, subflow contracts, branching and teleports, terminals, error handling, the hosted page surface, session checking, connectors.
- `pingone:terraform` - both as code. The plan as a drift detector, provider selection, `pingone_davinci_flow` shapes, subflow substitution, deploy and flow policy, apply failure modes, resource notes.
- `/pingone:learn` - records a confirmed finding into the skills and the journal.
