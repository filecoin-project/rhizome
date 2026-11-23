# Rhizome design

**Status: Work in progress.** The architecture records agreed direction. The contracts below state the current design, including requirements for upstream integrations; they are not a claim that those integrations are implemented or verified.

[Architecture](../architecture.md) defines system boundaries and ownership; [engineering principles](../engineering/principles.md) guide implementation. This plan tracks the remaining schema, integration, deployment and operational decisions.

## Design documents

Keep each contract in one canonical document and link to it from consumers. Add planned documents as their design work starts.

| Document | Coverage and remaining work |
| --- | --- |
| [Storage model](storage-model.md) | Identities, protection accounting, invariants, locking and admission records/constraints. |
| [Storage workflows](storage-workflows.md) | Admission through initial protection, transaction-aware module APIs and recovery boundaries. Extend through repair, removal, and retrieval. |
| [Execution and integrations](execution-and-integrations.md) | Signed request/replay identity, transaction-hash response, upload preflight, and source confirmation. Complete external contracts and transaction/chain/signing integration. |
| [Funding and lifecycle](funding-and-lifecycle.md) | Funding scope and admission policy. Complete settlement, grace/holds, termination, and run-off. |
| `docs/design/proving-and-assessment.md` — planned | Challenges, witness verification/persistence, round deadlines, takeover, and provider assessment. |
| `docs/design/operations-and-verification.md` — planned | Workload/timing budgets, capacity, outages, rolling updates, observability, and verification. |

The standalone [direct tus upload proposal](../proposals/tus-upload-proposal.md) is intended for sharing. Use the [execution and integration contract](execution-and-integrations.md#upload-preflight-and-destination-handoff) for the current system design.

## Remaining work

### 1. Establish workload and timing budgets

Record representative piece sizes, managed pieces/placements, dataset counts, upload rate, repair bursts, and concurrently due proof rounds. Budget sole-copy exposure, detection, repair, proof collection/failover/submission, observation freshness, and outage catch-up.

Separate product commitments, protocol deadlines, configurable policy and estimates. Include correlated SP failure, staging loss, process/database/RPC interruption, unknown outcomes and reorgs. Derive durability commitments from these assumptions and targets.

**Review gate:** observable success criteria and unvalidated numerical assumptions are explicit.

### 2. Close external contracts

- **Curio witnesses:** close the architecture's Q1 with the Curio team. Verify PieceCID-based witnesses for staged and committed data, including authorization, limits, and a first applicable proof before backend commitment.
- **Synapse:** integrate tus Creation, offset resume, parked-result recovery and per-entry upload IDs; verify terms-aware creation signing and signed-request replay while preserving [transaction-hash waiting](execution-and-integrations.md#sdk-transaction-waiting).
- **Curio upload, staging and pulls:** implement tus JWT grants and stable resource/results, parking refresh coordinated with GC, and pull authorization and replay using MSP JWTs. Verify retention against execution/recovery budgets and prove partial pull outcomes and retry behavior.
- **Curio backend AddPieces:** establish recovery of provider-local tracking if submission escapes before membership tracking is durable.
- **FWSS / Filecoin Pay / chain access:** pin deployment versions, complete nonce/outcome correlation and state reads, and choose confidence for actions beyond AddPieces.
- **Signer:** select transaction/typed-data signing, custody/audit, Filecoin support and failover tooling.

For each integration decision, record the counterpart, affected workflow, exact question and closure evidence.

**Review gate:** every initial workflow has a supported execution and recovery path; unavailable capabilities remain explicit dependencies.

### 3. Make the admission slice implementable

The [admission records](storage-model.md#admission-persistence) and [transaction/API contracts](storage-workflows.md#local-transactions-and-module-apis) cover acceptance, FWSS nonce claims, ordered entries, inboxes, inclusion/acknowledgement and initial placements/batches. Preserve authority, preconditions, consistency scope, atomic writes, replay identity, outcome application and caller-visible result when converting them into migrations and Go APIs.

Cover either ordering of client/backend commitments, definitive failure, lost responses, duplicate delivery, cancellation and owner loss. Exercise lost tus Creation and final PATCH responses, lost parking refresh responses and refresh/GC races, pull credential renewal, and a first proof using staged data. Follow the defined identity and locking contracts, with database-only transactions.

**Review gate:** another engineer can implement the slice and recover it at every boundary; uncertain integration behavior has executable evidence.

### 4. Complete storage maintenance and retrieval

Specify ordinary repair, provider/dataset-wide replacement, bounded overlapping attempts, urgency-sensitive batching, asynchronous removal, and location selection for staged/suspect/outgoing data. Show competing planning/removal transactions, frozen-batch recovery, and late-result application against current demand.

**Review gate:** concurrent planners, remove/re-add with reversed outcomes, and takeover preserve the model's invariants. Representative cost estimates show urgent work is not starved.

### 5. Complete proving, assessment, and financial lifecycle

Budget a full proof round including collection, fallback, takeover, and submission. Persist accepted witnesses, retain the original deadline, and keep failed-provider observations even after successful fallback.

Specify payer/asset-scoped settlement and funding monitoring, adjustable grace/holds, operator-approved termination with rechecks, and initial run-off behavior. Keep settlement intent separate from actual progress, and payment shortfall separate from provider failure/deletion.

**Review gate:** near-deadline owner loss, partial settlement, and funding recovery during pending termination have unambiguous outcomes.

### 6. Review operation and rollout

Specify backpressure, proving/urgent-repair priority, durable-work discovery, scoped claims/fencing, bounded shutdown, dependency outages, and current-state recovery after complete MSP loss. Define mixed-version/schema transitions that preserve outstanding identities and authorized payloads.

**Review gate:** accepted work remains discoverable after restart, routine recovery needs no operator replay/database edits, and every core invariant has a verification scenario.

## Implementation-review gate

The initial design is ready when ownership, schemas, transactions, interfaces, recovery paths, timing/capacity assumptions and verification scenarios are concrete. Catalogue publication, packing, PoRep and repair-in-place follow the initial release.
