---
name: qa
description: Test and diagnose PingOne DaVinci journeys end to end with the flow's own execution log as the oracle - proving which path a run took, following it into subflows, reporting the node where it went wrong, and the test-harness traps that make a broken journey pass. Use whenever writing or running an end-to-end test of a DaVinci flow, asserting on which branch a login took, triaging a failed test or a reported login defect, or choosing the credential a test suite reads logs with. Assumes pingone:core for the log endpoints and pingone:davinci for what the nodes mean.
---

# Testing DaVinci journeys

**What the client rendered is not evidence of which path the flow took.** A flow can take the wrong branch and still draw a screen that looks right, still issue a code, still complete. The execution log is the only record of what actually ran: every node, in order, with its resolved inputs and its output. Assert on it.

Companion skills: `pingone:core` ("Observing flow executions") for the endpoints and the subflow lookup, `pingone:davinci` for what a node's log entry means, `pingone:sdk` for what an embedded client can render.

## The oracle

A test that drives a journey should, after the client-side assertions, read that run's execution log and check the path.

**Get the run's `interactionId` from the run itself.** The main flow's success response carries it, and it is the execution's `id`. Do not pick "the newest execution in the window": two runs that overlap will read each other's logs, and the test will pass or fail on someone else's run.

**Follow the run into its subflows.** A subflow is a separate execution with its own `interactionId`. Events carry no `transactionId`, so take it from the parent flow's execution listing (searched around the run's first event timestamp, matched on `id`), then list each subflow's executions filtered on it. The subflow to search is the one the parent's `startUiSubFlow` event names in `properties.subFlowId`, not the one its node title suggests. Details in `pingone:core`.

**Assert milestones, not the whole path.** Pick the nodes that make the journey what it is, per flow, and require them in order with anything allowed between them. Asserting every node makes the test fail whenever someone adds a variable merge that changes nothing a member sees. Add forbidden nodes for the branch the test exists to rule out.

**Match on node title, per flow.** Titles are what Studio and the log both show. A renamed milestone breaks the test, which is correct: the milestone list is part of the journey's contract, and a rename should be a deliberate change to it.

Reading the log correctly:

- A completed node logs `Send Response`. A node can log two (a `startNode` and then its capability), and a form logs one per render, so collapse consecutive repeats before comparing.
- `success: "false"` is not a failure. A comparison node that takes its false branch logs it, correctly. Assert on the path, not on the absence of `false`.
- `success` is a string, not a boolean.
- The events response links a `next` page even when the page is shorter than the requested limit. Only a full page means events were left behind; fail loudly on one rather than paging blind.

## When a check fails, say where

A failing path check should print, without anyone opening Studio: which milestone was missed or which forbidden node ran, the completed path of every flow in the run, and the resolved `properties` and `response` of the last node each one completed. Most defects are diagnosable from that alone. The node where a run stopped, and what it was given, is usually the whole answer.

## Diagnose from the log first

When a test fails or a login defect is reported, read the execution log before reading flow source or reasoning about configuration. The expensive failures in DaVinci are the silent ones, and several have a signature only the log shows:

- A node logs success in a few milliseconds and **nothing downstream is logged at all**, then the client times out. That is an unclaimed outcome or a shared evaluator, not a timeout. See `pingone:davinci`, "Branching".
- A comparison takes the mismatch branch every time. Check its inputs in `properties`: a subflow bound to a caller variable compares against nothing. See `pingone:davinci`, "Subflows".
- The relying party receives `login_required` with a generic description. The log shows the real cause, often `subflowFailed`. See `pingone:davinci`, "Error handling".

For interactive diagnosis from an agent, the PingOne remote MCP server reads the same logs; see `pingone:core`'s `reference/execution-logs.md`.

## The credential a test suite reads with

Give the suite a worker that can read logs and nothing else. `DaVinci Admin Read Only` at environment scope reads both the execution listing and the events, and refuses writes: a `DELETE` of a flow and a `POST` of a variable both return `403` for it, where a DaVinci Admin worker gets past authorisation on the same calls. Its permissions are all reads. Do not let the suite borrow the worker that manages the environment.

The MCP server is not a substitute here. It signs in a person through the environment's own authorization server, with MFA, which a test run cannot do.

Make every failure to read loud:

- **Missing credential: fail the test, naming the variable.** A log check that skips quietly reports a pass it never checked.
- **Refused read: throw with the status.** A `401` or `403` parsed as an empty log surfaces later as "milestone did not complete", which sends the reader after the flow instead of the credential.
- **Late log: retry for a bounded time, then fail naming what was searched for.** A subflow execution that is not listed yet is not the same as one that never ran.

## Harness traps that make a broken journey pass

These are the ways a journey test reports success for a run that did not do what it claims. Each was found by a real test passing a broken flow.

- **An action that matches nothing must throw.** A click helper that silently does nothing leaves the screen unchanged, so every "it re-rendered" assertion after it still holds. This hid a resend that never happened.
- **A captured OTP must be newer than the send.** A mock delivery gateway that keeps one record per recipient returns the previous run's code to a read that races the webhook. The flow then correctly rejects it, and the failure reads as broken activation. Record the latest message's timestamp before triggering the send and wait for a newer one. Allow tens of seconds: delivery degrades under repeated sends to one recipient.
- **Clear the session between cases.** A leftover PingOne session turns a first login into a returning one, and the flow takes a different route that looks like a pass.
- **Respect the lockout policy.** Repeated failing attempts lock the test account and turn one diagnosable failure into a wait. See `pingone:core`.

## Test the log helper offline

The code that reads and interprets logs is logic, and it should be unit-tested without a tenant. Capture one real run's API responses (the parent's events, the parent listing, the subflow listing, the subflow's events) as fixtures and stub only the network. A helper tested against a shape written from memory will be wrong about exactly the details above: where `transactionId` lives, the string `success`, the repeated `Send Response`.

## Correcting this skill

When an instruction here does not match observed behaviour, or you confirm something it does not cover, run `/pingone:learn`. Nothing goes in unconfirmed.
