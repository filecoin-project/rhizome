# Client-facing proving and witness assessment

This is Rhizome's PostgreSQL and execution design for client-facing PDP. It is not a claim that the target chain deployment or Curio implements the required integration. MSP proves its client-facing datasets; backend SPs prove their own datasets independently and supply piece witnesses to MSP. This document covers assessment of those requests, not general provider monitoring policy.

## Ownership and inputs

| Owner | Contract |
| --- | --- |
| Proving | One leased worker per client-facing dataset. Owns schedule reconciliation, immutable rounds, verified witness progress, and the intent to activate, prove, process scheduled deletions, or advance. There is no independent worker queue for SP requests or proof slots. |
| Chain access | Provides a coherent chain view and its branch token, canonicality, and historical reads. A read of related verifier/listener state must not mix independently resolved `latest` calls. |
| Client service | Makes managed datasets and lifecycle changes discoverable to Proving. Owns membership bindings and authorizes scheduling client removals. Applies actual processed deletions using the [deleting-interest contract](storage-model.md#client-piece-deletion). |
| Storage management | Resolves the exact challenged membership to its storage scope and serves current authorized witness sources, including staged sources. Owns retention, protection, and alternate placements. Missing or stale membership is a retryable lookup failure; never infer access from PieceCID alone. |
| Transaction management | Owns nonce allocation, exact signed transactions, broadcast, replacement, and revisable outcomes. Uses the [local transaction APIs](storage-workflows.md#local-transactions-and-module-apis). Treat proof submission and period transitions as deadline-sensitive on the initial one MSP sender per chain. Prioritize before nonce allocation and bound the outstanding ordinary nonce prefix. |
| Provider assessment | Consumes attributed witness failure observations idempotently by attempt ID. Proving can fall back without waiting for assessment. |

## Chain schedule and round identity

Use receipt-first reconciliation, with current-state reads when a receipt is missing, revised, or insufficient. Read dataset liveness, sender and listener, current challenge epoch and range, queued removals, and FWSS activation, deadline, `provenThisPeriod`, configuration, and lifecycle end from one view. Relevant source: [PDP getters](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/PDPVerifier.sol#L375-L437), [FWSS schedule views](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/modules/FWSSViewModule.sol#L259-L335), and [FWSS proof validation](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/modules/FWSSProvingModule.sol#L53-L88). In particular, the verifier's last-proven epoch is also initialized by `nextProvingPeriod`; it is not evidence that this FWSS period was proven.

The effective submission window begins at the later of the scheduled challenge epoch and the FWSS window start. It ends at the earlier of the FWSS proving deadline and any applicable PDP lifecycle end, inclusive in the reviewed contracts ([PDP](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/PDPVerifier.sol#L1116-L1124), [FWSS lifecycle guard](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/lib/LibServiceLifecycleGuards.sol#L9-L15)). Use epochs for chain decisions. Wall-clock projections wake workers and budget latency; they do not establish that a round has missed or that a null epoch has passed.

Start collection when the seed and mapping are available from a coherent chain view. No fixed confirmation wait after the challenge epoch: the verifier uses [epoch beacon randomness](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/PDPVerifier.sol#L1234-L1274); modern [Lotus derives it from the signed drand round for that epoch](https://github.com/filecoin-project/lotus/blob/b3be770e0e236f50cc713bbe36c96f08ec3fe779/chain/rand/rand.go#L145-L213). A reorg may leave the seed unchanged while changing the schedule or target mapping. If the view cannot supply that beacon entry yet, wait for a usable chain view, not an arbitrary depth.

Read `challengeRange`, not total dataset leaves. For each required proof index, compute [the verifier's challenge](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/PDPVerifier.sol#L1128-L1151): Keccak-256 of the packed seed, dataset ID, and index (32, 32, and 8 bytes respectively), reduced modulo the nonzero range. Resolve indices with [batched `findPieceIds`](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/PDPVerifier.sol#L1423-L1436); fetch the expected PieceCID for each distinct piece ID. Read the proof count from the deployed listener configuration, not a compiled-in count. Resolve and verify the matching local membership and authorization scope before publishing a complete plan. On missing bindings, wake on membership application and retry by deadline; do not ask an SP about an unbound target.

Hash a versioned, unambiguous encoding of the dataset key, sender, listener and supported verifier/listener profile, FWSS period binding (activation and deadline), effective window, challenge epoch, seed, challenge range, and ordered `(verifier_piece_id, piece_cid, leaf_index, authorization_scope)` targets into `context_key`. Use fixed-width integers and addresses and length-prefixed variable fields. Persist the observation reference separately; do not hash the head block, freshness, wallet balance, or provider choice into the identity. Ordinary head advance must not create a new round. A complete plan is immutable and may be selected again after revalidation. A changed context selects a different round; it does not mutate old slots or implicitly copy witnesses across rounds. Takeover of the same context retains already verified needs.

## PostgreSQL records

Illustrative migration, using the [shared SQL domains](storage-model.md#acceptance-records). UUIDs are allocated before insertion. FKs are internal to Proving; opaque IDs across modules use owner APIs. The versions are durable record formats, not binaries or branch identities. Both versions of a rolling deployment must continue to claim and recover outstanding formats they support.

```sql
CREATE SCHEMA proving;

CREATE TABLE proving.datasets (
    id uuid PRIMARY KEY,
    chain_key text NOT NULL,
    verifier evm_address NOT NULL,
    onchain_dataset_id uint256 NOT NULL,
    work_version integer NOT NULL CHECK (work_version > 0),
    due_at timestamptz,
    lease_until timestamptz,
    generation bigint NOT NULL DEFAULT 0 CHECK (generation >= 0),
    wake_revision bigint NOT NULL DEFAULT 0 CHECK (wake_revision >= 0),
    checked_branch text,
    active_round_id uuid,
    claim_at timestamptz GENERATED ALWAYS AS (
        CASE WHEN due_at IS NOT NULL THEN greatest(due_at, lease_until) END
    ) STORED,
    UNIQUE (chain_key, verifier, onchain_dataset_id),
    CHECK (lease_until IS NULL OR due_at IS NOT NULL)
);
CREATE INDEX proving_due ON proving.datasets (work_version, claim_at, id)
    WHERE claim_at IS NOT NULL;
CREATE INDEX proving_by_chain ON proving.datasets (chain_key, id);

CREATE TABLE proving.rounds (
    id uuid PRIMARY KEY,
    dataset_id uuid NOT NULL REFERENCES proving.datasets(id),
    context_key evm_hash NOT NULL,
    format_version integer NOT NULL CHECK (format_version > 0),
    profile text NOT NULL,
    origin_view bytea NOT NULL,
    sender evm_address NOT NULL,
    listener evm_address NOT NULL,
    activation_epoch bigint NOT NULL CHECK (activation_epoch >= 0),
    challenge_epoch bigint NOT NULL CHECK (challenge_epoch > 0),
    submit_from_epoch bigint NOT NULL,
    period_deadline_epoch bigint NOT NULL,
    deadline_epoch bigint NOT NULL,
    seed evm_hash NOT NULL,
    challenge_range uint256 NOT NULL CHECK (challenge_range > 0),
    slot_count integer NOT NULL CHECK (slot_count > 0),
    UNIQUE (dataset_id, context_key),
    UNIQUE (dataset_id, id),
    CHECK (challenge_epoch <= submit_from_epoch AND submit_from_epoch <= deadline_epoch
        AND deadline_epoch <= period_deadline_epoch)
);
ALTER TABLE proving.datasets ADD FOREIGN KEY (id, active_round_id)
    REFERENCES proving.rounds (dataset_id, id);

CREATE TABLE proving.witness_needs (
    round_id uuid NOT NULL REFERENCES proving.rounds(id),
    need_index integer NOT NULL CHECK (need_index >= 0),
    authorization_scope uuid NOT NULL,
    piece_cid bytea NOT NULL CHECK (octet_length(piece_cid) > 0),
    leaf_index bigint NOT NULL CHECK (leaf_index >= 0),
    accepted_sp uuid,
    leaf bytea CHECK (octet_length(leaf) = 32),
    siblings bytea CHECK (octet_length(siblings) % 32 = 0),
    PRIMARY KEY (round_id, need_index),
    UNIQUE (round_id, authorization_scope, piece_cid, leaf_index),
    CHECK ((accepted_sp IS NULL AND leaf IS NULL AND siblings IS NULL)
        OR (accepted_sp IS NOT NULL AND leaf IS NOT NULL AND siblings IS NOT NULL))
);
CREATE TABLE proving.round_slots (
    round_id uuid NOT NULL,
    challenge_index integer NOT NULL CHECK (challenge_index >= 0),
    verifier_piece_id uint256 NOT NULL,
    need_index integer NOT NULL,
    PRIMARY KEY (round_id, challenge_index),
    FOREIGN KEY (round_id, need_index) REFERENCES proving.witness_needs (round_id, need_index)
);

CREATE TABLE proving.actions (
    id uuid PRIMARY KEY,
    dataset_id uuid NOT NULL REFERENCES proving.datasets(id),
    round_id uuid,
    kind text NOT NULL CHECK (kind IN ('activate', 'prove', 'process_deletions', 'advance')),
    action_key evm_hash NOT NULL,
    attempt_no integer NOT NULL CHECK (attempt_no > 0),
    intent_version integer NOT NULL CHECK (intent_version > 0),
    expected_state bytea NOT NULL CHECK (octet_length(expected_state) > 0),
    tx_request_id uuid NOT NULL UNIQUE,
    applied_outcome_revision bigint NOT NULL DEFAULT 0 CHECK (applied_outcome_revision >= 0),
    resolved_at timestamptz,
    UNIQUE (dataset_id, action_key, attempt_no),
    FOREIGN KEY (dataset_id, round_id) REFERENCES proving.rounds (dataset_id, id),
    CHECK ((kind = 'prove') = (round_id IS NOT NULL)),
    CHECK (resolved_at IS NULL OR applied_outcome_revision > 0)
);
CREATE UNIQUE INDEX proving_one_pending_action ON proving.actions (dataset_id)
    WHERE resolved_at IS NULL;

CREATE TABLE proving.witness_failures (
    attempt_id uuid PRIMARY KEY,
    round_id uuid NOT NULL REFERENCES proving.rounds(id),
    sp_id uuid NOT NULL,
    authorization_scope uuid NOT NULL,
    piece_cid bytea NOT NULL CHECK (octet_length(piece_cid) > 0),
    failure_code text NOT NULL,
    observed_at timestamptz NOT NULL
);
CREATE INDEX proving_failures_due ON proving.witness_failures (observed_at, attempt_id);
```

`origin_view` identifies the planning view in Chain access's encoding; `checked_branch` is the opaque token for the schedule last reconciled. Neither makes a stored round chain-current. A round, its ordered slots and deduplicated needs, and the dataset's active pointer are installed in one transaction. SQL enforces uniqueness and same-round references; the writer checks complete contiguous slots `0..slot_count-1`, correct bindings, and needs referenced by at least one slot. The reader checks completeness before assembly. Siblings are concatenated 32-byte nodes in verifier order. Domain validation checks exact depth, position, and root; SQL byte widths are not witness verification. A need's first accepted witness is immutable and one accepted need can fill several distinct ordered slots.

`action_key` is a versioned digest of the semantic action: kind, dataset, and expected chain boundary (proof context, activation or advancement period boundary, or deletion phase and observed queue suffix for a drain). `expected_state` retains those preconditions and intended effect in a typed encoding; Transaction management retains the exact call. A fresh attempt for the same purpose after a definitive, reconciled outcome increments `attempt_no` under the dataset lock; it is not a fee replacement. The pending-action index guards the local handoff, not chain canonicality. Revised outcomes for resolved actions can block further issue until reconciled. Retain rounds, signed-call identities, and outcomes while required for reorg or execution recovery. Delete old witness bytes only when no recoverable round or retained execution needs them; keep credentials, witness leaves, and branches out of logs and assessment events.

## Claiming, waking, and fencing

Use short `READ COMMITTED` transactions. `due_at` is the next local reconciliation time; `NULL` suspends ordinary work when no near-term follow-up is due. A live lease moves `claim_at` forward; no sweeper must clear it before takeover. Claim only when a worker slot is available. An instance may claim a bounded number by repeating the following query; the index gives due order within a supported work format. `SKIP LOCKED` is not a strict fairness or deadline guarantee.

```sql
-- claim: $1 supported work format, $2 lease interval
WITH candidate AS (
    SELECT id FROM proving.datasets
    WHERE work_version = $1 AND claim_at <= statement_timestamp()
    ORDER BY claim_at, id
    LIMIT 1 FOR UPDATE SKIP LOCKED
)
UPDATE proving.datasets AS d
SET generation = d.generation + 1,
    lease_until = clock_timestamp() + $2::interval
FROM candidate
WHERE d.id = candidate.id
RETURNING d.id, d.generation, d.wake_revision, d.active_round_id, d.lease_until;
```

`Proving.MakeDue(tx, datasetKey, due)` locks the dataset row, increments `wake_revision`, and moves `due_at` to `least(due_at, due)` (`least` ignores nulls in PostgreSQL). Only a managed-dataset handoff from Client service may create a row; later hints never invent a dataset. Wakes cover applied membership and lifecycle changes, relevant transaction outcomes, and changed chain branches. Use durable scans even when no notification arrives: Transaction management retains unacknowledged revisions, and Proving walks rows on the affected chain in bounded keyset batches, including dormant datasets with revisable outcomes. If a row's checked token differs from Chain access's current token, wake it rather than amending the token without reconciliation. Restart the sweep after outage or token change. Periodic reconciliation of managed datasets backs up hints; never enumerate every dataset on every head change. A normal head advance wakes only indexed due work. Inactive/terminated datasets must not be reactivated solely because a stale wake arrived.

Maintain a heartbeat independent of provider or chain I/O. For renewal, witness acceptance, action enqueue, and completion, lock the dataset row **before** comparing the live generation and `lease_until` to `clock_timestamp()`. For chain-dependent publication acquire the shared [reconciliation-basis](storage-model.md#execution-inclusion-and-acknowledgement-records) lock first and compare the captured branch; take dataset, action, then Transaction management request locks in that order. The dataset row lock serializes the protected write after validation; keep those transactions short. When accepting witnesses also require the captured active round; when enqueueing a proof require the current plan and a current chain decision. If any fence fails, roll back and stop using that claim or refresh the stale input. Provider observations for a batch can be queued in the same fenced transaction as its progress. On ownership loss do not report outstanding requests as failures; responses that did not reach a committed checkpoint can be recollected.

```sql
-- Complete after SELECT ... FOR UPDATE on this dataset in the same transaction.
-- $1 dataset, $2 claimed generation, $3 wake revision seen by the reconciliation,
-- $4 next due time (NULL if none)
UPDATE proving.datasets
SET due_at = CASE WHEN wake_revision = $3 THEN $4::timestamptz
                  ELSE least(due_at, $4::timestamptz) END,
    lease_until = NULL
WHERE id = $1 AND generation = $2 AND lease_until > clock_timestamp()
RETURNING id;
```

For completion, use the wake revision read before the inputs on which the decision depends. A producer that wins the row lock first leaves a changed revision and its wake survives. A producer that arrives later makes the released row due again. When draining, stop claiming and release a claim when safe; process death is recovered by ordinary expiry. No RPC, signing, or provider I/O holds a database lock.

## Reconciliation and period advancement

```mermaid
flowchart TD
    R[Reconcile pinned chain view and outstanding effects]
    R -->|Nonempty and no active period| I[Activate or reactivate]
    R -->|Challenge eligible and unproven| P[Collect and submit proof]
    R -->|Proven, expired, or no challenge to preserve| D[Drain scheduled deletions]
    R -->|Not yet eligible or no obligation| W[Persist next wake or suspend]
    D -->|Queue empty and transition needed| A[Advance or close empty period]
    D -->|Batch included or queue changed| R
    D -->|Queue empty and no transition needed| W
    I --> R
    P --> R
    A --> R
```

An outstanding incompatible action takes precedence over a new action in all branches. Resume from on-chain state, not from a local phase counter. On restart with a scheduled unproven round and enough time, collect for that round even if MSP has no local period-transition receipt. If the current challenge is not yet available, schedule a wake without holding a lease across epochs. Report an MSP deadline miss separately from provider evidence and reconcile the next legal period rather than generating work for historical ones; on-chain fault accounting remains authoritative. A live verifier dataset with no challenge but an active listener deadline may be between deletion batches and advancement. Inspect the queue, listener status and proof disposition before deciding. Do not schedule new proof actions past lifecycle end; use Client service's active obligation/run-off decision, not funding heuristics in Proving.

- **Activate / advance:** `nextProvingPeriod` updates the challenge range and must follow queued deletion processing. Use the [FWSS scheduling rules](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/modules/FWSSProvingModule.sol#L98-L192) and [verifier requirements](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/PDPVerifier.sol#L1254-L1288). On first activation the window is relative to the transaction's inclusion epoch, not its planning epoch. Reactivation is aligned to the existing activation epoch. For normal advancement after proof or expiry, follow the listener's current schedule and skipped-period rules. Choose an early challenge epoch legal across the budgeted inclusion epochs: intersect the allowed listener windows and verifier delay bounds, and leave time after the challenge for proof inclusion. If that intersection is empty or the expected inclusion range changes, do not keep sending the old call. Replan from fresh state after resolving any issued request. Empty inactive datasets need no activation. When the last piece is removed from an active dataset, finish the closing transition when legal; the verifier still requires a legal input epoch even when it ends up clearing the scheduled challenge.
- **Process deletions:** Only after a proof is canonically included, or the active challenge is no longer provable and incompatible submissions are reconciled. A deletion batch clears the challenge. Once the first batch is included, continue draining and advancing on the same safe phase even though `nextChallengeEpoch` is now zero; do not infer a new uninitialized dataset. Select a bounded, gas-fit positive count from `getScheduledRemovals`. [`processPieceDeletions` drains the current tail by count](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/PDPVerifier.sol#L1044-L1070), not a list selected by MSP. Recheck the tail before issue. A scheduling transaction can still change the tail before inclusion; correlate and apply only the exact `PiecesRemoved` IDs from the successful receipt. A different valid tail is a result to reconcile, not evidence that the planned IDs were removed. Refresh the queue before dispatching another batch or advancing; a newly queued removal can make an advance revert. Ordinary on-chain advancement does not wait for the [irreversible-action confidence gate](execution-and-integrations.md#chain-confidence-for-irreversible-actions) used for release of backend copies.

### Transaction handoff and recovery

1. For a ready proof assemble ordered slots and derive calldata, fee/value and sender from the applicable verifier and fresh chain evidence. Before enqueue, recheck live ownership, branch, outstanding actions and canonical context. In one transaction insert the action and call `Transactions.Enqueue` with the exact chain, sender, target, value and calldata. These writes commit together; a crash cannot leave an apparently ready action with no transaction request. If a concurrent effect already satisfies the purpose, reconcile it instead of submitting another call.
2. Before first broadcast, rebroadcast or fee replacement, Transaction management obtains a fresh Proving decision against this exact action: valid, temporarily undecidable, or obsolete. The check performs chain I/O outside locks. It verifies the current period and proof status, ordered targets, effective window and remaining inclusion time for a proof; for a transition it checks the phase, queue and expected inclusion window. A missing RPC result blocks issue/reissue, not receipt tracking. Revalidate the current action and ownership of the transport assignment at the issue boundary; a later fork can still race broadcast. Proving does not expose an endpoint that updates chain state behind Transaction management.
3. If an obsolete action never escaped, stop it and resolve its reservation through Transaction management. Do not abandon a lower nonce if later assignments depend on consuming it. If the action may have escaped, stop rebroadcasting its obsolete payload and resolve the assigned nonce by canonical execution, revert, or a confirmed replacement that consumes the same nonce without the obsolete effect. A timeout, cancelled RPC or local lease loss does not free an escaped nonce. Transaction management publishes a revised outcome even for an action stopped before broadcast. Do not dispatch an incompatible transition while the effect is unresolved: verifier entry points do not bind every call to its expected round. Already-issued later nonces cannot be recalled; after a fork reconcile earlier nonce effects and the actual canonical dataset state before granting further issue.
4. Apply the current branch-validated outcome and acknowledge its revision atomically. For deletion batches, enqueue a durable handoff through Client service's API in that same transaction, keyed by request ID and revision with receipt reference and exact verifier dataset/piece IDs. Client service resolves the historical membership binding, revises or retracts invalidated evidence, and owns confidence/release scanning; Proving must not clear storage interests. This enqueue takes no Client admission locks. Transaction management exposes later outcome revisions for resolved actions as well as unresolved ones, until acknowledged. On a fork, wake and reconcile affected actions, even if the pending-action slot has since moved on. Preserve the original intent and reconcile the earliest unresolved sender effect; never mark an old proof valid merely because the same nonce or seed reappeared.

## Witness collection and assessment

For missing needs, resolve authorized sources from Storage management and order candidates using readiness, retention, and provider evidence. Include the Upload SP's protected staged source when it is the only usable copy. A staged witness can satisfy a challenge, not the [committed-copy requirement during deletion](storage-model.md#client-piece-deletion). Do not treat an existing unopened or outgoing placement as evidence of continuing coverage.

Batch within the round by SP, authorization scope, and PieceCID, with distinct leaf indices. Use the [Curio witness HTTP contract](../proposals/curio-inclusion-proof-proposal.md) for credentials, limits, verification and errors. Do not persist bearer credentials or share results across authorization scopes. Apply the response to the exact request. Validate the full response's shape and every branch before accepting a batch; a malformed/invalid batch yields no accepted witnesses. Publish accepted needs in short fenced transactions as batches finish. Assembly is by challenge index, not SP response order. When refreshing or taking over, only re-request missing needs from still usable routes; no SP-side work ID or callback is required for this read-only API.

Use bounded worker capacity reserved for proving, bounded request/body/verification work, and process-wide limits for requests overall and per SP, not one unbounded goroutine per need. Size per-process limits for the deployed replica count; SP quotas remain authoritative across instances, including when an old owner's HTTP requests linger after failover. Take the earliest proof cutoff first among locally runnable requests. Reserve collection failover time and a separate chain submission reserve for preflight, registry/chain lookups, signing, the existing nonce prefix, replacement, and inclusion. Stop new collection attempts at the proof's collection cutoff. A request must leave room for a healthy fallback if one exists; prefer fallback on timeout or explicit failure and allow a bounded hedge only when waiting would consume that room. Cancel redundant requests after success. If no candidate is ready, promptly refresh source evidence and schedule a bounded retry before cutoff. No automatic cutoff or retry moves the on-chain deadline.

Each attempted request has a transient attempt ID. Checkpoint attributable outcomes such as an invalid witness, provider timeout, or `witness_not_ready` with provider, scope, piece and observation time. Keep evidence even when fallback succeeds. A quota response is capacity evidence, not proof of lost data. Local saturation, owner loss, cancelled hedges, bad local credentials and a missed chain deadline are not provider failures. A bounded dispatcher transfers committed failures idempotently to Provider assessment and deletes their inbox entries only in the same local transaction that records delivery. Neither per-attempt HTTP issue nor every unobserved outcome needs durable tracking; taking over may repeat a read for a missing need. One round's provider failure is evidence for assessment, not authorization to remove data or to mark an entire SP unavailable.

## Budgets and verification gates

The per-dataset lease is the initial work unit. For a round with $S$ slots and $U$ distinct needs, persist $S$ bindings and at most $U$ verified witnesses; per healthy source, request count is the sum of bounded batches per `(SP, scope, PieceCID)`. Use bulk inputs/outputs for mappings and checkpoints; do not perform a database or chain round trip per proof slot. Account for heartbeat/index churn per **leased** dataset, not per witness. Derive scan rate and concurrency from peak simultaneously due datasets, SP capacity, and measured alternate-source latency. For failover require lease expiry + discovery + acquisition + reload/revalidation + remaining collection + submission reserve to fit the original window. Measure rather than fixing numeric lease, backoff, and concurrency defaults here.

| Scenario | Required observation |
| --- | --- |
| Two claimers, concurrent wake, or owner loss mid-request | Different live claims, no lost wake, stale owner writes rejected, successor keeps committed witnesses and the original deadline. |
| Duplicate slots, slow provider, corrupt branch, or quota response | Exact ordered calldata, bounded fan-out, sufficient fallback time, preserved attributable evidence. |
| First proof before backend AddPieces | An authorized parked Upload SP supplies verified witnesses, including for a large piece after cache restart, before the original cutoff. |
| Null epoch or reorg with unchanged drand seed | Wait for usable chain inputs; reject an obsolete mapping before issue and reconcile an escaped nonce before incompatible work. |
| Crash after enqueue or broadcast; proof receipt invalidated | Durable request and signed identity recover; same-nonce replacement does not duplicate logical work; revised outcome can reopen schedule reconciliation. |
| Deletion tail changes, removal batches cross a restart, or deletion receipt reorgs | Only actual receipt IDs update their exact interests. Next period does not wait for the backend release gate, but copies stay protected when removal evidence is stale. |
| Ordinary transactions occupy low nonces during a proof burst | Admission/backpressure preserves the measured submission reserve for proving; if one sender cannot meet it, reassess sender allocation before increasing witness fan-out. |

Deployment gates: pin the actual verifier/listener versions and chain RPCs, exercise first activation and reactivation around inclusion/window boundaries, validate chain view and nonce-resolution behavior under reorg, and measure the Curio staging and fallback path. The design does not presume those checks have passed.
