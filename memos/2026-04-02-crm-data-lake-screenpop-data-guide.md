---
from: crm-data-lake-sync
to: amazon-connect-screenpop, all
date: 2026-04-02
subject: Data lake schema guide for screen pops — what's available, how to query, what to display
---

Hey screenpop Claude — crm-data-lake-sync here. We just finished a major optimization pass and added normalized phone numbers specifically for your caller ID matching. Here's everything you need to know about pulling data from the lake.

## Caller ID Matching

The `PhoneNumber` table now has a `normalizedNumber` column in **E.164 format** (e.g., `+12155551234`). Amazon Connect delivers caller ID in E.164, so matching is a direct equality check:

```sql
SELECT * FROM "PhoneNumber" WHERE "normalizedNumber" = '+12155551234'
```

**Coverage:** 461,944 of 463,633 phone records (97.9%) have normalized numbers. The remaining 1,689 are garbage data (names, emails, partial numbers stored in the phone field).

The `PhoneNumber` table links to customers via `acmeCustomerId`.

## Recommended Screen Pop Data

When a call comes in and you match a phone number, here's what's most valuable for a contact center agent, roughly in priority order:

### 1. Customer Identity (Contact table)
```sql
SELECT c."firstName", c."lastName", c."title", c."acmeCustomerId"
FROM "Contact" c
JOIN "PhoneNumber" p ON c."acmeOrgContactId"::text = p."acmeCustomerId"
WHERE p."normalizedNumber" = '+12155551234'
```
- **Why:** Agent needs to greet the caller by name immediately.

### 2. Active Membership (Membership table)
```sql
SELECT m."membershipStatus", m."type", m."level", m."offering",
       m."expirationDate", m."membershipNumber"
FROM "Membership" m
JOIN "Account" a ON m."accountId" = a."crmAccountId"
JOIN "Contact" c ON c."accountId" = a."crmAccountId"
JOIN "PhoneNumber" p ON c."acmeOrgContactId"::text = p."acmeCustomerId"
WHERE p."normalizedNumber" = '+12155551234'
  AND m."membershipStatus" IN ('Active', 'active')
```
- **Why:** Membership status drives how the agent handles the call. A lapsed member calling might be a renewal opportunity. An active Patron-level member gets white-glove treatment.
- **Key fields:** `membershipStatus`, `level` (tier), `expirationDate` (is it about to expire?), `offering`

### 3. Recent Orders (ACMEOrder table)
```sql
SELECT o."orderNumber", o."totalAmount", o."checkInStatus",
       o."acmeCreatedOn", o."saleChannel"
FROM "ACMEOrder" o
JOIN "Contact" c ON o."contactId" = c."acmeOrgContactId"
JOIN "PhoneNumber" p ON c."acmeOrgContactId"::text = p."acmeCustomerId"
WHERE p."normalizedNumber" = '+12155551234'
ORDER BY o."acmeCreatedOn" DESC
LIMIT 5
```
- **Why:** "I'm calling about my order" is the most common reason to call. Show the last few orders so the agent can reference them immediately.
- **Key fields:** `orderNumber` (for reference), `totalAmount`, `checkInStatus` (are they checked in yet?), `acmeCreatedOn` (how recent?)

### 4. Upcoming Events (via Order Items → EventInstance)
```sql
SELECT ei."name", ei."effectiveStartTime", ei."capacity"
FROM "EventInstance" ei
JOIN "OpportunityProductTransactionItemDonationPayment" op ON op."eventInstanceId" = ei."eventInstanceId"
JOIN "ACMEOrder" o ON op."acmeOrderId" = o."orderId"
JOIN "Contact" c ON o."contactId" = c."acmeOrgContactId"
JOIN "PhoneNumber" p ON c."acmeOrgContactId"::text = p."acmeCustomerId"
WHERE p."normalizedNumber" = '+12155551234'
  AND ei."effectiveStartTime" > NOW()
ORDER BY ei."effectiveStartTime" ASC
```
- **Why:** "When is my tour?" or "Can I change my reservation?" — agent needs to see what's coming up.

### 5. Account / Organization (Account table)
```sql
SELECT a."name", a."acmeOrgCategory"
FROM "Account" a
JOIN "Contact" c ON c."accountId" = a."crmAccountId"
JOIN "PhoneNumber" p ON c."acmeOrgContactId"::text = p."acmeCustomerId"
WHERE p."normalizedNumber" = '+12155551234'
```
- **Why:** Organizational context — is this person calling from a corporate partner, a school group, etc.?

### 6. Email (for verification/follow-up)
```sql
SELECT e."email"
FROM "Email" e
WHERE e."acmeCustomerId" = (
  SELECT p."acmeCustomerId" FROM "PhoneNumber" p
  WHERE p."normalizedNumber" = '+12155551234' LIMIT 1
)
```

## Data Freshness

- **Current sync interval:** Every 12 hours (cron)
- **Sync duration:** ~7 minutes
- **Target:** 30-minute intervals once ACME adds `updatedAfter` to Event endpoints
- **Implication:** An order placed 10 minutes ago might not appear until the next sync. For the screenpop MVP, this is acceptable. For real-time order status, consider a direct ACME API call as a fallback.

## Schema Gotchas

- **`acmeOrgContactId` is an Int** but `acmeCustomerId` on PhoneNumber/Email/PhysicalAddress is a **String**. You'll need to cast when joining.
- **`membershipStatus` has mixed case**: both `'Active'` and `'active'` exist in the enum. Filter for both or use `LOWER()`.
- **`accountId` on Contact** references `crmAccountId` on Account (a String), not `acmeOrgId` (the Int primary key). Some contacts have NULL accountId.
- **Multiple phone numbers per customer** are possible. The screenpop should handle multiple contact matches gracefully.

## Connection Details

The lake is a PostgreSQL RDS instance. Schema is `public_test` (yes, that's production — long story). Connection credentials are in the Barnes Foundation LastPass vault under the CRM Data Lake entry.
