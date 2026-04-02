# Claude/Steve — CRM Data Lake Sync

## Session Expertise (2026-04-02)

### Implemented
- **Database-as-cursor incremental sync**: Query `MAX(cursorColumn)` per model before fetching to replace global `INCREMENTAL_DATE`. Applied to 13 models. Eliminates re-pulling unchanged data.
- **Membership v2 → v3 migration**: Upgraded from `/v2/b2b/memberships` (SDK, no pagination, hack params) to `/v3/b2b/memberships` (CustomACMEFetch, `hasMore` pagination). Eliminated MembershipsReport secondary source that caused 504 crashes.
- **Orders early-termination pattern**: For ACME endpoints without `updatedAfter`, sort by `updatedOn desc` and stop fetching when records are older than cursor. Implemented for ACMEOrder.
- **E.164 phone normalization**: Added `normalizedNumber` column using `libphonenumber-js` with country-aware parsing from customer address data. 97.9% success rate on 463K records.
- **ACME API audit and feedback document**: Structured gap analysis of all endpoints, prioritized by business impact.

### Key Files
- `src/services/databaseLoader/databaseLoader.ts` — cursor query logic (`getEffectiveStartDate`)
- `src/services/apiRetrieverService/apiRetrieverService.ts` — all endpoint fetchers (Membership v3, Orders, Events)
- `src/schema/models/modelSourceConfigurationTypes.ts` — `cursorColumn` type definition
- `src/services/informationObjectBuilderService/informationObjectBuilderService.ts` — phone normalization + country map
- `src/backfillNormalizedPhones.ts` — one-time backfill script
- `docs/claude/context/acme-api-gaps.md` — ACME feedback document

### Knows But Hasn't Implemented
- ACME Report-based models (Transactions, TicketAnalytics, Forms) — understood the async report pattern but didn't modify it
- Prisma migration gotchas on this project — multiple failed historical migrations needed `resolve --applied`
- The EC2 deployment is on Ubuntu with GLIBC too old for Node 18; must stay on Node 16.20.2
- `public_test` schema is production (not `public`)

### Gotchas Discovered
- ACME Memberships v2 List endpoint didn't return `importId`/`externalMembershipId` — v3 does
- ACME Orders search requires `email: ""` hack to list all orders
- EventTemplate objects are very large (schedules, resources) — `pageSize > 100` causes Node string length overflow
- `ApiRetrieverService` was hardcoding `Config.incrementalDate` instead of using the date range passed from DatabaseLoader — had to thread it through
- Phone number field in ACME contains garbage: names, emails, date ranges, question marks
