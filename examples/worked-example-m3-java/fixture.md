# Fixture — fictional repository content of `millhouse-approvals` (CAP Java)

> **ILLUSTRATIVE / NON-PRODUCTION.** The only files the example review "inspects"; report line references point here. No real credentials anywhere.

## `pom.xml` (excerpt)

```xml
<project>                                                     <!-- L1 -->
  <groupId>com.acme.millhouse</groupId>
  <artifactId>millhouse-approvals</artifactId>
  <properties><java.version>21</java.version></properties>    <!-- L5 -->
  <dependencyManagement><dependencies>
    <dependency>
      <groupId>com.sap.cds</groupId>
      <artifactId>cds-services-bom</artifactId>
      <version>5.0.0</version> <type>pom</type> <scope>import</scope>  <!-- L10 -->
    </dependency>
  </dependencies></dependencyManagement>
  <dependencies>
    <dependency><groupId>com.sap.cds</groupId><artifactId>cds-starter-cloudfoundry</artifactId></dependency>
  </dependencies>
</project>
```

## `db/schema.cds`

```cds
namespace acme.millhouse;
using { cuid, managed } from '@sap/cds/common';

entity Approvals : cuid, managed {       // L5
  amount    : Decimal;
  requester : String;
  note      : String;
  payroll   : Association to PayrollRecords;   // L9 — link to stricter entity
}
entity PayrollRecords : cuid {           // L11
  employee  : String;
  salary    : Decimal;                   // L13 — sensitive
}
```

## `srv/approval-service.cds`

```cds
using { acme.millhouse as my } from '../db/schema';

@requires: 'Approver'                    // L3 — service protected ✓
service ApprovalService {
  entity Approvals as projection on my.Approvals;   // L5 — payroll association
                                                     //      exposed, target has
                                                     //      stricter restriction
  action approve (ID : UUID) returns String;         // L8 — modifies → action ✓
}
```

## `srv/access-control.cds`

```cds
using { acme.millhouse as my } from '../db/schema';
annotate my.PayrollRecords with @(restrict: [
  { grant: '*', to: 'PayrollAdmin' }     // L4 — stricter than 'Approver'
]);
annotate ApprovalService.Approvals with {
  amount @assert.range: [ 0, 500000 ];   // L7
  note   @mandatory;                     // L8
};
```

## `srv/src/main/java/com/acme/millhouse/ApprovalHandler.java`

```java
package com.acme.millhouse;                                      // L1
@Component
@ServiceName(ApprovalService_.CDS_NAME)
public class ApprovalHandler implements EventHandler {            // L4 ✓ pattern
  @On(event = "approve", entity = Approvals_.CDS_NAME)            // L5 — @On for the
  public void onApprove(ApproveContext ctx) {                     //      action ✓
    ctx.setResult("approved"); ctx.setCompleted();
  }
}
```

## `srv/src/test/java/com/acme/millhouse/ApprovalServiceIT.java`

```java
@SpringBootTest @AutoConfigureMockMvc                             // L1
class ApprovalServiceIT {
  @Autowired MockMvc mvc;
  @Test void rejectsUnauthenticated() throws Exception {
    mvc.perform(get("/api/approval/Approvals")).andExpect(status().isUnauthorized());  // L5
  }
}
```

## `srv/src/main/resources/application.yaml`

```yaml
cds:                                     # L1
  odataV4.endpoint.path: /api            # L2
# NOTE: no @protocol restriction anywhere, no odata-v2 decision —
# CAP Java serves BOTH odata-v4 AND odata-v2 by default (L4)
```

## Notes the review will use

- `ApprovalService.Approvals` exposes the `payroll` association (approval-service.cds L5) whose target `PayrollRecords` carries a **stricter** restriction (`PayrollAdmin`, access-control.cds L4). Java does not enforce target restrictions on expand paths of exposed associations without mitigation — no exclusion, no equivalent root restriction, no custom handler exists.
- No `@protocol`/`@protocols` annotation and no adapter configuration anywhere: the documented Java default serves the service on **both** OData V4 and V2 — no recorded V2 decision exists.
- No M4-relevant defects are staged (the M4 selection demonstration is set-level only).
