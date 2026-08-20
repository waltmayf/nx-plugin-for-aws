# Spike findings — Epic #5 (validate server-side AG-UI translation against a deployed Harness)

Tracks empirical findings for [Epic #5](https://github.com/waltmayf/nx-plugin-for-aws/issues/5), Milestone 1 of
`DESIGN-react-agentcore-harness-connection.md`. Resolves open questions Q4/Q5/Q6/Q7 (design doc §12).

This is a spike: the code referenced below is hand-written directly against a scaffolded test workspace
(not a generator — that is Epic #13). Findings here feed back into the design doc once confirmed.

## Status: in progress

## Checklist (mirrors issues #6-#12)

- [ ] #6 — Deploy a Harness + hand-written `/agui` SSE route on a generated tRPC API
- [ ] #7 — Confirm a stock `HttpAgent` and a bring-your-own AG-UI client both render the stream
- [ ] #8 — Confirm `history`/`ListEvents` reconstructs a turn after a dropped connection
- [ ] #9 — Measure `/agui` SSE survival against the API Gateway REST 29s integration cap (Q6)
- [ ] #10 — Prototype AG-UI route registration on the operation-driven REST API (Q4)
- [ ] #11 — Investigate async invocation kickoff for turns beyond the connection caps (Q6)
- [ ] #12 — Confirm `InvokeHarness` tool-call stream shape for future AG-UI `TOOL_CALL_*` (Q5)

## Findings

_(populated as each item above is completed)_
