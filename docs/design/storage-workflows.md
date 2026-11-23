# Admission to initial protection

An admission is one accepted signed AddPieces request, optionally combined with dataset creation. This workflow covers upload, acceptance, client-facing inclusion and initial backend protection. The [storage model](storage-model.md#admission-persistence) defines records and consistency rules; [execution](execution-and-integrations.md) and [funding](funding-and-lifecycle.md) define external contracts and admission policy.

## Observable contract

- The client uses tus directly at an Upload SP. Parking confirmation, sufficient remaining retention, request validation, and required preflight precede provisional responsibility.
- The SDK waits on the returned transaction hash and succeeds at successful client-to-MSP AddPieces inclusion.
- Backend work begins after acceptance and parking confirmation; each placement completes at successful backend-SP AddPieces inclusion.
- Reuse compatible protection and in-flight work while retaining each ordered entry's membership and interest.
- Durable operations reconcile after lost responses; a timeout or unknown outcome does not withdraw accepted demand.

## Running scenario

One signed request adds P after it is parked at Upload SP A. If all three placements are needed, A, B and C are selected: A needs commitment, while B and C need transfers followed by commitment. Reused placements reduce that work.

```mermaid
sequenceDiagram
    participant C as Client / Synapse
    participant A as Upload SP A
    participant CS as Client service
    participant TM as Transaction management
    participant SM as Storage management
    participant SP as Backend SPs A, B, C
    participant Chain as Chain RPC
    C->>CS: Request upload preflight
    CS-->>C: MSP upload ID, tus creation URL, JWT grant
    C->>A: Create tus resource and PATCH bytes
    C->>A: Read parked result
    C->>CS: Signed client-facing AddPieces with upload ID
    CS->>A: Confirm retrieval and refresh parking if needed
    Note over CS: Validate request, funding context, admission eligibility
    Note over CS,TM: One local transaction: admission, transaction request, durable handoff
    CS->>SM: Durable provisional-interest delivery
    Note over SM: Apply interest and publish required placement work atomically
    par Client-facing commitment
        TM->>TM: Sign, submit, reconcile client-facing transaction
        TM-->>CS: Durable outcome
        C->>Chain: Wait for returned transaction hash
        alt Client-facing inclusion
            Chain-->>C: Successful receipt and assigned IDs
            Note over C: SDK storage success
            CS->>SM: Durable promotion of the same interest
        else Definitive admission failure
            CS->>SM: Durable withdrawal of this interest
        else Outcome still unknown
            Note over CS,TM: Retain operation and reconcile without withdrawing interest
        end
    and Backend protection
        SM->>SP: Authorize eager pulls with MSP JWTs
        Note over SP: Piece bytes move directly SP to SP
        SM->>SP: Submit fixed signed AddPieces batches when ready/due
        SP-->>SM: Outcomes, or recover through stable operation identity
        Note over SM: Apply exact placement outcomes and reconcile remaining demand
    end
```

Client-facing and backend operations proceed and reconcile independently.

## Step contracts

### 1. Upload routing and parking

The [upload-preflight response](execution-and-integrations.md#upload-preflight-and-destination-handoff) supplies the MSP upload ID and the selected Upload SP's tus Creation URL and grant. The SDK creates the resource there, uploads or resumes it, reads the parked result, and submits AddPieces to MSP with an upload ID for every ordered entry.

Before acceptance, confirm the source with [retrieval HEAD](execution-and-integrations.md#mandatory-confirmation-retrieval-head) and establish [sufficient remaining retention](execution-and-integrations.md#parking-refresh-and-retention), refreshing parking as needed. Neither the result lookup nor HEAD extends retention.

### 2. Preflight and admission decision

Identify the request and ordered entries under the [execution contract](execution-and-integrations.md#request-and-entry-identities). Validate authority and correlation, then recover an accepted identical request before new-admission checks. For a new request, confirm parking and perform required client preflight; [funding policy](funding-and-lifecycle.md#agreed-admission-policies) defines contract-only client eligibility and known backend/provider shortfalls.

Evaluate backend-funding and three-SP shortfalls using only already-available evidence. Unknown backend readiness is admissible after mandatory checks pass. A later shortfall leaves accepted work pending for fulfillment and replanning.

An accepted replay bypasses new-admission gates. A fresh nonce is a new admission even for identical PieceCIDs.

### 3. Accept provisional responsibility

In one short PostgreSQL transaction, Client service:

1. Records the validated request, its nonce claim(s), exact authorization/call, and stable ordered entry identities.
2. Enqueues the client-facing transaction through Transaction management's transaction-aware API.
3. Records the durable provisional-interest handoff for each entry to Storage management.

Each module writes its own records through an API using the shared local transaction. Commit before signing, RPC or provider calls.

Commit accepts the obligation. Cancellation or disconnection then stops the HTTP wait while durable transaction and fulfillment work continue.

### 4. Apply demand and plan backend work

Atomically apply delivery/revision bookkeeping, provisional interest, and discoverable reconciliation or placement work through Storage management. Under the piece coordination lock, read current interests, suitable placements, and outstanding work, then reuse protection before publishing only the deficit. The [model locking protocol](storage-model.md#planning-consistency-protocol) also applies to interest and outcome updates.

In the running scenario, A needs backend commitment while B and C need transfers and later commitments. This work may run concurrently with the client-facing transaction.

### 5. Run transfers and form commitments

Persist the transfer identity and exact request before issue. Authorize staging with the [MSP pull JWT](execution-and-integrations.md#provider-pulls) and recover the same operation after a lost response. Use each successful pull item's staging reference and deadline for commitment and refresh scheduling; a staged source can also supply witnesses without counting as a committed placement.

Batch independently ready placements by destination and urgency. Before freezing a batch, acquire affected piece locks in canonical order, recheck need and readiness, and atomically persist destination, ordered entries, exact payload, stable FWSS client nonce, and placement assignments. Follow the model's lock hierarchy for destination/operation records. Sign and submit after the transaction. Once authorization may have escaped, its payload and memberships remain immutable.

The SP submits backend AddPieces. Recover through its persisted authorization, correlated receipt and scoped `clientNonces(payer, nonce)` outcome under the [recovery contract](execution-and-integrations.md#correlating-success-and-recovering-an-unknown-outcome).

### 6. Apply client-facing outcomes

Client service correlates inclusion and membership assignments, updates continuing interests and requests Payments reconciliation through the [application protocol](#inclusion-acknowledgement-and-reorgs). Storage management promotes entries whose service remains requested.

The SDK succeeds when the client-facing transaction is included successfully. A definitive failure withdraws the affected interests; an orphaned failure restores retained service intent. The [inclusion/reorg contract](execution-and-integrations.md#addpieces-inclusion-and-reorgs) defines confirmation and the [response contract](execution-and-integrations.md#transaction-hash-response-contract) defines hash waiting and replay.

### 7. Apply backend outcomes and expose progress

Successful backend AddPieces inclusion completes the placement. Apply its exact ordered memberships and chain reference atomically, then make follow-on reconciliation due. After a reorg, retire invalidated mappings and reconcile placement/commit work, reusing parked data.

Protection status reads Storage management's views. Clients retrieve directly from SPs; [maintenance and retrieval design](README.md#4-complete-storage-maintenance-and-retrieval) covers location selection.

## Local transactions and module APIs

`LocalTx` methods use the caller's PostgreSQL connection and `READ COMMITTED` transaction. The caller commits; signing, RPC and HTTP run outside the transaction. Workers claim durable formats they support.

Allocate admission and transaction-request IDs before acceptance writes. This allows the admission's required transaction reference to be inserted before enqueue; verify the returned request ID before committing.

| API | Contract |
| --- | --- |
| `Transactions.ClaimFWSS(tx, owner, claims)` | Insert sorted [FWSS nonce claims](storage-model.md#acceptance-records) for the operation. Return the existing owner on collision; the caller compares the complete authorization. |
| `Transactions.Enqueue(tx, requestID, callerKey, exactCall)` | Persist or recover the request by caller module/operation. Verify request ID, sender, chain, target, value and calldata; return the retained request ID. |
| `Transactions.ReadOutcome(tx, requestID, mode)` | Acquire the shared reconciliation-basis row lock, then the request lock: exclusive for application, shared for observation. Read the current outcome and validate its saved branch token; return the outcome or `revalidation_required`. |
| `Transactions.AcknowledgeOutcome(tx, requestID, revision)` | Clear pending delivery conditionally for the applied current revision. Retirement leaves it pending. |
| `Storage.EnqueueInterest(tx, sourceSnapshot)` | Coalesce the highest revision per entry, comparing equal revisions and preserving initial source attribution. The piece-locked consumer applies it. |
| `Payments.RequestReconcile(tx, datasetKey, evidenceHint)` | Make the coalesced dataset key due for Payments to discover and refresh its account/rail observations. |
| `Client.ReadProgress(originalHash or admissionID)` | Recover the operation's current chain facts and ordered results through owner APIs. |

Client service locks the admission, then the branch/request and enqueues updates in entry order. Storage workers discover candidates, acquire ordered piece locks, then lock inbox and interest rows. Transaction-management workers use their branch/account/request lock path. Each path accesses other modules through their APIs.

### Acceptance and concurrent replay

1. Authenticate and correlate the request, then look up its logical key through FWSS nonce claims. Recover an equivalent accepted request before admission checks; compare the complete envelope, including any creation claim.
2. For a new request, run mandatory contract preflight, source correlation/HEAD, staging-retention and admission gates. Before returning a gate failure, recheck whether a concurrent attempt accepted the request.
3. In one transaction, insert admission/entries, claim add/create keys, enqueue the exact call and enqueue each entry's initial provisional state. Persist distinct source attribution and admission evidence.
4. On uniqueness conflict, roll back the complete transaction. Load the winning operation in a fresh transaction and compare its bound request. Recover an equivalent operation or return a conflict for different semantics/ownership.
5. After commit, wait for persisted signed bytes/hash and submission work, then return `201` with the original hash in `Location`. Workers discover accepted work independently of the HTTP connection.

Before commit, verify ordered indexes `0..N-1`, one entry/update per signed occurrence, the source binding for each entry, fresh HEAD and sufficient retention for each selected source, nonce roles/digests and the returned transaction ID.

### Interest application and placement planning

Apply interest updates under the piece lock: lock and reread the latest pending entry, apply a higher revision or discard a stale/equal update, retain withdrawn state/revision and delete the consumed pending row atomically. Make planning due with each applied change. Concurrent enqueue either updates the locked row first or inserts after deletion commits.

Freeze batches under the [ordered locks](storage-model.md#multi-piece-batches-and-lock-order). Recheck need, the destination dataset's current creation result and placement assignment, then persist payload, nonce claims and ordered entries atomically. Sign after commit and persist signed authorization, provider request and submission work before issue. Retry the saved request; alternative authorizations use new placements and may reuse staged data.

Create backend datasets under the destination lock, taking affected piece locks first. Persist creation intent and its nonce claim before signing; combined creation/addition claims both keys atomically. Only uncommitted internal candidates can change after a collision. Accepted or potentially issued authorizations retain their keys and payloads.

### Inclusion, acknowledgement, and reorgs

| Transition | Atomic local changes and recovery |
| --- | --- |
| Publish transaction observation | Capture operation revision, ownership and branch token before external reads. Lock and recheck at publication; replace the current snapshot, increment revision and set pending delivery atomically. Carry the established inclusion revision until its binding changes. |
| Retire a superseded binding | Validate that the current outcome supersedes the local inclusion. In its own transaction, clear those mappings/pointer, publish provisional interest where service is requested and request Payments reconciliation. Leave outcome application/delivery pending until the current outcome is fully applied. |
| Install current inclusion | Lock admission and validate the current outcome for application. Insert inclusion/mappings; derive interest from outcome and `service_requested`. Enqueue changed updates and Payments reconciliation, then advance/acknowledge together. Conflicting mappings leave installation pending after committed retirement. |
| Apply without installing | Apply a validated unresolved/failed outcome or update to the installed inclusion, then advance/acknowledge its revision. Defer `revalidation_required` until Transaction management publishes a validated outcome. |
| Correct a reorg or failure | Retire superseded inclusion and membership mappings first. Re-inclusion uses a new inclusion revision. A definitive failure withdraws published interest while preserving service intent; if orphaned, restore interest where service remains requested. Reconcile the original authorization under its retained client nonce. |

The [inclusion revision](storage-model.md#execution-inclusion-and-acknowledgement-records) identifies the successful inclusion being applied. It remains stable through evidence updates and changes after invalidation/restoration. Reconciliation continues after SDK success.

If a fork swaps A's and B's assigned IDs, retire each old mapping in a committed transaction before installing replacements. An installation conflict then leaves the old blocker retired. Backend membership application uses the same sequence under ordered piece locks.

Explicitly withdrawn entries remain withdrawn. Otherwise terminal failure withdraws interest, installed inclusion confirms it and validated unresolved/retired state is provisional. Increment the source revision when its snapshot changes; Payments receives reconciliation keys.

Resolve removals to entry/placement, inclusion and exact external target at the authorization/observation view. Persist this binding before handoff, revalidate before issue/retry and apply later results to the recorded relationship. Reconcile escaped effects against their actual chain context.

Invalidated backend creation makes its slot non-accepting. Validate the destination dataset's creation result under the dataset lock before freezing, and reopen after correlating its current on-chain ID. Resolve already-frozen requests against their recorded targets.

### Durable discovery

Bounded due scans cover unresolved effects and outcomes awaiting application, including resolved outcomes awaiting membership installation. Stale branch tokens select resolved success/revert/conflict for revalidation. Lifecycle/funding checks follow their freshness schedules; accepted work recovers independently of HTTP waiters.

The SDK follows [transaction inclusion](execution-and-integrations.md#sdk-transaction-waiting) independently of local outcome application.


## Interruption and race obligations

| Event | Required result |
| --- | --- |
| Known backend-funding or three-SP shortfall before acceptance | Retryable failure; no accepted demand or client transaction |
| Backend readiness unknown, after required validation and parking | Do not probe solely for readiness or reject solely for missing information |
| Required contract preflight, parking HEAD, or sufficient remaining retention cannot be established | Do not accept or issue the client transaction |
| Admission transaction commits, then response or handoff is lost | Recover admission, transaction request, and handoff durably |
| Interest deliveries reorder | Apply by source relationship and revision; old withdrawal cannot erase new interest |
| Transfer response is lost, or owner dies after issuing work | Reconcile the stable operation; retain unknown outcome and fence stale local writes |
| A parking refresh response is lost or the owner dies | Recover against the same staging reference and requested deadline; do not assume a renewal took effect |
| Batch formation is interrupted | Batch and placement assignments exist together or neither does |
| Backend outcome is missing or reorged | Resolve against the persisted authorization at an identified chain view; apply exact ordered memberships, or retain/reconcile unresolved work |
| Client confirmation precedes replication, or later definitively fails | Report actual progress separately; on failure withdraw only that interest and reconcile shared/issued work |
| Eligibility falls after acceptance | Preserve demand and replan; do not discard the obligation |
| One of repeated `[P, P]` entries or Alice's shared interest is removed | Withdraw only its entry/interest; retain protection needed by the other demand |
| A-old removal completes after A-new at the same SP | Apply only to A-old and reassess protection |

