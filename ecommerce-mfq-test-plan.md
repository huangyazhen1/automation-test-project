# 1. Strategy Note (1 Page)
## Test Priorities
- P0 (Critical): Inventory synchronization consistency, order state machine, payment idempotency, inventory reserve/deduct/restore
- P1 (High): Core order flow (place→pay→ship→sign), WMS/SC2P external interaction, high-concurrency flash sale
- P2 (Medium): Exception handling, compensation strategy, critical API validation
- P3 (Low): Boundary scenarios (zero stock, partial shipment), return inventory restoration

## Key Trade-offs
- Consistency: Strong consistency for inventory reserve/deduct (avoid over-selling), eventual consistency for WMS/SC2P sync (balance performance)
- External Interaction: Retry + fallback + circuit breaker for WMS/SC2P calls, with compensation jobs for sync failures
- Performance: Prioritize atomicity of inventory deduction + order creation in flash sales; asynchronous processing for non-core links (notifications)
- Observability: Focus on 3 core metrics (inventory diff ratio, message backlog, stuck orders) for rapid fault locating

# 2. M – Models
## 2.1 End-to-End Business Flow Model
1. User browses products → Adds to cart
2. Places order → System reserves inventory
3. Completes payment → System deducts inventory
4. System syncs order/inventory to WMS → WMS processes shipment
5. User signs for goods → Order status updates to "Signed"
6. (Optional) User applies for return → System reviews → Restores inventory → Order closes

## 2.2 Order State Machine
- States: Created → Paid → Shipped → Signed → Returned → Closed
- Events: Pay, Cancel, Ship, Sign, Return Application, Return Approval
- Transition Conditions:
  - Created → Paid: Payment success + inventory deduction success
  - Created → Closed: Payment timeout (≥30min) or user cancels (before payment)
  - Paid → Shipped: WMS receives sync message and confirms shipment
  - Shipped → Signed: User confirms receipt or logistics callback confirms delivery
  - Signed → Returned: Return application approved + goods returned
  - Returned/Paid/Shipped → Closed: After-sales completed or order expires

## 2.3 Inventory State Model
- Core States: Available → Reserved → Deducted → Sold
- State Transitions:
  - Available → Reserved: Order placed (unpaid)
  - Reserved → Deducted: Payment successful
  - Deducted → Sold: User signs for goods (no return)
  - Reserved → Available: Order canceled (before payment) or payment timeout
  - Deducted → Available: Return approved + goods returned
  - Deducted → Reserved: Partial return (only part of inventory restored)

## 2.4 External Interaction Model
- Storefront ↔ WMS:
  - Storefront → WMS: Order shipment request, return request, inventory query
  - WMS → Storefront: Shipment status callback, delivery confirmation, inventory sync result
- Storefront ↔ SC2P:
  - Storefront → SC2P: Inventory data sync, order data sync
  - SC2P → Storefront: Inventory adjustment callback, data reconciliation result

## 2.5 Consistency Boundaries
- Strong Consistency: Inventory reserve/deduct operations (atomic transaction, avoid data inconsistency)
- Eventual Consistency: WMS/SC2P data sync (allow short-term inconsistency, rely on reconciliation)
- Reconciliation Points:
  - Hourly: Inventory data reconciliation between storefront and WMS/SC2P
  - Daily: Full order + inventory reconciliation (24:00 daily)
  - Abnormal Trigger: Inventory diff ratio ≥0.5% (automatic reconciliation)

# 3. F – Functional
## 3.1 Core Happy-Path Coverage
1. Order Creation: User selects SKU → Adds to cart → Places order → Reserve inventory successfully
2. Payment: Selects payment method → Pays successfully → Deduct inventory → Sync to WMS
3. Shipment: WMS receives sync → Processes shipment → Callback to storefront (update order status)
4. Sign: User signs for goods → Order status updates to "Signed" → Order closes normally

## 3.2 Inventory Synchronization Scenarios
- Reserve Success: Order placed → Inventory reserved → Available inventory decreased, reserved inventory increased
- Reserve Failure: Insufficient available inventory → Reserve failed → Order creation rejected
- Deduct Success: Payment successful → Reserved inventory decreased, deducted inventory increased
- Deduct Failure: Payment successful but inventory deduct failed → Automatic compensation (restore reserved inventory + refund)
- Cancel and Restore: Order canceled (before payment) → Reserved inventory restored to available
- Idempotency for Duplicate Messages: Duplicate WMS/SC2P sync messages → System ignores (idempotent check via message ID)
- Out-of-Order Message Handling: Shipment callback arrives before payment callback → System caches message → Processes after payment is confirmed

## 3.3 Boundary Scenarios
- Zero Stock: User tries to place order for out-of-stock SKU → Order creation rejected + prompt "Out of stock"
- High-Concurrency Flash Sale: 1000+ users place orders simultaneously → Inventory reserved/deducted correctly (no over-selling/under-selling)
- Partial Shipment/Split Shipment: Order contains multiple SKUs → Some SKUs shipped first, others shipped later → Inventory deducted partially, order status updated to "Partially Shipped"
- Return and Stock Restoration: Return approved → Deducted inventory restored to available → WMS syncs return receipt → Inventory data consistent

## 3.4 Critical API Validations
### Order API
- Request Fields: Required (user ID, SKU ID, quantity, address ID); Optional (coupon ID, remark)
- Error Codes: 400 (missing required fields), 404 (SKU not found), 409 (insufficient inventory)
- Callback/Writeback Fields: Order ID, status, operation time, inventory change amount

### Inventory API
- Request Fields: Required (inventory type, SKU ID, quantity); Optional (order ID)
- Error Codes: 400 (invalid inventory type), 409 (insufficient available/reserved inventory)
- Callback/Writeback Fields: SKU ID, available inventory, reserved inventory, deducted inventory

### WMS/SC2P Sync API
- Request Fields: Required (order ID, SKU ID, operation type); Optional (logistics ID)
- Error Codes: 503 (WMS/SC2P unavailable), 400 (invalid sync data)
- Callback/Writeback Fields: Sync status, callback time, error message (if failed)

# 4. Q – Quality
## 4.1 Performance
- Concurrency Target: ≥1000 TPS (peak flash sale scenario)
- Latency: Order creation/payment ≤200ms; Inventory sync ≤500ms; WMS callback response ≤1s
- Error Rate: ≤0.1% (all core operations)
- Throughput: ≥10000 orders/hour (peak period)

## 4.2 Reliability
- Retry Mechanism: External API calls (WMS/SC2P) → 3 retries (interval: 1s, 3s, 5s)
- Fallback Strategy: WMS unavailable → Order status set to "Pending Shipment" + alert; Payment gateway unavailable → Prompt user to try again later
- Timeout Control: All external calls ≤3s; Order payment timeout = 30min; Inventory sync timeout = 5s
- Circuit Breaker: WMS/SC2P error rate ≥50% → Circuit breaker open (stop calling for 5min) → Switch to manual processing
- Compensation Jobs:
  - Inventory inconsistency → Hourly automatic reconciliation + inventory adjustment
  - Order stuck (no status update for 1h) → Automatic timeout close + inventory restore
  - Sync failure → Daily batch compensation + alert to operation team

## 4.3 Observability
### Logs
- Order Logs: Record all state transitions (operation time, operator, status before/after)
- Inventory Logs: Record all inventory changes (type, quantity, order ID, sync status)
- External Sync Logs: Record WMS/SC2P request/response data, error messages
- Security Logs: Record user authentication, permission operations

### Metrics
- Inventory Metrics: Inventory diff ratio (storefront vs WMS/SC2P), available/reserved/deducted inventory count
- Order Metrics: Order creation/payment/shipment/signed conversion rate, stuck order count
- Sync Metrics: Message backlog count, sync success rate, callback delay
- Performance Metrics: TPS, latency, error rate

### Alerts
- Inventory: Inventory diff ratio ≥0.5%, available inventory ≤10 (key SKUs)
- Order: Stuck order count ≥5, order error rate ≥0.5%
- Sync: Message backlog ≥100, sync success rate ≤95%
- Performance: Latency ≥500ms (continuous 5min), TPS ≤500 (peak period)

## 4.4 Security
- Authentication: User login via account/password + SMS verification; API calls via token (valid for 2h)
- Permission Boundaries:
  - Ordinary User: Only operate own orders/inventory (no access to other users' data)
  - Merchant: Manage own products/inventory/orders (no access to platform data)
  - Admin: Full access (order/inventory/user management)
- Sensitive Data Masking:
  - User phone: Hide middle 4 digits (e.g., 138****1234)
  - User address: Hide detailed street (e.g., Shanghai Pudong New Area ****)
  - Payment information: No plaintext storage (encrypted via AES)

## 4.5 Recoverability
### Failure Drills
- Simulate WMS downtime: Verify fallback strategy + compensation job effectiveness
- Simulate inventory inconsistency: Verify automatic reconciliation accuracy
- Simulate high-concurrency flash sale: Verify system stability + no over-selling
- Simulate payment gateway failure: Verify user experience + order consistency

### Data Repair Process
- Inventory Data: Manual adjustment (requires admin approval) + record repair log
- Order Data: Manual status correction (requires 2 admin approvals) + compensation for affected users
- Sync Data: Manual re-sync (batch/individual) + reconciliation after sync

# 5. Inventory Synchronization Consistency (Dedicated Section)
## 5.1 Consistency Guarantee Mechanism
1. Atomic Transaction: Inventory reserve/deduct operations are wrapped in transactions (either all succeed or all fail)
2. Real-Time Sync: Order/inventory changes are synchronized to WMS/SC2P in real time (async but ordered)
3. Idempotent Check: All sync messages have unique IDs → Avoid duplicate processing
4. Reconciliation Mechanism: Hourly incremental reconciliation + daily full reconciliation

## 5.2 Inconsistency Handling
- Detection: System compares storefront inventory with WMS/SC2P inventory → Calculate diff ratio
- Classification:
  - Minor Inconsistency (diff ratio <0.5%): Automatic adjustment (take storefront data as standard)
  - Major Inconsistency (diff ratio ≥0.5%): Alert to operation team → Manual verification + adjustment
- Root Cause Analysis: Log sync process → Locate failure points (e.g., network interruption, WMS error) → Optimize sync mechanism

# 6. Exception Handling & Compensation Strategy
## 6.1 Common Exceptions & Handling
1. Inventory Reserve Failure:
   - Exception: Insufficient inventory, system error during reserve
   - Handling: Reject order creation → Prompt user "Insufficient inventory" → Log error
2. Inventory Deduct Failure (Payment Successful):
   - Exception: System error, network interruption
   - Handling: Automatic compensation (restore reserved inventory + initiate refund) → Alert operation team
3. WMS Sync Failure:
   - Exception: WMS unavailable, sync message lost
   - Handling: Retry 3 times → Fallback to "Pending Shipment" → Hourly re-sync + alert
4. Duplicate Sync Messages:
   - Exception: WMS/SC2P sends duplicate messages
   - Handling: Idempotent check (via message ID) → Ignore duplicate messages → Log
5. Order Stuck:
   - Exception: No status update for 1h (e.g., payment successful but no shipment)
   - Handling: Automatic timeout close → Restore inventory (if applicable) → Alert operation team

## 6.2 Compensation Strategy Principles
- Timeliness: Automatic compensation within 10min for critical exceptions (e.g., deduct failure)
- Atomicity: Compensation operations are atomic (avoid secondary inconsistency)
- Traceability: All compensation operations are logged (operation time, operator, details)
- Manual Backup: Automatic compensation failed → Manual compensation (requires admin approval)

# 7. Release Entry & Exit Criteria
## 7.1 Entry Criteria
1. All P0/P1 functional modules are developed and integrated
2. Unit testing pass rate ≥95% (core modules 100%)
3. Test environment is ready (consistent with production environment)
4. Test data is prepared (normal/abnormal scenarios, high-concurrency data)
5. WMS/SC2P test environment is available (for external sync testing)

## 7.2 Exit Criteria
1. No P0/P1 bugs (critical/major bugs)
2. P2 bugs ≤5 (minor bugs) and no recurrence after repair
3. Inventory synchronization consistency test pass rate 100%
4. Performance meets target (TPS, latency, error rate)
5. All exception handling and compensation scenarios are verified
6. Observability metrics/alerts are configured and effective
7. Security vulnerability scan pass (no high-risk vulnerabilities)
8. Test report is completed (includes test results, risk assessment)
