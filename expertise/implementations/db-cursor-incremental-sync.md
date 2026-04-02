# Database-as-Cursor Incremental Sync

## Project
crm-data-lake-sync (Barnes Foundation)

## Date
2026-04-02

## What Was Built
A pattern for incremental ETL syncs where the database itself serves as the cursor. Before each model's fetch, the system queries `MAX(cursorColumn)` from the target table and uses that timestamp as the API filter start date, replacing a hardcoded global date.

## Approach

1. Add a `cursorColumn` field to each model's source configuration — names the DB column to query (e.g., `modifiedDate`, `lastModifiedDate`, `transactionDate`)
2. Before fetching, run `SELECT MAX("cursorColumn") FROM "ModelName"` 
3. Use the result as the start date for the API's date filter (`updatedAfter`, `updatedStartTime`, etc.)
4. Fall back to a global date when the table is empty or cursor is unavailable

## Key Files
- `src/schema/models/modelSourceConfigurationTypes.ts` — `cursorColumn` type
- `src/services/databaseLoader/databaseLoader.ts` — `getEffectiveStartDate()` method
- Each model mapping in `src/schema/modelMappings/` — `cursorColumn` config

## Gotchas
- The cursor column should be the **source system's** modification timestamp, not Prisma's `@updatedAt` (which reflects when *we* last wrote, not when the source changed)
- For report-based models, the cursor is the transaction/submission date, not a modification date
- For endpoints without `updatedAfter` support (like ACME Orders), combine the cursor with sort-by-date-desc + early-termination instead
- Table names in raw SQL must match Prisma model names exactly (Prisma doesn't pluralize by default in PostgreSQL)

## Reuse Potential
**High** — this pattern applies to any ETL that syncs from paginated APIs into a database. The configuration-driven approach (just add `cursorColumn` to a mapping) means adding it to a new model is a one-line change.

## Results
Sync time reduced from 26 minutes to 7 minutes (73%). Models with `updatedAfter` support now pull only records changed since last sync — often single-digit records instead of hundreds of thousands.
