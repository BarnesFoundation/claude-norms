# Conditional Tiered Sync

A pattern for ETL systems that sync data from external APIs into a database, where some data sources are expensive to pull but rarely change.

## The Problem

In a typical ETL sync, every model is pulled on every run — even if the data hasn't changed. When some models are expensive (large datasets, slow APIs, no date filtering), the entire sync is bottlenecked by the slowest model. Running more frequently to improve data freshness is impractical because the slow models dominate the runtime.

## The Solution: Three Tiers

### Tier 1 — Every Run
Models that change frequently and are critical for freshness. These should be fast (cursor-based, date-filtered, small result sets).

**Characteristics:**
- Source API supports `updatedAfter` or equivalent date filtering
- Data is time-sensitive (customer records, orders, transactions)
- Cursor-based incremental sync brings each model to single-digit seconds

### Tier 2 — Conditional
Models that rarely change but are expensive to pull. Only sync when a Tier 1 record references a parent that's missing from the database.

**Trigger mechanism:** After Tier 1 completes, run orphaned foreign key checks:
```sql
-- Does a child record reference a parent that's not in the lake?
SELECT 1 FROM "ChildTable"
WHERE "parentId" IS NOT NULL
AND "parentId" NOT IN (SELECT "id" FROM "ParentTable")
LIMIT 1
```
If the query returns a row, the parent model needs syncing. If empty, skip it.

**Characteristics:**
- Source API lacks date filtering (must pull entire catalog)
- Data is reference/catalog data (event templates, categories, price lists)
- Changes are infrequent (new catalog items published weekly/monthly)
- Only matters when something references it

### Tier 3 — Safety Net
Full catalog pull of all models, run once daily off-peak. Catches anything the conditional logic missed — stale records, updates to existing catalog items, deletions.

## Why This Works

The insight is that **reference data only matters when something references it.** A new event template in ACME is irrelevant to the data lake until someone buys a ticket to one of its events. At that point, the transaction (Tier 1) creates a foreign key reference to the event instance, which references the template. The orphaned FK check detects this and triggers the template sync.

This inverts the sync model from "pull everything in case it changed" to "pull only what's needed right now."

## Example: Barnes Foundation CRM Data Lake

| Tier | Models | Time | Frequency |
|------|--------|------|-----------|
| 1 | Orders, Contacts, Memberships, Cards, Phone, Email, Account, Address, Subscriptions, Transactions, TicketAnalytics, SalesChannel | ~56 sec | Hourly |
| 2 | EventInstance, EventTemplate, EventPriceList | 0 sec (when no orphans) to ~5 min (when triggered) | Conditional |
| 3 | All models (full catalog) | ~6 min | Daily 2 AM |

**Result:** Hourly sync dropped from 6+ minutes to under 1 minute. Data freshness for the contact center screen pop improved from 12 hours to 1 hour.

## Implementation Checklist

1. Add a `syncTier` field (1, 2, or 3) to each model's source configuration
2. Add an environment variable (`SYNC_TIER`) to control which mode runs: `ALL`, `TIER1_ONLY`, or `CONDITIONAL`
3. Implement orphaned FK check queries for each Tier 2 model's parent relationship
4. Configure cron/scheduler:
   - Hourly: `SYNC_TIER=CONDITIONAL` (Tier 1 + conditional Tier 2)
   - Daily off-peak: `SYNC_TIER=ALL` (full safety net)
5. Error handling: if an FK check fails, include the model (safe default)

## When NOT to Use This

- When all models are fast (cursor-based, date-filtered) — just run everything
- When all data is equally time-sensitive — no tier differentiation
- When the source API supports webhooks or change feeds — use those instead
- When the dataset is small enough that full pulls are cheap

## Origin

Created 2026-04-05 on crm-data-lake-sync (Barnes Foundation). The sync was optimized from 26 minutes to 6 minutes through cursor-based incremental sync, but 5 of those 6 minutes were EventTemplate and EventInstance — models that rarely change but had no `updatedAfter` support from the vendor API. The tiered approach reduced the hourly run to 56 seconds without waiting for vendor API improvements.
