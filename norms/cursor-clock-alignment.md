# Cursor Clock Alignment in Incremental Sync

## The Rule

When using a database column as a cursor for incremental API sync, the cursor MUST reflect the **source system's timestamp**, not your system's write timestamp. These are different clocks that drift apart.

## Why This Matters

On CRM Data Lake Sync (Barnes Foundation, 2026-04-03), the MembershipContactRole_MembershipCard model used Prisma's `@updatedAt` (the time *we* wrote the record) as the cursor for ACME's `updatedStartTime` API filter (the time *ACME* modified the record). Our write time is always later than ACME's modification time — typically seconds, sometimes minutes.

**The incident**: New memberships created in ACME overnight at 23:45-02:19 were synced into the Membership table. But when the card model ran, its cursor (our write time from a previous sync) was at 13:36 — already past those memberships' ACME modification times. The API correctly returned "nothing modified after 13:36" and the cards were permanently skipped.

**The fix**: Changed the card model's cursor to read from `Membership.modifiedDate` (ACME's timestamp) instead of its own `lastModifiedDate` (Prisma's `@updatedAt`). Added a `cursorTable` config option for models that need to cursor off a related table.

## The Principle

> The cursor must live on the same clock as the API filter it feeds.

If the API filters by "records modified after X" using the source system's clock, then X must come from the source system's clock — not yours, not UTC wall time, not your ORM's auto-timestamp.

## Common Cursor Clock Mismatches

| Cursor Source | API Filter | Problem |
|---|---|---|
| ORM `@updatedAt` | Source system's `updatedAfter` | Your write time > source time. Cursor races ahead. |
| `Date.now()` at sync start | Source `updatedAfter` | Wall clock may differ from source system clock. |
| Source `modifiedDate` | Source `updatedAfter` | **Correct.** Same clock. |

## Additional Risk: Non-Atomic Source Timestamps

Even with the correct clock, source systems don't guarantee perfect atomicity. A record timestamped at 09:31:45 might not be available via the API until 09:31:49. If your cursor advances to 09:31:47 (the latest record you saw), you'll miss it.

**Mitigation**: Accept a small overlap window. Subtract a buffer (e.g., 30 seconds) from the cursor value before using it as the API filter. Re-pulling a few records you already have is cheap — the operation identifier will detect no changes and skip them. Missing records is expensive.

## Applying This

1. For each model in your sync, identify: what clock does the API filter against?
2. Ensure the cursor column stores a value from that same clock
3. If the model doesn't store the source timestamp directly, use `cursorTable` (or equivalent) to read from a related table that does
4. Consider a small buffer subtraction for non-atomic source systems
