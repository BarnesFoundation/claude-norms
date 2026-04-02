# MCP Server on AWS Lambda with Entra ID OAuth

**Project**: TemiDataMCP (BarnesFoundation/TemiDataMCP)
**Date**: 2026-03-03 through 2026-04-01

## Approach

Remote MCP server giving Claude.ai read-only SQL access to a PostgreSQL database. Deployed as a single Lambda function behind HTTP API Gateway, authenticated via Microsoft Entra ID OAuth.

**Critical architectural decision**: Bypass the MCP SDK's `StreamableHTTPServerTransport` entirely. It uses SSE streams that never close inside `serverless-express`, causing Lambda to timeout at 60 seconds. Handle JSON-RPC manually with plain `res.json()` responses. This is valid per the MCP spec.

## Stack

- Express + `@codegenie/serverless-express` (Lambda adapter)
- AWS CDK TypeScript (NodejsFunction, HttpApi, VPC, security groups)
- `pg` for PostgreSQL (ESM requires `createRequire` banner)
- `jsonwebtoken` + `jwks-rsa` for Entra ID JWT validation
- `dotenv` for secrets management in CDK

## Key Files

| File | Purpose |
|------|---------|
| `server/src/index.ts` | Express app, OAuth proxy endpoints, manual JSON-RPC MCP handler, Lambda export |
| `server/src/schema/temi-context.ts` | Schema + domain knowledge + query patterns (the product) |
| `server/src/auth/entra.ts` | JWT validation middleware — reusable for any Entra ID project |
| `server/src/db/client.ts` | pg Pool config (max:1, statement_timeout) and executeQuery |
| `cdk/lib/temi-mcp-stack.ts` | Full CDK stack — Lambda, HTTP API, VPC, security groups |
| `cdk/.env.example` | Template for secrets |

## What Worked

- **Manual JSON-RPC is simple and reliable** — ~50 lines of switch/case handles the entire MCP protocol for stateless servers. `initialize`, `tools/list`, `tools/call`, `ping`, `notifications/initialized`.
- **OAuth proxy pattern** — Three Express routes proxy Claude.ai's OAuth flow to Entra ID. No custom token issuance, no session management. The server is a translator, not a provider.
- **Schema engineering** — Iterating tool descriptions like prompts. Warnings first, defaults labeled, intent mapping, explicit NEVER warnings, copy-paste SQL patterns.
- **Fast iteration loop** — CDK deploy takes ~30 seconds. Schema change → deploy → test in Claude.ai → ask model what went wrong → fix → redeploy.
- **Health endpoint** — `/health` with no auth, queries the database. First debugging stop for any issue.

## Gotchas

1. **SSE transport hangs on Lambda** — THE critical gotcha. `StreamableHTTPServerTransport` opens SSE streams. `serverless-express` can't close them. Lambda hangs until 60s timeout. `callbackWaitsForEmptyEventLoop = false` does NOT fix this — it fixes the pg socket issue, not the SSE issue.
2. **Entra ID dual issuers** — Tokens come with v1 (`https://sts.windows.net/{tenant}/`) OR v2 (`https://login.microsoftonline.com/{tenant}/v2.0`) issuer. Accept both in JWT validation.
3. **HTTP API, not REST API** — REST API adds `/prod/` prefix to all paths. This breaks `/.well-known/oauth-authorization-server` discovery. HTTP API has clean paths.
4. **ESM + pg** — `pg` uses CJS internally. Add createRequire banner to esbuild: `import { createRequire } from 'module'; const require = createRequire(import.meta.url);`
5. **dotenv and `#`** — Passwords containing `#` get truncated. Quote them: `DB_PASSWORD="pass#word"`.
6. **Claude skips tool descriptions** — The model's trained SQL instincts override tool description guidance. MUST use imperative language ("You MUST call describe_schema BEFORE any query") not conditional ("call describe_schema if needed").

## Reuse Potential

**HIGH** — The entire infrastructure pattern (Lambda + Express + manual JSON-RPC + Entra ID OAuth + CDK) is copy-paste reusable for any MCP server. Change the tool definitions and schema context for a new database.

Specifically reusable components:
- `auth/entra.ts` — Drop-in Entra ID JWT middleware for any Express app
- `index.ts` JSON-RPC handler — Template for any stateless MCP server on Lambda
- CDK stack — Template for Lambda + HTTP API + VPC
- OAuth proxy pattern — Template for any Entra ID (or similar IdP) OAuth integration
- Schema engineering methodology — Applicable to any MCP tool that generates SQL

## Reference

- `docs/mcp-manifesto.md` in TemiDataMCP — Complete playbook with 10 principles
- `docs/acme-mcp-spec.md` in TemiDataMCP — Design spec showing how to clone this pattern for a new database
