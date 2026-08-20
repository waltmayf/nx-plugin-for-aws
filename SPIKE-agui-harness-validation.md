# Spike: validate server-side AG-UI translation against a deployed Harness

**Tracking:** Epic #5 (child spikes #6–#12)
**Reference:** `DESIGN-react-agentcore-harness-connection.md`
**Status:** In progress

This is the running findings doc for Epic #5. It records what was built, deployed, and
observed, and answers the open questions (Q4/Q5/Q6/Q7, §12 of the design doc) empirically.
Code produced by this spike is **throwaway/prototype**, living under `spikes/agui-harness-validation/`
in this repo — it is not the generator (that's tracked separately in Epic #13) but exists to
de-risk the generator design before it's built.

## Exit criteria (from Epic #5)

- [ ] A Harness (Korey's generator) is deployed and invokable.
- [ ] A hand-written AG-UI SSE route on a generated tRPC API renders in a stock `@ag-ui/client`
      `HttpAgent` / CopilotKit client.
- [ ] A bring-your-own AG-UI client connects by URL alone.
- [ ] Memory reconstruction (history/`ListEvents` after a dropped connection) is answered empirically.
- [ ] The 29s API Gateway REST integration cap behaviour against a held SSE response is answered empirically.
- [ ] The route-registration mechanism for attaching a non-tRPC route to the operation-driven
      REST API is prototyped (Q4).
- [ ] Async-kickoff feasibility for turns beyond the connection caps is answered empirically (Q6).

## Child spikes

| # | Title | Status |
|---|---|---|
| #6 | Deploy a Harness + hand-written `/agui` SSE route | Not started |
| #7 | Stock `HttpAgent` + bring-your-own AG-UI client both render the stream | Not started |
| #8 | `history`/`ListEvents` reconstructs a turn after a dropped connection | Not started |
| #9 | `/agui` SSE survival against the API Gateway REST 29s cap (Q6) | Not started |
| #10 | AG-UI route registration on the operation-driven REST API (Q4) | Not started |
| #11 | Async invocation kickoff for turns beyond the connection caps (Q6) | Not started |
| #12 | `InvokeHarness` tool-call stream shape for future `TOOL_CALL_*` (Q5) | Not started |

## Plan

1. Build the plugin locally (`pnpm i`, `pnpm nx run-many --target build --all`) and make it
   available to a scratch workspace via `pnpm link` (per CONTRIBUTING's e2e-validation steps).
2. Scaffold a fresh Nx workspace, add a `ts#trpc-api` project and an `agentcore-harness` project
   using the locally-compiled generators.
3. Hand-write the `/agui` SSE route (Converse→AG-UI mapper + handler) directly into the generated
   API project's CDK app and Lambda handler — no new generator code, matching the design doc's
   "hand-written" framing for this spike (§13 Milestone 1).
4. Deploy to AWS (account visible via the assumed role in this container), invoke the Harness
   through the route, and observe with:
   - a small script using `@ag-ui/client`'s `HttpAgent` directly,
   - the CopilotKit chat UI in the generated React website,
   - a bring-your-own script that only knows the `/agui` URL.
5. Drop the SSE connection mid-turn and call `ListEvents` to confirm Memory reconstruction (#8).
6. Time the SSE response against the 29s REST integration cap (#9).
7. Try attaching the extra route to the operation-driven REST CDK construct a couple of ways and
   record which is cleanest (#10).
8. Attempt an async kickoff path (Step Functions / fire-and-forget invoke) for turns beyond the
   streaming cap (#11).
9. Inspect the raw `InvokeHarness` stream for tool-call-shaped events (#12).
10. Tear down all provisioned AWS resources.
11. Write up findings + recommendations for each open question below, and update the child issues.

## Findings

_(filled in as the spike proceeds)_

### Q4 — AG-UI route registration mechanism

### Q5 — Tool-call stream shape

### Q6 — 29s cap behaviour / async kickoff feasibility

### Q7 — Article alignment

## Teardown log

_(record of `cdk destroy` / stack deletions, with timestamps)_
