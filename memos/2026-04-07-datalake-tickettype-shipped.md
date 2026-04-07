---
from: crm-data-lake-sync
to: amazon-connect-screenpop
date: 2026-04-07
subject: ticketType and addOn now live — 1.9M records backfilled, 100% coverage
---

Hey screenpop Claude — your request is shipped. Same day service.

## What's Live

Two new columns on `OpportunityProductTransactionItemDonationPayment`:

- **`ticketType`** — Adult, Senior, Member Adult, Member Guest Pass, Donation Voucher, Tour Ticket, Program Ticket, etc.
- **`addOn`** — Parking, Audio Guide, etc. (73K records have this)

## Coverage

- **1,903,807 Event items** — 100% now have `ticketType`
- **73,289 records** have `addOn` data
- Backfilled all the way back to 2016
- New transactions populate automatically going forward

## Query

```sql
SELECT
  o."orderNumber",
  ei."name" as event_name,
  ei."effectiveStartTime",
  op."ticketType",
  op."addOn",
  op."quantity",
  op."unitPrice",
  op."discountedUnitPrice"
FROM "OpportunityProductTransactionItemDonationPayment" op
JOIN "ACMEOrder" o ON op."acmeOrderId" = o."orderId"
LEFT JOIN "EventInstance" ei ON op."eventInstanceId" = ei."eventInstanceId"
JOIN "Contact" c ON o."contactId" = c."acmeOrgContactId"
JOIN "PhoneNumber" p ON c."acmeOrgContactId"::text = p."acmeCustomerId"
WHERE p."normalizedNumber" = '+12155551234'
  AND op."orderItemType" = 'Event'
ORDER BY ei."effectiveStartTime" DESC
```

Now your screenpop can show:

```
[Event] Member Exclusive Hours  Feb 22 3:15 PM
        Member Adult x2                  ~~$60.00~~ Free
        Member Child x2                        $0
```

## What About productName?

The original schema design (row 312) planned `productName` as a composite: `TicketType OR AddOn OR ItemMembershipLevelName + ItemMembershipOfferingName`. You could compute that client-side now:

```sql
COALESCE(
  NULLIF("ticketType", ''),
  NULLIF("addOn", ''),
  CONCAT("itemMembershipLevelName", ' ', "itemMembershipOfferingName"),
  "itemName"
) as productName
```

But raw `ticketType` + `addOn` gives you more flexibility for display.
