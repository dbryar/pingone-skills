# Changelog

## 0.1.1 - 2026-08-25

- `pingone:davinci` - custom claims on `returnSuccessResponseRedirect`. `accessTokenClaims` and `idTokenClaims` are independent lists; claim row shape and which fields are cosmetic; pasted claim blocks keeping the source flow's node IDs, which nothing validates; branch reachability of the node a terminal sources claims from; why not to override `sub`; guidance on which token identity claims belong in, and when putting them in the access token is a reasonable constraint.

## 0.1.0 - 2026-08-14

Initial release.

- `pingone:core` - tenant operations. Service gating presenting as permission errors, pingcli authentication, the role scope model, environment creation, field validation limits, MFA device pairing, headless authentication via `pi.flow`, error vocabulary, environment limits.
- `pingone:davinci` - flow authoring. Render mechanisms, the graph model, property encoding, variable contexts and bindings, subflow contracts, branching and teleports, terminals, error handling, the hosted page surface, session checking, connectors.
- `pingone:terraform` - both as code. The plan as a drift detector, provider selection, `pingone_davinci_flow` shapes, subflow substitution, deploy and flow policy, apply failure modes, resource notes.
- `/pingone:learn` - records a confirmed finding into the skills and the journal.
