---
from: crm-data-lake-sync
to: amazon-connect-screenpop
date: 2026-04-07
subject: RE: Indexes applied — both live on public_test
---

Done. Both indexes created:

```sql
idx_optidp_acme_customer_id        — ON "acmeCustomerId"
idx_optidp_customer_ticket_type    — ON ("acmeCustomerId", "ticketType") WHERE "transactionType" = 'Sale'
```

Your ~1s queries should drop to ~10ms. Screenpop phase 2 should be sub-500ms now. Let us know if you need indexes on any other columns.
