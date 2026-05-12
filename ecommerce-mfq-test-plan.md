MFQ Framework Test Plan for E-commerce System
Full Markdown Answer (Directly Submit to GitHub)
1. Strategy Note (1 Page)
1.1 Test Priorities
P0 (Critical): Inventory synchronization consistency, order state machine, payment idempotency, inventory deduction/restore
P1 (High): Core order flow, WMS/SC2P external interaction, high-concurrency flash sale
P2 (Medium): Exception handling, compensation strategy, API validation
P3 (Low): Boundary scenarios, partial shipment, return restore
1.2 Key Trade-offs
Use strong consistency for core inventory actions (reserve/deduct)
Use eventual consistency for external system sync (WMS/SC2P)
Apply retry + fallback + compensation for all external calls
Focus observability on inventory diff, message backlog, stuck orders
2. M – Models
2.1 End-to-End Business Flow Model
Browse product → Add to cart
Place order → Reserve inventory
Pay → Deduct inventory
WMS/SC2P sync → Ship
Sign for goods
Return → Restore inventory
2.2 Order State Machine
States: Created → Paid → Shipped → Signed → Returned → Closed
Events: Pay, Cancel, Ship, Sign, Return
Transition Conditions: Payment success, inventory deduction success, WMS callback success
2.3 Inventory State Model
Available → Reserved → Deducted → Sold
Cancel: Reserved → Available
Return: Deducted → Available
2.4 External Interaction Model
Storefront ↔ WMS: shipment, delivery, return
Storefront ↔ SC2P: inventory sync, order sync
2.5 Consistency Boundaries
Strong consistency: inventory reserve/deduct
Eventual consistency: WMS/SC2P sync
Reconciliation points: hourly inventory check, daily order reconciliation
3. F – Functional
3.1 Core Happy Path
Create order successfully
Pay successfully
Ship and sync to WMS
Sign order
Normal close
3.2 Inventory Synchronization Scenarios
Reserve success / failure
Deduct success / failure
Cancel order and restore inventory
Idempotency for duplicate messages
Out-of-order message handling
3.3 Boundary Scenarios
Zero stock
High-concurrency flash sale
Partial shipment / split shipment
Return and inventory restoration
3.4 Critical API Validations
Required request fields
Correct error codes
Callback and writeback fields
Permission and auth validation
4. Q – Quality
4.1 Performance
Concurrency target: 1000+ TPS
Latency: < 200ms per order
Error rate < 0.1%
Throughput meets peak flash sale
4.2 Reliability
Retry mechanism for external calls
Fallback & timeout control
Circuit breaker
Automatic compensation jobs
4.3 Observability
Logs: order, inventory, payment
Metrics: inventory diff ratio, message backlog
Alerts: stuck orders, sync failure
4.4 Security
User authentication
Permission boundaries
Sensitive data masking (phone, address)
4.5 Recoverability
Failure drill process
Data repair and reconciliation
Manual compensation support
5. Release Entry & Exit Criteria
Entry Criteria
All P0/P1 functions developed
Unit testing passed
Test environment ready
Exit Criteria
No P0/P1 bugs
Inventory consistency 100% pass
Performance meets target
All compensation scenarios verified
6. Exception Handling & Compensation Strategy
Inventory inconsistency → automatic reconciliation job
Order stuck → timeout close + inventory restore
WMS timeout → retry + manual compensation
Duplicate message → idempotent check
