# NorthStar Retail — Enterprise Omnichannel Commerce Platform
## Phase 1: Business & Architecture Discovery

**Status:** Conceptual discovery completed (10/10 stages)  
**Document type:** Portfolio project discovery baseline  
**Next phase:** Phase 2 — Functional & Non-Functional Requirements

> This is a fictional enterprise and an architecture learning project. Business scale, baselines, targets, and proposed technology choices are planning assumptions, not validated operational facts or deployed infrastructure.

## 1. Executive Summary

NorthStar Retail is a fictional omnichannel retailer modernizing commerce across web, mobile, stores, pickup, shipping, and warehouses. The proposed platform must enable reliable inventory reservations, resilient checkout and order processing, event-driven integrations, clear domain ownership, and measurable business outcomes. The preferred migration strategy is incremental modernization with controlled coexistence of legacy and new services.

## 2. Business Context and Scale Assumptions

| Dimension | Planning assumption |
|---|---|
| Stores | 1,200 |
| Customers | 30 million |
| Annual orders | 100 million+ |
| Annual revenue | Approximately $15 billion |
| Channels | Website, mobile, POS, BOPIS, shipping |
| Existing landscape | Legacy POS, inventory, ERP, OMS, WMS and fulfillment integrations |

These assumptions will be reviewed and converted into explicit workload and capacity estimates in Phase 3.

## 3. Business Problems

1. Inconsistent stock visibility between stores, websites, and fulfillment systems.
2. Overselling caused by concurrent orders and stale availability data.
3. Partial checkout failures that leave inventory reserved or orders in ambiguous states.
4. Duplicate requests and repeated event deliveries that can trigger repeated business actions.
5. Fragile integration between database commits and event publication.
6. Slow operational exception investigation across disconnected systems.
7. Risk and disruption associated with replacing all legacy systems at once.

## 4. Phase 1 Learning and Architecture Decisions

| Stage | Scenario / question | Discovery decision |
|---|---|---|
| 1A | Three customers compete for two televisions | Use atomic conditional inventory reservations; do not rely on read-then-write stock checks. |
| 1B | Payment fails after stock reservation | Run a compensating action to release reserved inventory. |
| 1C | Inventory service is unavailable during compensation | Persist compensation work, retry with backoff/jitter, and reconcile or escalate unresolved failures. |
| 1D | Customer retries checkout after a timeout | Use an idempotency key with durable request processing and result state. |
| 1E | Duplicate `OrderConfirmed` event is delivered | Make consumers idempotent and protect downstream fulfillment operations with durable uniqueness controls. |
| 1F | Order is committed but event publication fails | Use a transactional outbox written atomically with the order and published asynchronously. |
| 1G | POS sale updates inventory while website cache is stale | Allow eventual consistency for browsing; check and atomically reserve against authoritative inventory at checkout. |
| 1H | Website, POS, orders, and warehouse all need stock information | Establish a dedicated Inventory domain as authoritative owner of inventory and reservation rules; expose APIs/events. |
| 1I | Leadership wants improved customer experience and fewer failures | Define measurable business KPIs, technical SLIs/SLOs, baselines, targets, and measurement rules. |
| 1J | CTO asks whether to replace every legacy system immediately | Modernize incrementally, integrate through controlled interfaces, measure results, and preserve rollback paths. |

## 5. Architecture Principles

### 5.1 Correctness at transaction boundaries
- Inventory reservation must use an atomic conditional operation or an equivalent concurrency-safe transaction.
- Checkout must not promise inventory solely from a stale cache.
- Payment requests and checkout requests need independently appropriate idempotency protection.
- The final order-confirmation policy must account for both successful reservation and payment state.

### 5.2 Reliable distributed workflows
- Use sagas or equivalent explicit workflow state for cross-service operations.
- Compensation is durable and retried; failed actions are not silently dropped.
- Event delivery should be assumed at-least-once unless proven otherwise.
- Consumers must tolerate duplicates and reordered events where applicable.
- Transactional outbox closes the local database-write/event-publication gap, but does not guarantee exactly-once downstream processing.

### 5.3 Clear data ownership
- Each bounded context defines its business model and authoritative writes.
- Services must not directly modify another service's private data store.
- Inventory ownership remains unambiguous during migration; legacy and new systems must not independently reserve the same stock partition.
- APIs and versioned events are the primary integration contracts.

### 5.4 Performance and availability
- Fast cached browsing can tolerate bounded staleness; authoritative checkout decisions cannot.
- Apply backpressure, timeouts, retries, dead-letter handling, and reconciliation based on failure modes.
- Optimize toward measured user outcomes rather than maximizing the number of AWS services.

## 6. Candidate Domain Boundaries

| Domain | Primary responsibility |
|---|---|
| Catalog | Product attributes, descriptions, media and merchandising data |
| Pricing | Price books, discounts and promotions |
| Inventory | Stock accounting, available-to-promise, reservations and adjustments |
| Cart / Checkout | Customer intent, validation and purchase initiation |
| Orders | Order lifecycle and customer-facing order state |
| Payments | Provider integration, authorization/capture/refunds and reconciliation |
| Fulfillment | Allocation, BOPIS, shipping and delivery workflow |
| Notifications | Customer communications and delivery status messaging |
| Analytics | Governed reporting and historical insights; not authoritative transactional writes |

A bounded context does not automatically imply a separately deployed microservice. Phase 4 will refine boundaries and deployment decisions.

## 7. Candidate Technology Direction (Not Finalized)

- **Cloud and IaC:** AWS and Terraform.
- **Application development:** Python / FastAPI.
- **Compute:** ECS and Lambda as initial candidates; evaluate EKS only where justified.
- **Databases:** Aurora PostgreSQL for relational transactional workloads; DynamoDB for access patterns suited to conditional writes; Redis/ElastiCache for appropriate caches.
- **Events and workflows:** EventBridge, SQS, DLQs, and possibly Step Functions; assess Kafka/MSK based on demonstrated needs.
- **Data and AI:** S3, Glue, Athena, SageMaker and Amazon Bedrock subject to cost, security, and governance assessment.

All service choices require workload, consistency, security, operational and cost evaluation before adoption.

## 8. Proposed Business KPIs and Technical SLOs

**Illustrative planning assumptions only.** These figures are not actual observed metrics.

| Measure | Assumed baseline | Proposed target | Type |
|---|---:|---:|---|
| Inventory accuracy | 94% | 99% | Business KPI |
| Inventory-related cancellation rate | 3% | Below 0.5% | Business KPI |
| Checkout success rate for eligible requests | 97% | 99.9% | Business / reliability KPI |
| Checkout API p95 latency | 1,500 ms | Below 500 ms | Technical SLI/SLO |
| On-time fulfillment | 91% | 98% | Business KPI |
| Order exception resolution time | 45 minutes | Below 15 minutes | Operational KPI |

**Phase 2 work:** Specify exact numerator/denominator, eligible traffic, observation window, percentiles, exclusions, telemetry sources, and acceptance criteria. Avoid treating a proposed target as a contractual guarantee.

## 9. Modernization Strategy

1. Identify high-value capabilities with clear ownership and measurable outcomes.
2. Introduce an API/integration layer to isolate channel consumers from legacy and modern services.
3. Modernize incrementally using the Strangler Fig Pattern where appropriate.
4. Ensure a single authoritative inventory writer per defined stock partition during transition.
5. Use event contracts, reconciliation, and controlled cutover procedures.
6. Validate operational readiness, rollback, and business metrics before expanding rollout.

## 10. Four Planned AI Use Cases

1. **Order Exception Investigation Agent:** Amazon Bedrock with governed tools, workflow integrations, and human approval boundaries.
2. **Demand Forecasting:** SageMaker/Python with data sourced through S3 and Glue.
3. **Shopping Assistant:** Bedrock-based assistant with retrieval and authoritative live catalog/pricing/inventory APIs.
4. **Retail Operations Intelligence Agent:** Bedrock with governed Athena/Glue/S3 analytics and permission enforcement.

AI is not permitted to bypass inventory reservation, payment authorization, or policy approvals. These use cases are planned for later phases, not implemented in Phase 1.

## 11. Risks and Open Questions

| Area | Open question / risk | Planned follow-up |
|---|---|---|
| Inventory | Exact stock ownership and POS/WMS synchronization semantics | Phases 2, 4, 13 |
| Checkout | Order confirmation policy under payment and reservation partial failures | Phases 2, 14, 16 |
| Capacity | Real peak traffic and burst multipliers are unknown | Phase 3 |
| Consistency | Maximum acceptable cache staleness and event lag | Phase 2 |
| Compliance | Specific regulatory, PCI and privacy boundaries | Phases 2, 6, 9 |
| Reliability | RTO, RPO, SLO windows and regional failure strategy | Phases 2, 19 |
| Cost | AWS service and data transfer cost ceilings | Phases 3, 6, 24 |
| AI | Data access permissions, evaluation and human oversight | Phases 2, 22, 23 |
| Migration | Legacy interface quality, cutover and rollback readiness | Phases 4, 5, 24 |

## 12. Phase 1 Exit Criteria

- [x] Core distributed commerce failure modes discussed.
- [x] Ten conceptual discovery stages and recommended decisions completed.
- [x] Candidate business domains and ownership principles identified.
- [x] Initial business assumptions and illustrative success metrics recorded.
- [x] Incremental modernization strategy selected.
- [ ] Phase 1 document committed and pushed to GitHub (complete after repository steps).
- [ ] Business stakeholder validation (not applicable yet; fictional project).

**Phase 2 readiness:** Ready to begin Functional & Non-Functional Requirements after the Phase 1 artifact is stored and versioned. Requirements, service boundaries, AWS choices, and metric targets remain provisional until validated in later phases.
