# Claude/Steve — CRM Data Lake Sync

## Session Expertise (2026-04-02 through 2026-04-07)

### Implemented
- **Database-as-cursor incremental sync**: Query `MAX(cursorColumn)` per model before fetching to replace global `INCREMENTAL_DATE`. Applied to 13 models. Eliminates re-pulling unchanged data.
- **Cursor snapshot**: All cursors read once before any model processes. Prevents race conditions where an earlier model advances the cursor before a later model reads it (e.g., Membership advancing before MembershipCard reads).
- **Membership v2 → v3 migration**: Upgraded from `/v2/b2b/memberships` (SDK, no pagination, hack params) to `/v3/b2b/memberships` (CustomACMEFetch, `hasMore` pagination). Eliminated MembershipsReport secondary source that caused 504 crashes.
- **Membership + MembershipCard co-processing**: Merged into single API call. Cards extracted from the same membership response — no separate model, no separate cursor, no race condition.
- **Orders early-termination pattern**: For ACME endpoints without `updatedAfter`, sort by `updatedOn desc` and stop fetching when records are older than cursor. Implemented for ACMEOrder.
- **Tiered conditional sync**: Tier 1 (every run, ~55 sec), Tier 2 (conditional on orphaned FKs using business dates), Tier 3 (daily safety net). `SYNC_TIER` env var: ALL, TIER1_ONLY, CONDITIONAL.
- **Report result cache**: Static cache on ReportRetrieverService keyed by reportUUID + date range. Eliminates duplicate Transactions report execution.
- **E.164 phone normalization**: `normalizedNumber` column using `libphonenumber-js` with country-aware parsing from customer address data. 97.9% success rate on 463K records.
- **ticketType + addOn on OPTIDP**: Added from Transactions Report. 1.9M Event items backfilled to 100% coverage. Surgical backfill script runs report month-by-month with only 3 fields.
- **eventLocation on EventInstance**: From `customFields.Location`. Values: barnesOnsite, calderOnsite, barnesOnline, offsite, TRUE (COVID era). Backfilled via day-by-day API calls back to Jan 2020. Field started Oct 6, 2020 (Barnes reopening).
- **OPTIDP indexes**: `idx_optidp_acme_customer_id` and composite `idx_optidp_customer_ticket_type` for screenpop query performance (~1s → ~10ms).
- **CURSOR_OVERRIDE**: CLI env var for catch-up runs. Supports absolute dates and relative time (30s, 5m, 5h, 3d, 2w, 1mo, 1y).
- **Membership audit logging**: Impacted membership IDs with action + card count emitted per sync run.
- **ACME API audit and feedback document**: Structured gap analysis of all endpoints, prioritized by business impact.

### Key Files
- `src/services/databaseLoader/databaseLoader.ts` — cursor snapshot, co-processing, tier logic
- `src/services/databaseLoader/utils/index.ts` — tier filtering, conditional FK checks (business date based)
- `src/services/apiRetrieverService/apiRetrieverService.ts` — all endpoint fetchers (Membership v3, Orders, Events)
- `src/services/reportRetrieverService/reportRetrieverService.ts` — report result cache
- `src/schema/models/modelSourceConfigurationTypes.ts` — `cursorColumn`, `cursorTable`, `syncTier`
- `src/services/informationObjectBuilderService/informationObjectBuilderService.ts` — phone normalization + country map
- `src/app.ts` — tiered sync orchestration (TIER1_ONLY, CONDITIONAL, ALL)
- `src/config.ts` — SYNC_TIER, CURSOR_OVERRIDE with relative time parser
- `src/backfillNormalizedPhones.ts` — phone normalization backfill
- `src/backfillTicketTypes.ts` — ticketType/addOn surgical backfill (month-by-month report)
- `docs/claude/context/acme-api-gaps.md` — ACME feedback document
- `docs/claude/context/Data Schema.xlsx` — original Hygraph schema design (Heather Hamilton)

### Knows But Hasn't Implemented
- `eventPriceListPrice` model — planned in original schema for ticket type lookup table (person types with prices). Needed for SFDC product catalog parity.
- `productName` composite field on OPTIDP — was planned as `TicketType OR AddOn OR ItemMembershipLevelName + ItemMembershipOfferingName`
- Lambda migration — memory profiled at 742 MB peak heap / 1 GB RSS. 2 GB Lambda viable. Blocked on sync reaching ~2 min target.
- BarnesSalesChannel internal classification logic — the `barnesSalesChannel` enum (Back_Office, Call_Center, Kiosk, Online, Walk_Up) is computed externally, possibly by Workato

### Gotchas Discovered
- **Cursor clock mismatch**: Prisma's `@updatedAt` (our write time) races ahead of ACME's `updatedOn`. MembershipCard was dropping records. Fix: `cursorTable` to read from Membership's ACME timestamp, plus cursor snapshot before processing.
- **ACME list endpoints silently exclude old events**: 53,776 EventInstances accessible by individual GET but invisible to list endpoint. Transaction records reference them, creating permanent orphaned FKs.
- **ACME Transactions report returns empty strings for absent FKs**: 280K OPTIDP records have `eventInstanceId = ""` instead of NULL (non-event transactions). Must exclude from orphan checks.
- **EventInstance `isOnline` Boolean was wrong type**: CustomField3 evolved from unused → TRUE/FALSE (COVID) → barnesOnsite/calderOnsite/barnesOnline/offsite (Calder 2025). Boolean column silently nulled all values. Fixed with new `eventLocation` String column.
- **EventTemplate API doesn't have Location custom field**: Only EventInstance has it. Templates define event type, instances carry venue.
- **Prisma migrate targets `DATABASE_URL` not `DATABASE_SCHEMA`**: Must append `?schema=public_test` manually.
- **Unzip strips execute permissions on Linux**: `chmod +x cron/nightly.sh` after every deploy.
- **Node 16 on EC2**: GLIBC too old for Node 18. Cron hardcodes v16.20.2 path.
- **EventTemplate pageSize > 100 causes Node string overflow**: Templates are huge objects (schedules, resources, memberships).
- **`public_test` is production**: Not `public`. Gets everyone at least once.

### Performance Results
- Full sync: 26 min → 6 min (cursor + v3 + early termination)
- Conditional sync: 6 min → 55 seconds (tiered conditional)
- Production EC2 CONDITIONAL: 54 seconds
- Production EC2 ALL: 22 minutes (Node 16 + slower hardware)
- Screenpop OPTIDP queries: ~1s → ~10ms (indexes)
