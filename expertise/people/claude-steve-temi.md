# Claude/Steve — TemiDataMCP

**Date**: 2026-03-03 through 2026-04-01
**Project**: TemiDataMCP (BarnesFoundation/TemiDataMCP)

## Implemented

- **MCP server on Lambda** — Full remote MCP server deployed as Lambda via CDK, with manual JSON-RPC handler (bypassed MCP SDK SSE transport that hangs on Lambda). Express + `@codegenie/serverless-express`. See `patterns/mcp-server-lambda.md`.
- **Microsoft Entra ID OAuth proxy** — Three endpoints (`/.well-known/oauth-authorization-server`, `/authorize`, `/token`) proxying Claude.ai's OAuth flow to Entra ID. JWT validation middleware accepting both v1.0 and v2.0 token issuers. See memo to amazon-connect-screenpop.
- **Schema-engineered MCP tool descriptions** — Iterative prompt engineering of tool descriptions to guide LLM SQL generation. Achieved correct-first-answer through: warnings first, default query patterns, intent mapping, explicit NEVER warnings, mandatory schema reading.
- **PostgreSQL read-only query tool** — SQL validation (reject non-SELECT), 30s timeout, 10k row limit, SQL echoed in results for user iteration.
- **CDK infrastructure** — NodejsFunction with ESM bundling, HTTP API Gateway (no stage prefix), VPC-attached Lambda, security group rules for RDS access.
- **Secrets management** — Extracted from cdk.json to .env via dotenv. Caught `#` in password being parsed as comment delimiter.

## Key Lessons

1. **MCP SDK SSE transport is incompatible with Lambda** — `StreamableHTTPServerTransport` opens SSE streams that never close inside serverless-express. Handler Promise never resolves, Lambda times out at 60s. Fix: handle JSON-RPC manually with `res.json()`.
2. **`callbackWaitsForEmptyEventLoop = false` is necessary but not sufficient** — Fixes the pg Pool socket issue (Lambda not exiting) but does NOT fix the SSE hang (different problem entirely).
3. **Schema engineering is prompt engineering** — Tool descriptions are prompts for the model. "should" gets skipped, "MUST" gets followed. Warnings at top, defaults labeled, intent mapping bridging natural language to SQL.
4. **Ask the model what went wrong** — When Claude.ai makes a mistake, ask it "how would you improve the schema so you get the right answer first try?" It will tell you exactly what to fix. Then encode that fix and redeploy.
5. **dotenv parses `#` as comment** — Passwords containing `#` MUST be quoted in `.env` files.
6. **ESM + pg requires createRequire banner** — `import { createRequire } from 'module'; const require = createRequire(import.meta.url);` in esbuild banner.
7. **HTTP API, not REST API** — REST API adds `/prod/` stage prefix that breaks MCP endpoint discovery and OAuth metadata URLs.
8. **Entra ID v1 vs v2 issuers** — Tokens may come with either `https://login.microsoftonline.com/{tenant}/v2.0` or `https://sts.windows.net/{tenant}/` as issuer. Accept both.

## Key Files

- `server/src/index.ts` — Manual JSON-RPC handler pattern (the Lambda fix)
- `server/src/schema/temi-context.ts` — Schema + domain knowledge (the product)
- `server/src/auth/entra.ts` — Entra ID JWT validation (reusable)
- `cdk/lib/temi-mcp-stack.ts` — CDK stack pattern (reusable)
- `docs/mcp-manifesto.md` — Complete playbook for MCP connectors
- `docs/acme-mcp-spec.md` — Design spec for next connector
