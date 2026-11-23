# Design and execution principles

The [architecture](../architecture.md) describes MSP's responsibilities and workflows. These principles guide their design, implementation, and verification. Persistence here means application database access; SPs store the bulk data.

## Failure is normal

Design for ongoing partial failure. Dependencies can be unavailable, requests can time out after taking effect, and a process can stop between any two steps. Concurrent work, duplicate delivery, and delayed responses are normal; correctness must not depend on uninterrupted execution or exactly-once delivery.

## Failure must not create an unrecoverable state

Design recovery alongside execution:

- Persist the intent, identity, and progress needed to resume or reconcile unfinished work after instance loss.
- Treat an unknown external outcome as unresolved. Establish how to recover it or safely replay the same operation before relying on the integration.
- Define a recovery path for interruption before and after each durable write and external effect. Preserve domain invariants and distinguish a definitive domain failure from interrupted execution.
- Checkpoint when restarting would lose necessary facts, change an escaped effect's identity, or risk material cost or deadlines. Cheap HEAD and RPC reads, admission simulation, polling, and response assembly can restart. Retain attributed results needed by policy or replay, and proof witnesses when recollection risks the round deadline.

Recoverability avoids software-created dead ends and retains the information needed to recover. Physical loss of every copy can still destroy the data.

The architecture's [recoverable external operations](../architecture.md#recoverable-external-operations) and the storage model's [atomicity contracts](../design/storage-model.md#invariants-and-atomicity) apply this principle to MSP workflows.

## Routine recovery is automatic

After crashes, failover, temporary outages, or uncertain outcomes, automatically rediscover, reconcile, and resume unfinished work when dependencies and capacity return. Reserve operator intervention for exceptional judgment or remediation, and for deliberate business approvals such as client termination.

## Support rolling updates without stranding durable work

Rolling deployments must allow overlapping supported versions to share PostgreSQL and durable work safely, without switching every instance together.

Stage schema and durable-record changes so both versions can operate correctly during the transition. Do not let an instance claim work whose format or semantics it cannot safely handle. Existing operation identities and already-authorized payloads must retain their meaning across an upgrade.

Remove obsolete schema and compatibility paths once no supported running version or outstanding work needs them. Verify version overlap and recovery of pre-update work; compatibility does not extend to every historical format.

## Cancellation and shutdown

A disconnected API client, cancelled context, or expired request deadline may stop an execution attempt. Durably accepted work continues until a separate domain action withdraws it.

On shutdown, stop accepting requests and claiming work, then allow a bounded drain. Preserve unfinished work for recovery even if shutdown is interrupted. An issued external operation may still take effect after cancellation; retain its identity for outcome reconciliation.

## Make correctness contracts executable

Express internal correctness assumptions as executable contracts. When a contract fails, recovery must still account for earlier commits and external effects.

- Prefer types that prevent invalid combinations; assert meaningful preconditions, postconditions, and invariants that types cannot express.
- State whether a postcondition applies on successful return or every exit. An expected error must not trigger a postcondition that assumes success.
- Distinguish assertions from validation and operational error handling:

| Condition | Treatment |
| --- | --- |
| Malformed client input or an invalid provider witness | Validate and reject through the normal error path. |
| Provider timeout, lost lease, conflicting update, or transaction serialization failure | Handle as an expected execution condition; reconcile or retry as appropriate. |
| Our own batch-building code produces duplicate placement entries or a result inconsistent with its validated inputs | Internal contract violation; terminate the process by default. |

- Use [paired assertions](https://tigerbeetle.com/blog/2023-12-27-it-takes-two-to-contract/) where independently useful: producers check guarantees, consumers check assumptions. Durable write and recovery/read paths are valuable pairs; do not add redundant checks to meet a quota.
- Keep safety-critical contract checks enabled in production. Checks must be side-effect-free, bounded, and precise enough to identify the violated invariant; do not add network calls or database scans solely to assert a condition.
- Revalidate mutable assumptions through the concurrency protocol after I/O, transaction boundaries and ownership changes. Treat changed or contended plans as normal execution conditions and recalculate from current facts.

### Assertion failures terminate the process by default

An internal assertion failure must terminate the process by default, even inside an HTTP handler or worker loop. Multiple active instances and automatic restart restore capacity; durable intent and reconciliation recover interrupted work without relying on instance cleanup.

A deterministic defect may cause successive instances to hit the same assertion. Fix the code while preserving durable work.

In Go, the standard [HTTP server recovers handler panics](https://github.com/golang/go/blob/master/src/net/http/server.go). The assertion-failure path must deliberately terminate the process even across request or worker recovery boundaries.

## Separate and test domain logic, persistence, and complex algorithms

Keep domain logic, persistence, and nontrivial algorithms separable from request handlers and worker loops:

- **Domain logic:** test consumer-visible decisions and state transitions with explicit inputs.
- **Persistence:** exercise SQL constraints, atomicity and concurrency directly against PostgreSQL.
- **Complex algorithms:** expose scheduling, selection, batching, or other nontrivial calculations so their behavior can be tested independently of orchestration. Cover boundaries and invariants, not just a normal example.
- **Execution and recovery:** complement component tests with scenarios that interrupt operations, repeat delivery, reorder outcomes, and overlap workers. Verify that work is rediscovered and resumed without operator action after restart or dependency recovery. Separately correct components do not prove that their coordination is correct.

Use the smallest testable boundaries that clarify ownership and effects.

## Keep execution bounded and predictable

- **Bound work and resource use:** bound request/response bodies, batch sizes, queue admission, concurrency, and per-attempt deadlines. An indefinite worker loop must have bounded iterations.
- **Keep calculations deterministic where practical:** make time, randomness, and external observations explicit inputs to planning and other algorithms. This supports reproducible property/fuzz tests and failure/interleaving scenarios without requiring a simulation framework for the entire application.
- **Make identities and units hard to confuse:** use scoped Go types for client/backend datasets, placements, operations, epochs/durations, and leaf indexes/byte offsets.
- **Sketch costs before fixing the design:** estimate SQL round trips, network requests, memory, and critical-path latency for representative workloads. Preserve useful batching while respecting urgency; use measurements to refine the estimates once an implementation exists.

### Apply backpressure and preserve urgency

- When capacity is exhausted, explicitly reject or defer new admission.
- Keep accepted work durable while full queues or exhausted attempt budgets defer execution.
- Prevent new admissions and bulk background work from starving deadline-critical proving or urgent repair. Concrete capacities and scheduling weights can be selected during implementation.

## Make trust and authority boundaries explicit

Authenticate and authorize client and operator requests. Keep intentionally public surfaces public.

Treat provider and chain responses as external evidence. Validate content and correlate it with the expected request, resource, and chain context before making domain decisions. Invalid input follows the normal error path, not assertion failure.

Minimize credentials and permissions granted to each role. Do not expose private keys, authentication tokens, or other secrets in logs, traces, or errors.

Use the signer for key custody and signing audit, with operation authorization in the domain modules.

## Explicit SQL; no ORM

Do not use an ORM. Keep SQL and transaction boundaries explicit and reviewable, especially where correctness depends on constraints, concurrent updates, or atomic changes across records.

Typed SQL code generation versus handwritten parameter/result handling remains undecided. Choose when representative queries and transactions clarify the trade-offs.

## References and scope

[TigerStyle](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md) and TigerBeetle's [paired-contract discussion](https://tigerbeetle.com/blog/2023-12-27-it-takes-two-to-contract/) informed these principles. Follow Go conventions and select tools against MSP's needs.
