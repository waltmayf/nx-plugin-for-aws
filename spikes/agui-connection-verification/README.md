# AG-UI connection — human verification deployment

Tracking: #43 ("Create a deployment for human verification of the ag-ui front end").

This records the exact steps used to scaffold, connect and deploy a real demo app that
exercises the **generator-produced** (not hand-written) AG-UI wiring introduced in #38–#41:

```
ts#react-website  ──AguiProvider/useAgui hook──▶  ts#trpc-api  ──/agui route──▶  agentcore-harness
```

Unlike `spikes/agui-harness-validation` (which validated the *design* with hand-written code
before the generator existed), this uses only `pnpm nx g @aws/nx-plugin:...` commands against
this branch's built plugin — i.e. exactly what a user gets by cloning `main` today.

## Reproduce locally

From a clone of this repo:

```bash
pnpm i
pnpm nx run-many --target build --all

# in a scratch directory outside this repo
pnpm create @aws/nx-workspace agui-demo --no-interactive --iac=cdk
cd agui-demo
# point the new workspace at the locally-built plugin (see CONTRIBUTING.md / CLAUDE.md
# "Testing Changes End-to-End" — a local Verdaccio registry or overwriting
# node_modules/@aws/nx-plugin with dist/packages/nx-plugin both work)

pnpm nx g @aws/nx-plugin:ts#api --no-interactive --name=chatApi --framework=trpc --infra=rest-lambda --auth=iam
pnpm nx g @aws/nx-plugin:agentcore-harness --no-interactive --name=chatHarness
pnpm nx g @aws/nx-plugin:connection --no-interactive --sourceProject=chatApi --targetProject=chatHarness
pnpm nx g @aws/nx-plugin:ts#website --no-interactive --name=web --ux=cloudscape
pnpm nx g @aws/nx-plugin:connection --no-interactive --sourceProject=web --targetProject=chatApi
pnpm nx g @aws/nx-plugin:ts#infra --no-interactive --name=infra
```

Then wire `api.addAguiRoute(harness)` into `packages/infra/src/stacks/application-stack.ts` (the
connection generator adds the method but does not call it — see the
[tRPC → AgentCore Harness guide](../../docs/src/content/docs/en/guides/connection/trpc-agentcore-harness.mdx)).

Run the website locally against the deployed API/Harness with:

```bash
pnpm nx run web:dev
```

## Status

_Filled in as this is executed — see the PR description for the live deployment status and URL._
