# Fixture — fictional repository content of `granary-orders`

> **ILLUSTRATIVE / NON-PRODUCTION.** These are the *only* files the example review "inspects"; line references in the report point here.

## `db/schema.cds`

```cds
namespace acme.granary;
using { cuid, managed } from '@sap/cds/common';

entity Orders : cuid, managed {          // L5
  quantity   : Integer;
  buyerEmail : String;
  internalMargin : Decimal;              // L8 — internal costing field
  status     : String;
  items      : Composition of many { key pos : Integer; grain : String; tons : Decimal; };
}
```

## `srv/catalog-service.cds`

```cds
using { acme.granary as my } from '../db/schema';

@requires: 'authenticated-user'          // L3
service CatalogService {
  @readonly entity Orders as projection on my.Orders
    excluding { internalMargin, buyerEmail };          // L6 — tailored projection
  action reorder (orderID : UUID) returns String;      // L7 — modifies → action ✓
}
```

## `srv/orders-service.cds`

```cds
using { acme.granary as my } from '../db/schema';

service OrdersService {                  // L3 — NO @requires / @restrict anywhere
  entity Orders as projection on my.Orders;            // L4 — incl. internalMargin,
}                                                       //      buyerEmail
```

## `srv/internal-recalc.cds`

```cds
using { acme.granary as my } from '../db/schema';

service RecalcHelper {                   // L3 — internal helper, but not annotated
  action recalcAll();                    //      @protocol:'none' → served publicly
}
```

## `package.json` (excerpt)

```jsonc
{
  "name": "granary-orders",                          // L2
  "dependencies": { "@sap/cds": "^10", "@cap-js/hana": "^3" },
  "devDependencies": { "@cap-js/sqlite": "^3", "@cap-js/cds-test": "^1" },
  "cds": { "requires": { "auth": "mocked",
    "[production]": { "auth": { "kind": "xsuaa" } } } }
}
```

## Validation annotations — `srv/annotations.cds`

```cds
using { OrdersService } from './orders-service';
annotate OrdersService.Orders with {
  quantity @assert.range: [1, 10000];    // L4
  status   @readonly;                    // L5
};
```

## Notes the review will use

- No `@odata.etag` annotation exists anywhere, and no ADR or requirement note records a concurrency decision for `Orders` — although `workload.concurrent_edit: true` (profile) and the UI edits orders.
- No test files exist yet for authorization denial (M3 starts them; M6/M7 complete them).
