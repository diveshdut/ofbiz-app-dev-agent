---
name: manage-service-groups
description: Orchestrate multiple services using the OFBiz Group Engine for sequential or concurrent execution.
---

# Manage Service Groups

Service Groups allow you to define a single service that acts as a wrapper for multiple child services. This is ideal for orchestrating complex workflows where several independent services need to run together.

## Goal
The goal of this skill is to correctly define and execute Service Groups, understanding when to use the `group` engine versus a Groovy orchestrator, and properly configuring execution modes and context passing.

## Triggers
- Executing multiple services in a specific sequence.
- Running services concurrently to improve performance.
- Wrapping multiple operations in a single transactional unit.
- Scaffolding complex integration flows (e.g., "Complete Purchase Order" which might involve receiving items, creating invoices, and posting to GL).

## Rules & Procedures

### 1. Engine vs Workflow Definitions
- **Group Engine (`engine="group"`)**: Use exclusively for straight-line sequential or concurrent fire-and-forget flows that do not have complex conditional branching or data dependencies between steps.
- **Groovy Engine (`engine="groovy"`)**: Use if your workflow requires conditional `if/else` logic, try/catch error handling, complex iteration, or complex data transformations between service steps.

### 2. Define the Group Service (`services.xml`)
In your `services.xml`, define a service using `engine="group"`. Ensure you include the `invoke` and `use-transaction` attributes. (**Cross-Reference**: See the `manage-services` skill for comprehensive rules on service definitions, authentication, and REST exports.)

```xml
<service name="myServiceGroup" engine="group" invoke="myServiceGroup"
         auth="true" use-transaction="true">
    <description>Orchestrates multiple services</description>
    <attribute name="orderId" mode="IN" type="String"/>
</service>
```

### 3. Define the Group Members (`groups.xml`)
Create a `groups.xml` (or similar) file to define which services belong to the group. Wrap your `<invoke>` elements inside the `<group>` element.

- **Location**: Typically `servicedef/groups.xml`.
- **Registration**: Register in `ofbiz-component.xml`:
  ```xml
  <service-resource type="group" loader="main" location="servicedef/groups.xml"/>
  ```

#### Group Definition Pattern
```xml
<service-group>
    <group name="myServiceGroup" send-mode="all">
        <invoke name="firstService" mode="sync"/>
        <invoke name="secondService" mode="async" pause="1000"/>
        <invoke name="thirdService" mode="sync" optional="true"/>
    </group>
</service-group>
```

### 4. Group Attributes (`send-mode`)
The `<group>` element defines how the child services are executed.
- `name`: Must match the service name defined in `services.xml`.
- `send-mode`: Defines execution strategy. **Default is `all`**.
  - **`all` (Default)**: Executes all services in the group. Use for standard sequential/parallel execution.
  - **`first-available`**: Executes the first service that does not fail. Execution stops upon first success. Use as a fallback mechanism.
  - **`none`**: Does not execute any services. Use for temporary disabling/debugging.
  - **`round-robin`**: Executes one service in the round-robin order. Use for load balancing identical worker tasks.
  - **`random`**: Executes one randomly selected service. Use for A/B testing or random distribution.

### 5. Invoke Attributes (`mode` and `result-to-context`)
The `<invoke>` element defines the individual service calls.
- `name`: The service to call.
- `optional`: If `true`, the group continues execution even if this specific service fails.
- `mode`: `sync` or `async`.
  - **`sync` (Synchronous)**: The group waits for this service to complete. Use when subsequent steps depend on these results or for transaction integrity.
  - **`async` (Asynchronous)**: The group fires this service in a separate thread and moves on immediately. Use for non-blocking fire-and-forget background tasks. **Never** use `async` when subsequent steps depend on its results.
- `result-to-context`: If `true` AND `mode="sync"`, the results of this service are pushed into the context for subsequent services. **Default is `false`**.

---

## Guardrails
- **Transaction Behavior**: A group service is just a service call. The transaction boundaries follow the service attributes (`use-transaction`, `require-new-transaction`) of the group service and/or the invoked child services. There is no hard rule that a group always equals a single transaction. (**Cross-Reference**: See the `manage-services` skill for deep-dives into `use-transaction` versus `require-new-transaction` mapping).
- **Context collisions**: Context passing only happens when `result-to-context="true"` for `sync` steps. Context collisions are a risk here; encourage prefixing output keys or mapping results in wrapper services.

---

## Anti-Patterns

### Anti-Pattern 1: Hidden Branching Logic
**Bad:** Using `optional="true"` to simulate an `if/else` block.
```xml
<group name="processPayment" send-mode="all">
    <invoke name="chargeStripe" mode="sync" optional="true"/>
    <!-- If Stripe fails, just keep running and hope it was handled -->
    <invoke name="logPaymentSuccess" mode="sync"/>
</group>
```

**Good:** Using a Groovy orchestrator service for real conditional logic. (**Cross-Reference**: See the `manage-groovy` skill)
```groovy
def stripeResult = run service: "chargeStripe", with: context
if (ServiceUtil.isError(stripeResult)) {
    logWarning("Stripe failed, trying alternative...")
    // Handle error or branch
} else {
    run service: "logPaymentSuccess", with: context
}
```

### Anti-Pattern 2: The `result-to-context` Trap
**Bad:** Overwriting generic context variables (e.g., `statusId`) across multiple services.
```xml
<group name="createOrderAndInvoice" send-mode="all">
    <invoke name="createOrder" mode="sync" result-to-context="true"/>
    <!-- createOrder drops 'statusId' into context. createInvoice reads standard statusId and gets the order's status instead of the invoice's -->
    <invoke name="createInvoice" mode="sync" result-to-context="true"/>
</group>
```

**Good:** Explicitly mapping values in a Groovy service instead of blindly sharing context. (**Cross-Reference**: See the `manage-groovy` skill)
```groovy
def orderMap = run service: "createOrder", with: orderCtx
def invoiceCtx = [orderId: orderMap.orderId, partyId: context.partyId]
def invoiceMap = run service: "createInvoice", with: invoiceCtx
```

### Anti-Pattern 3: Async Race Conditions
**Bad:** Expecting a `sync` step 2 to wait for an `async` step 1.
```xml
<group name="reportFlow" send-mode="all">
    <!-- Returns immediately, report is generating in background -->
    <invoke name="generateReport" mode="async"/>
    <!-- Runs immediately, fails because report doesn't exist yet -->
    <invoke name="emailReport" mode="sync"/>
</group>
```

**Good:** Run both as `sync` in a fire-and-forget orchestrator, OR use a SECA on `generateReport` success. (**Cross-Reference**: See the `manage-eca` skill)

### Anti-Pattern 4: Thread Exhaustion with `pause`
**Bad:** Putting the Tomcat web thread to sleep using `pause`.
```xml
<group name="waitAndRetry" send-mode="all">
    <invoke name="waitForExternalSystem" mode="sync" pause="5000"/>
</group>
```

**Good:** Use explicit asynchronous scheduled jobs instead of pausing the current thread.

### Anti-Pattern 5: Distributed Transaction Failures
**Bad:** Mixing local database commits with external ERP REST calls where compensation is impossible.
```xml
<group name="fulfillOrder" send-mode="all">
    <invoke name="updateInventoryDB" mode="sync"/> <!-- Commits -->
    <invoke name="notifyERP" mode="sync"/> <!-- If this fails, DB is NOT rolled back -->
</group>
```

**Good:** Avoid doing this in Service Groups. Use manual Groovy transaction catch blocks (`manage-groovy`) or `global-rollback` SECAs (`manage-eca`). 

### Anti-Pattern 6: Output Parameter Masking
**Bad:** Relying on Service Groups to smartly map specific `OUT` parameters back to the original caller. Group services blindly gather whatever was thrown into the context.

**Good:** Use a Groovy wrapper service to explicitly construct and `return [invoiceId: subResult.invoiceId]`. (**Cross-Reference**: See the `manage-groovy` skill)

---

## Examples

### Sequential Purchase Order Completion
Executing actions in a strict order.
```xml
<service-group>
    <group name="completePurchaseOrder" send-mode="all">
        <invoke name="receiveInventory" mode="sync"/>
        <invoke name="createPurchaseInvoice" mode="sync"/>
        <invoke name="postToGeneralLedger" mode="sync" optional="true"/>
    </group>
</service-group>
```

### Concurrent Notifications
Triggering multiple independent notifications asynchronously.
```xml
<service-group>
    <group name="sendMassNotifications" send-mode="all">
        <invoke name="sendEmailNotification" mode="async"/>
        <invoke name="sendSMSNotification" mode="async"/>
        <invoke name="logNotificationStats" mode="sync"/>
    </group>
</service-group>
```
