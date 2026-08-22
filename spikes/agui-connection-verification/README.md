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

## Live deployment (bonus — remote link)

A real instance is deployed to account `796988593450` / `us-east-1`:

- **Website (CloudFront):** https://d3m4pu7bjp9we9.cloudfront.net
- Self-signup is enabled — click **"Create an account"** on the sign-in page, verify your
  email, then set up an authenticator-app MFA code (Cognito's `Mfa.REQUIRED` default from the
  `ts#website#auth` generator — any TOTP app works, e.g. Google Authenticator).
- After signing in, go to `/chat` and talk to the AgentCore Harness. Verified end-to-end
  2026-08-22 with `"Say the word PONG and nothing else."` → the CopilotKit UI rendered `PONG`,
  confirming the full path: browser → Cognito → SigV4 → `/agui` route → `InvokeHarness` →
  Converse→AG-UI mapping → CopilotKit.

:::caution
The deployed Harness has `allowedTools: ['@builtin']` (shell + other built-in tools) so the tool-call
path could be exercised too — treat this like any other internet-reachable demo with code-execution
capability and tear it down when you're done verifying (see [Teardown](#teardown)).
:::

### Run the website locally against this live backend (no redeploy needed)

Because the `/agui` route has no local dev server (see [Local Development](../../docs/src/content/docs/en/guides/connection/trpc-agentcore-harness.mdx#local-development)
in the generator's guide), "local hosting" for this feature means running the website's Vite
dev server on your machine while it talks to a real deployed API + Harness — exactly what
`nx run <website>:serve` + a `runtime-config.json` in `public/` gives you. These values aren't
secret (they're the same ones the deployed site itself fetches from `/runtime-config.json`), so
you can use the live deployment above without your own AWS account or credentials:

1. Clone this repo and follow [Reproduce locally](#reproduce-locally) below up to (and
   including) `pnpm i` in the scratch workspace, **or** grab any existing
   `ts#react-website -> ts#trpc-api -> agentcore-harness` workspace.
2. Create `packages/web/public/runtime-config.json` (adjust the project path to your website's):
   ```json
   {
     "cognitoProps": {
       "region": "us-east-1",
       "identityPoolId": "us-east-1:86d81898-3548-460c-beac-58a149b5056e",
       "userPoolId": "us-east-1_TEhpG07aF",
       "userPoolWebClientId": "53ip3bmu484ud7h64ptumdac02"
     },
     "apis": { "ChatApi": "https://fc8lhl99wg.execute-api.us-east-1.amazonaws.com/prod/" }
   }
   ```
3. `pnpm nx run web:serve` (**not** `web:dev` — that target overrides `apis.ChatApi` to a local
   URL for the tRPC procedures, which is right for iterating on ordinary procedures but wrong
   here since it would point `/agui` at a URL with no Lambda behind it).
4. Open http://localhost:4200 — the generated Cognito App Client already allows this exact
   callback URL for local development against a deployed pool. Sign in (or sign up) and go to
   `/chat`.

Verified working this way from this sandbox (confirmed `curl http://localhost:4200/runtime-config.json`
returns the config above and the app serves at `http://localhost:4200/`); the sign-in/chat round
trip itself was only exercised end-to-end against the CloudFront URL directly (browser automation
in this environment can't reach a `localhost` port on the machine that ran the deploy).

## Reproduce locally

From a clone of this repo:

```bash
pnpm i
pnpm nx run-many --target build --all
```

In a scratch directory outside this repo, make the locally-built plugin available (see
CONTRIBUTING.md / CLAUDE.md "Testing Changes End-to-End" — a local Verdaccio registry or
overwriting `node_modules/@aws/nx-plugin` with `dist/packages/nx-plugin` both work), then:

```bash
pnpm create @aws/nx-workspace agui-demo --no-interactive --iac=cdk
cd agui-demo

pnpm nx g @aws/nx-plugin:ts#api --no-interactive --name=chatApi --framework=trpc --infra=rest-lambda --auth=iam
pnpm nx g @aws/nx-plugin:agentcore-harness --no-interactive --name=chatHarness
pnpm nx g @aws/nx-plugin:connection --no-interactive --sourceProject=chat-api --targetProject=chat-harness
pnpm nx g @aws/nx-plugin:ts#website --no-interactive --name=web --ux=cloudscape
pnpm nx g @aws/nx-plugin:connection --no-interactive --sourceProject=web --targetProject=chat-api
pnpm nx g @aws/nx-plugin:ts#website#auth --no-interactive --project=web --allowSignup=true
pnpm nx g @aws/nx-plugin:ts#infra --no-interactive --name=infra
pnpm nx sync
```

The `connection` generator run against `web -> chat-api` automatically detects that
`chat-harness` is already wired to `chat-api` and generates the `useAguiChatHarness` hook +
`AguiProvider` for you — no extra connection step needed for the harness/website pair.

### Manual wiring

Two things the generator deliberately leaves for you to assemble by hand (documented in the
[tRPC → AgentCore Harness guide](../../docs/src/content/docs/en/guides/connection/trpc-agentcore-harness.mdx)):

`packages/infra/src/stacks/application-stack.ts`:

```ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { ChatApi, ChatHarness, UserIdentity, Web } from '@agui-demo/common-constructs';

export class ApplicationStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const identity = new UserIdentity(this, 'Identity');

    const chatHarness = new ChatHarness(this, 'ChatHarness', {
      allowedTools: ['@builtin'],
    });

    const chatApi = new ChatApi(this, 'ChatApi', {
      integrations: ChatApi.defaultIntegrations(this).build(),
    });
    chatApi.addAguiRoute(chatHarness);
    chatApi.grantInvokeAccess(identity.identityPool.authenticatedRole);

    new Web(this, 'Web');
  }
}
```

A chat page (`packages/web/src/routes/chat.tsx`) using the generated `useAguiChatHarness` hook's
agent id (`chat-harness`, i.e. the harness project's kebab-case name):

```tsx
import { ContentLayout, Header } from '@cloudscape-design/components';
import { createFileRoute } from '@tanstack/react-router';
import { CopilotChat } from '../components/copilot';

export const Route = createFileRoute('/chat')({
  component: RouteComponent,
});

function RouteComponent() {
  return (
    <ContentLayout header={<Header>Chat with the AgentCore Harness</Header>}>
      <CopilotChat agentId="chat-harness" />
    </ContentLayout>
  );
}
```

### Build and deploy

```bash
pnpm nx run-many --target build --projects=common-constructs,chat-api,chat-harness,web,infra
pnpm nx run infra:deploy-sandbox
```

This is the exact recipe used to produce the live deployment linked above — `pnpm nx run-many
--target build ...` passed cleanly (including `cdk synth` and `checkov`) and `cdk deploy` reached
`CREATE_COMPLETE` (95/95 resources) with no manual patching beyond the two files above.

## Teardown

```bash
pnpm nx run infra:destroy -- "agui-demo-infra-sandbox/*"
```

or `cdk destroy "agui-demo-infra-sandbox/*"` from `packages/infra`.
