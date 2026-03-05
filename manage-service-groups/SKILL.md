---
name: manage-service-groups
description: Orchestrate multiple services using the OFBiz Group Engine for sequential or concurrent execution.
---

# Manage Service Groups

Service Groups allow you to define a single service that acts as a wrapper for multiple child services. This is ideal for orchestrating complex workflows where several independent services need to run together.

## Triggers
- Executing multiple services in a specific sequence.
- Running services concurrently to improve performance.
- Wrapping multiple operations in a single transactional unit.
- Scaffolding complex integration flows (e.g., "Complete Purchase Order" which might involve receiving items, creating invoices, and posting to GL).

## Procedures

### 1. Define the Group Service
In your `services.xml`, define a service using `engine="group"`.

```xml
<service name="myServiceGroup" engine="group" auth="true">
    <description>Orchestrates multiple services</description>
    <attribute name="orderId" mode="IN" type="String"/>
    <!-- Groups often return the combined results or specific attributes -->
</service>
```

### 2. Define the Group Members
Create a `groups.xml` (or similar) file to define which services belong to the group.

- **Location**: Typically `servicedef/groups.xml`.
- **Registration**: Register in `ofbiz-component.xml`:
  ```xml
  <service-resource type="group" loader="main" location="servicedef/groups.xml"/>
  ```

#### Group Definition Pattern
```xml
<service-group name="myServiceGroup">
    <invoke name="firstService" mode="sync"/>
    <invoke name="secondService" mode="async" pause="1000"/>
    <invoke name="thirdService" mode="sync" optional="true"/>
</service-group>
```

**Attributes for `<invoke>`**:
- `name`: The service to call.
- `mode`: `sync` (sequential) or `async` (parallel/background).
- `optional`: If `true`, the group continues even if this service fails.
- `result-to-context`: If `true`, the results of this service are added to the context for subsequent services in the group.

---

## Guardrails
- **Transaction Management**: By default, a group service runs in a single transaction. If a `sync` member fails and isn't `optional="true"`, the entire transaction rolls back.
- **Context Pollution**: Be aware that services in a group share the context if `result-to-context="true"`. Ensure attribute names don't clash.
- **Async Pitfalls**: If you use `mode="async"` within a group, that service runs in its own thread/transaction. The group won't wait for it to finish unless specifically architected otherwise.
- **Service Engine**: Use `engine="group"` only for orchestration. Don't put business logic in the group definition itself.

## Examples

### Sequential Purchase Order Completion
Executing actions in a strict order.
```xml
<service-group name="completePurchaseOrder">
    <invoke name="receiveInventory" mode="sync"/>
    <invoke name="createPurchaseInvoice" mode="sync"/>
    <invoke name="postToGeneralLedger" mode="sync" optional="true"/>
</service-group>
```

### Concurrent Notifications
Triggering multiple independent notifications.
```xml
<service-group name="sendMassNotifications">
    <invoke name="sendEmailNotification" mode="async"/>
    <invoke name="sendSMSNotification" mode="async"/>
    <invoke name="logNotificationStats" mode="sync"/>
</service-group>
```
