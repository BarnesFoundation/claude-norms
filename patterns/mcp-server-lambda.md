# Pattern: MCP Server on AWS Lambda

## Problem

You want to deploy a remote MCP (Model Context Protocol) server as an AWS Lambda function behind API Gateway, but the MCP SDK's `StreamableHTTPServerTransport` uses SSE (Server-Sent Events) that never close inside `serverless-express`, causing Lambda to timeout at 60 seconds.

## Solution

Handle the JSON-RPC protocol manually instead of using the MCP SDK transport. Parse the `method` field from the request body, route to your handlers, return plain JSON via `res.json()`. This is valid per the MCP spec — servers can respond with either JSON or SSE for streamable HTTP.

### Architecture

```
Claude.ai → HTTP API Gateway → Lambda → Express → JSON-RPC router → Tool handlers → res.json()
                                                ↕
                                        OAuth proxy (Entra ID / other IdP)
```

### Infrastructure Stack (CDK)

```typescript
// HTTP API (NOT REST API — REST adds /prod/ prefix that breaks OAuth discovery)
const httpApi = new apigwv2.HttpApi(this, 'McpHttpApi');

// Lambda with ESM bundling
const fn = new lambdaNodejs.NodejsFunction(this, 'McpFunction', {
  entry: 'server/src/index.ts',
  handler: 'handler',
  runtime: lambda.Runtime.NODEJS_20_X,
  timeout: cdk.Duration.seconds(60),
  memorySize: 256,
  vpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
  bundling: {
    format: lambdaNodejs.OutputFormat.ESM,
    banner: "import { createRequire } from 'module'; const require = createRequire(import.meta.url);",
    minify: true,
    sourceMap: true,
  },
});
```

### JSON-RPC Handler (the core pattern)

```typescript
app.post('/mcp', authMiddleware, async (req, res) => {
  const { jsonrpc, id, method, params } = req.body;

  // Notifications — no response expected
  if (method === 'notifications/initialized' || method === 'notifications/cancelled') {
    res.status(204).end();
    return;
  }

  switch (method) {
    case 'initialize':
      res.json({
        jsonrpc: '2.0', id,
        result: {
          protocolVersion: '2025-03-26',
          capabilities: { tools: {} },
          serverInfo: { name: 'my-mcp', version: '1.0.0' },
        },
      });
      break;

    case 'ping':
      res.json({ jsonrpc: '2.0', id, result: {} });
      break;

    case 'tools/list':
      res.json({ jsonrpc: '2.0', id, result: { tools: TOOL_DEFINITIONS } });
      break;

    case 'tools/call': {
      const result = await handleToolCall(params.name, params.arguments);
      res.json({ jsonrpc: '2.0', id, result });
      break;
    }

    default:
      res.json({ jsonrpc: '2.0', id, error: { code: -32601, message: `Method not found: ${method}` } });
  }
});
```

### Lambda Handler Export

```typescript
import serverlessExpress from '@codegenie/serverless-express';

const serverlessHandler = serverlessExpress({ app });
export const handler = (event: unknown, context: { callbackWaitsForEmptyEventLoop: boolean }) => {
  context.callbackWaitsForEmptyEventLoop = false; // Prevents pg Pool socket from holding Lambda open
  return serverlessHandler(event, context);
};
```

### OAuth Proxy (for Entra ID or any IdP)

Three routes that translate between the client's OAuth flow and the IdP:

1. `GET /.well-known/oauth-authorization-server` — Returns metadata pointing to your endpoints
2. `GET /authorize` — Redirects to IdP with corrected scopes
3. `POST /token` — Proxies token exchange to IdP, returns response

See memo `2026-04-01-temi-mcp-entra-oauth-for-screenpop.md` for full code examples.

## Why This Works

The MCP spec for streamable HTTP allows two response modes:
1. **JSON** — Single JSON-RPC response in HTTP body (`Content-Type: application/json`)
2. **SSE** — Server-Sent Events stream (`Content-Type: text/event-stream`)

`res.json()` produces response mode 1, which completes immediately. Lambda returns, API Gateway sends the response, done. No hanging streams.

## Key Gotchas

| Gotcha | Problem | Fix |
|--------|---------|-----|
| SSE on Lambda | `StreamableHTTPServerTransport` hangs forever | Handle JSON-RPC manually |
| `callbackWaitsForEmptyEventLoop` | pg Pool TCP socket holds Lambda open | Set to `false` in handler |
| ESM + pg | `pg` uses CJS internally | Add `createRequire` banner to esbuild |
| REST API stage prefix | `/prod/` breaks `/.well-known/` discovery | Use HTTP API instead |
| Entra ID dual issuers | v1 and v2 tokens have different issuer formats | Accept both in JWT validation |
| dotenv `#` parsing | Passwords truncated at `#` character | Quote in `.env`: `DB_PASSWORD="p#ss"` |
| Tool descriptions | Model ignores guidance, uses trained instincts | Use imperative language: "MUST", "NEVER" |

## Schema Engineering (for SQL-generating tools)

The tool description is a prompt. The LLM is your user. Critical principles:

1. **Warnings FIRST** — biggest mistakes at the top, before columns or patterns
2. **Defaults, not choices** — label the default query pattern as "DEFAULT"
3. **Intent mapping** — "how many visitors?" → correct query type
4. **NEVER warnings** — actively counterprogram trained instincts ("NEVER use COUNT(*)")
5. **Mandatory schema reading** — "You MUST call describe_schema BEFORE any query"
6. **Ask the model** — when it gets wrong answers, ask it what would fix the schema

Full methodology: `TemiDataMCP/docs/mcp-manifesto.md`

## Origin

Created 2026-03-03 on TemiDataMCP. First production use: Barnes Foundation Temi visitor tracking database via Claude Team custom connector. The SSE/Lambda incompatibility was discovered after hours of debugging 60-second timeouts where DB queries completed in <1s but responses never reached the client.

## Reuse

Copy the entire infrastructure from `BarnesFoundation/TemiDataMCP`. Change the tool definitions and schema context for your database/API. The ACME MCP connector (Phase 2) will be built this way — design spec at `TemiDataMCP/docs/acme-mcp-spec.md`.
