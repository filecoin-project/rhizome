# Storage identity and invariants

An admission is one accepted signed AddPieces request, optionally combined with dataset creation. This model defines storage identities, protection accounting, consistency rules and [admission records](#admission-persistence). The [workflow](storage-workflows.md) defines transaction boundaries and module APIs; terms are in the [glossary](../../GLOSSARY.md).

## Protection policy

Storage management targets three suitable PDP placements at distinct SPs and shares compatible established placements and in-flight work across client interests. A piece may have several placements at one SP, including during removal/re-addition overlap.

[Execution contracts](execution-and-integrations.md) own admission identity, replay, upload routing, and source confirmation. [Funding policy](funding-and-lifecycle.md#agreed-admission-policies) owns client preflight and known-shortfall gates.

## Independent facts

| Fact | Meaning |
| --- | --- |
| Client interest | One client-facing relationship requires protection for a piece; demand starts at acceptance, before chain inclusion |
| Availability observation | An attributed, timed observation that a source serves the piece |
| Placement | An intended or established storage relationship at an SP, with its exact backend membership |
| External operation | A requested effect with stable identity, payload and observed outcome |

Compute protection per interest from these records using the accounting rules below.

## Logical identities and ownership

Each record has a defined identity and owner:

| Identity / record | Owner | Key and purpose |
| --- | --- | --- |
| PieceCID | Referenced by domain owners | Immutable content identity |
| Client-facing dataset and membership | Client service | Chain/contract context and assigned client-facing membership key |
| Admission | Client service | Chain + FWSS contract + payer + client nonce, bound to the accepted request; creation and addition share nonce constraints |
| Admission entry | Client service | Request identity + zero-based entry index, retained through confirmation, membership assignment and removal |
| Placement identity | Storage management | Exists independently of backend assigned IDs; distinguishes old/outgoing and new placements at the same SP |
| Backend membership | Fields of a committed placement, owned by Storage management | SP, chain/contract context, backend dataset ID and piece ID identify the exact relationship targeted by removal |
| Transfer identity | Storage management | Stable replay/status identity and exact request, retained through retry and takeover |
| Commit-batch identity | Storage management | Fixed destination, ordered placement/piece entries, payload and authorization; each placement's outcome maps to its corresponding entry |
| Removal identity | Storage management | Exact targeted membership and immutable issued request, independent of renewed demand |
| Logical transaction request | Transaction management | Stable caller operation key; distinct from transport nonce, signed transaction hash, and replacement attempts |
| Chain view | Shared chain access | Chain identity, immutable block/tipset reference, observation time and confidence |
| Availability / health observation | Assessment/storage interfaces | Source attribution and the evidence used for assessment and action |

Each ordered AddPieces entry retains its admission/interest identity when confirmation assigns a client-facing membership. Repeated PieceCIDs therefore produce separate, independently removable interests, even when they share protection. A fresh request nonce creates new additions. The [admission execution contract](execution-and-integrations.md#request-and-entry-identities) defines keys, shared create/add nonce constraints, replay, and correlation.

Storage management owns the durable original-piece-to-representation/placement mapping. Each initial placement stores the original piece directly. Location and proof APIs expose typed original-piece, direct-representation and placement references.

## Demand and protection accounting

For a particular interest, accounting must distinguish:

- **Established protection:** committed placements suitable for the interest, with usability assessed from current evidence.
- **Available sources:** observed usable data locations, including staging and outgoing placements, that can supply replication or repair.
- **Planned protection:** suitable outstanding placement work, including transfers and unresolved commitments, used to calculate the remaining deficit. When a stalled commitment stops counting is [unresolved](README.md#4-complete-storage-maintenance-and-retrieval).
- **Outgoing protection:** placements with authorized removals, excluded from established protection.

Count distinct suitable SPs toward each active interest's three-SP target. Shared placements can satisfy several interests simultaneously.

Alice and Bob can share three suitable placements. Alice's withdrawal leaves the protection Bob requires in place. Coordinate this sharing [per piece](#planning-consistency-protocol).

### Client piece deletion

After canonical inclusion of [`PiecesScheduledForRemoval`](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/PDPVerifier.sol#L1029-L1040) for an exact membership, Client service changes the interest to `retain_for_proving` in the same transaction that records the scheduling outcome. The piece can still be challenged. Under the piece lock, Storage management retains all usable committed copies; do not intentionally downsize because an interest is deleting. If there is no continuing, authorized, witness-capable committed copy, plan to establish one. A staged copy can supply witnesses or repair bytes but does not satisfy this requirement. Other active interests still require their normal protection.

Proving processes scheduled deletions and supplies the exact [`PiecesRemoved`](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/PDPVerifier.sol#L1093-L1110) outcome to Client service. Transition to `released` only when that membership's removal meets the [irreversible-action confidence gate](execution-and-integrations.md#chain-confidence-for-irreversible-actions). Retain the release evidence on the interest and validate its branch before authorizing or issuing removals that depend on it. If a reorg invalidates the removal or its confidence, restore `retain_for_proving` if the schedule still applies, or `protect` if it no longer does and the membership remains live. Restore demand in the transaction that applies the correction. Preserve the removal intent for reconciliation. Never apply an old outcome to a re-added membership.

## Invariants and atomicity

1. **One entry, one continuing interest:** acceptance applies demand atomically; ordinary confirmation does not rewrite the interest.
2. **Relationship-scoped changes:** apply withdrawals and restorations only to the original entry, using the exact inclusion binding when membership evidence is involved.
3. **Shared retention:** a placement required by any continuing compatible interest cannot be authorized for removal as though unneeded.
4. **Serialized planning decisions:** each committed plan accounts for interests, placements, and outstanding work within its consistency scope. Two transactions cannot both publish additions based on the same stale deficit.
5. **Removal orders against demand:** removal authorization checks current locally applied demand and records outgoing state plus the exact removal operation atomically under the same consistency protocol as demand changes.
6. **Immutable escaped work:** a possibly issued request/authorization retains its identity and payload. Demand changes trigger reconciliation, not rewriting its history.
7. **One frozen batch per placement:** a placement appears in at most one frozen batch. Its entry is the assignment; alternate authorizations use new placement identities, though parked bytes may be reused. Signing/network I/O occurs after freezing.
8. **Specific outcome application:** record an operation result and its exact affected placements atomically, with chain reference where applicable. Reapplying the observation cannot create another placement or erase unrelated/newer work.
9. **Unresolved effects remain tracked:** lost responses, timeouts, cancellation and expired claims preserve the operation for reconciliation.
10. **Recoverable follow-on work:** applying an interest or outcome atomically leaves its reconciliation discoverable.

Modules access records through their owners' interfaces. Planners use Storage management's applied interest view. A new admission commits its interest under the same piece lock as removal authorization. An authorized removal counts as outgoing until it is resolved or cancelled. Under the piece lock recheck demand before marking a removal as possibly issued; cancel an unissued removal if the placement is needed again.

## Reuse compatibility

The initial offering applies common provider-eligibility rules across clients. Reuse depends on content and placement suitability; clients with different prices, currencies and StorageTermIds can share protection while retaining their commercial obligations.


## Planning consistency protocol

Use one stable coordination row per PieceCID within the MSP's storage scope. All interests, destinations and client terms for that piece share this serialization point.

### Single-piece decisions

1. Start a short PostgreSQL transaction at `READ COMMITTED` and ensure the canonical coordination row exists through a uniqueness-protected creation path. Concurrent first admissions must converge on that same row.
2. Acquire its row lock before reading mutable decision inputs.
3. After acquiring the lock, read current interests, placements, and operation/batch state. After waiting, see the predecessor's committed changes rather than a stale pre-lock snapshot.
4. Apply the domain change and publish durable follow-on work atomically. Preserve stable external operation identities and the exact immutable payload of issued work.
5. Commit, releasing the lock. Execute external I/O only from durable intent after that transaction.

All planning-fact mutations follow this protocol: interest application, placement intent, transfer results, removal authorization, batch assignment and backend outcomes. A planning pass is SQL-only and uses the piece lock, not a worker lease. Leases fence the separate executors that span I/O.

When a fact change does not plan immediately, it leaves the piece's due marker set in the same transaction. The planner later acquires the same row and reads current inputs.

Retain coordination rows while interests, placements, unresolved operations or reconciliation refer to them. Creation and any later cleanup preserve one serialization point under concurrent access.

### Multi-piece batches and lock order

Batch discovery reads candidate ready placements. A bounded freezing transaction takes distinct piece locks in canonical order, rechecks need/readiness and absence of a prior frozen entry, then validates that the destination dataset is open for additions and its on-chain ID matches its finalized creation result. Persist the exact ordered entries atomically, preserving every signed entry and interest.

For paths that combine these locks, use this order:

1. Shared `chain.reconciliation_basis` row (`FOR SHARE`) if validating chain applicability; then Client admission rows in ID order if the transaction changes Client records.
2. Distinct piece rows sorted by `(managed_scope, piece_cid)` bytes, then destination slots/datasets by ID.
3. FWSS nonce claims by `(chain_key, fwss, payer, nonce)`, then placement, batch and operation rows by ID. Take Transaction management request locks after the piece locks on paths that use both.
4. Separate scheduling rows last, under the [work coordination contract](execution-and-integrations.md#durable-work-coordination).

Acquire all affected piece locks before applying a multi-piece change. A caller that learns it needs another piece key after taking later locks rolls back and retries with the full key set. Storage paths do not acquire Client admission locks after taking piece locks.

Apply batch outcomes under the affected piece locks, recording the outcome and exact placement changes together. Fetch receipt/nonce evidence before entering the transaction, then validate its applicability during application.

Bound transaction work and lock acquisition. On contention, timeout or deadlock, roll back the attempt and retry the durable work.

### Assessment inputs

The protocol serializes local planning facts. Provider assessments and chain observations supply attributed decision evidence; changes trigger bounded reassessment of affected placements. Planning and batch-freezing budgets follow the workload envelope.

### Required concurrency scenarios

- Two initial admissions of the same piece create/find one coordination row. The second planner sees the first's published placement work, even if the clients use different StorageTermIds.
- Alice's withdrawal and Bob's acceptance order through the same piece row. If Bob accepts first, removal authorization sees his interest. If removal was authorized first, recheck Bob's demand before marking it as possibly issued. A possibly issued removal remains outgoing and is reconciled against its exact target.
- Two batch workers selecting overlapping placements cannot assign the same placement to different frozen batches.
- Applying A-old's removal and A-new's addition, in either order, changes only their specific placements.
- Process loss releases PostgreSQL locks without erasing committed intent. Recovery reacquires the row and re-evaluates; it does not depend on the former instance returning.

## Admission persistence

SQL schemas identify module ownership. Local IDs are UUIDs and local revisions are positive `bigint`s. Protocol integers use unsigned 256-bit values, encoded as decimal strings in JSON. `chain_key` identifies a configured chain. Addresses and hashes are binary; PieceCIDs use validated canonical v2 bytes.

### Acceptance records

`client.admissions` stores the accepted request envelope, transaction reference and decision evidence. Client service creates it after authorization, preflight and parking checks in the [acceptance transaction](storage-workflows.md#acceptance-and-concurrent-replay). `client.entries` records its ordered pieces.

`tx.fwss_nonce_claims` assigns each `(chain, FWSS contract, payer, nonce)` to a module's logical operation and authorization digest. Transaction management owns the table; Client service and Storage management insert claims through its API. Creation and addition share this key space. Insert claims in key order and commit them with the operation.

Owner APIs validate authorization bindings, complete nonce claims, contiguous entry indexes and an applied interest per entry before commit.

Client service owns preflight upload records. Retain the stable MSP upload ID, authenticated client and client-scoped idempotency key, immutable constraints, chosen SP and endpoint binding, and creation deadline. Retain enough to replay the original preflight without extending that deadline. This record authorizes staging and identifies the provider; it is not an admission, a nonce claim, or evidence of uploaded bytes.

```sql
CREATE DOMAIN uint256 AS numeric(78, 0)
    CHECK (VALUE >= 0 AND VALUE <=
      115792089237316195423570985008687907853269984665640564039457584007913129639935);
CREATE DOMAIN evm_address AS bytea CHECK (octet_length(VALUE) = 20);
CREATE DOMAIN evm_hash AS bytea CHECK (octet_length(VALUE) = 32);
CREATE SCHEMA client;
CREATE SCHEMA tx;
CREATE SCHEMA storage;

CREATE TABLE client.admissions (
    id uuid PRIMARY KEY,
    chain_key text NOT NULL,
    fwss evm_address NOT NULL,
    payer evm_address NOT NULL,
    add_nonce uint256 NOT NULL,
    create_nonce uint256,
    client_data_set_id uint256 NOT NULL,
    routed_data_set_id uint256,
    request_kind text NOT NULL CHECK (request_kind IN ('add', 'create_add')),
    format_version integer NOT NULL CHECK (format_version > 0),
    bound_request jsonb NOT NULL CHECK (jsonb_typeof(bound_request) = 'object'),
    admission_evidence jsonb NOT NULL
        CHECK (jsonb_typeof(admission_evidence) = 'object'),
    add_authorization bytea NOT NULL,
    create_authorization bytea,
    transaction_request_id uuid NOT NULL UNIQUE,
    applied_outcome_revision bigint NOT NULL DEFAULT 0
        CHECK (applied_outcome_revision >= 0),
    current_inclusion_revision bigint CHECK (current_inclusion_revision > 0),
    accepted_at timestamptz NOT NULL DEFAULT now(),
    CHECK (
      (request_kind = 'add' AND routed_data_set_id IS NOT NULL
       AND create_nonce IS NULL AND create_authorization IS NULL)
      OR
      (request_kind = 'create_add' AND routed_data_set_id IS NULL
       AND create_nonce = client_data_set_id AND create_nonce IS NOT NULL
       AND create_authorization IS NOT NULL)
    ),
    CHECK (create_nonce IS NULL OR create_nonce <> add_nonce)
);

CREATE TABLE tx.fwss_nonce_claims (
    chain_key text NOT NULL,
    fwss evm_address NOT NULL,
    payer evm_address NOT NULL,
    nonce uint256 NOT NULL,
    owner_module text NOT NULL,
    operation_id uuid NOT NULL,
    role text NOT NULL CHECK (role IN ('create', 'add')),
    authorization_digest evm_hash NOT NULL,
    PRIMARY KEY (chain_key, fwss, payer, nonce),
    UNIQUE (owner_module, operation_id, role)
);

CREATE TABLE client.entries (
    admission_id uuid NOT NULL REFERENCES client.admissions(id),
    entry_index integer NOT NULL CHECK (entry_index >= 0),
    piece_cid bytea NOT NULL CHECK (octet_length(piece_cid) > 0),
    managed_scope uuid NOT NULL,
    source_sp uuid NOT NULL,
    service_requested boolean NOT NULL DEFAULT true,
    PRIMARY KEY (admission_id, entry_index)
);

CREATE TABLE client.admission_work (
    admission_id uuid PRIMARY KEY REFERENCES client.admissions(id),
    due_at timestamptz
);
CREATE INDEX client_admission_due ON client.admission_work (due_at, admission_id)
    WHERE due_at IS NOT NULL;
```

`bound_request` preserves the complete typed envelope: EIP-712 domain, authorization digests, signed fields and metadata representation, dataset route or creation variant/terms, and call semantics. Preserve signed array order and use consistent scalar/binary encodings. Equivalent transport encodings and separately valid equivalent signatures compare equal on replay; execution retains the first accepted authorization and call.

Entries retain their PieceCID and selected SP. `admission_evidence` associates each ordered entry with its MSP upload ID for routing and retains [parked-source evidence](execution-and-integrations.md#parked-source-evidence) by `(source_sp, piece_cid)`, including trusted endpoint/version, observation time, and guaranteed deadline. Share evidence for the same source and PieceCID. Decode raw size from the signed v2 CID. Transport upload IDs do not prove provenance or change the signed authorization or accepted admission identity.

`service_requested` records the client's continuing service intent, not its current retention requirement. Acceptance sets it true; explicit withdrawal clears it. A chain-derived admission failure can release Storage demand while preserving service intent for reorg recovery. An on-chain removal follows the [deletion rules](#client-piece-deletion), not the explicit-withdrawal path. Request cancellation preserves accepted intent.

### Execution and inclusion records

Record keys and fields follow. Use foreign keys within a module and owner APIs for cross-module references.

| Record / owner | Required fields and constraints |
| --- | --- |
| `tx.requests` / Transaction management | `id` PK; `UNIQUE(caller_module, caller_operation_id)`; exact call/sender/chain, caller wake key, format version, **one current outcome**, its revision/nullable inclusion revision, validated branch token and retry due time. Resolved outcomes become due when their branch token is stale; unresolved work retains timed retries. |
| `tx.sender_accounts` / Transaction management | PK `(chain_key, sender)`; attributed chain nonce evidence. Lock the row to allocate against live local reservations. |
| `tx.nonce_assignments` / Transaction management | `id` PK; request FK, chain/sender/transport nonce, ownership generation/expiry and resolution reference. Partial unique indexes reserve each live nonce and one live assignment per request. Replacements share it; expiry cannot release the reservation. |
| `tx.attempts` / Transaction management | PK `(request_id, attempt_number)`; request-scoped assignment FK, `UNIQUE(chain_key, tx_hash)`, signed bytes and broadcast/reconciliation state. Number persisted signed transactions from 1; the first hash is the original response alias, and every retained hash resolves to its request. |
| `tx.receipts` / Transaction management | Key `(chain_key, tx_hash, immutable_block_ref)`; correlated receipt/log evidence and optional local-attempt link. Supports MSP submissions, externally recovered execution and conflicting nonce consumption. Retain evidence needed by replay or effects. |
| `chain.reconciliation_basis` / Shared chain access | PK `chain_key`; observed branch token. It changes on a known fork and remains stable through ordinary head advances. Publication and outcome reads hold a shared row lock while validating it. |
| `client.admission_work` / Client service | PK/FK by admission ID; indexed `due_at` only. Register it with the admission so Transaction management can wake it without taking the admission lock. See [lease-free outcome application](storage-workflows.md#durable-discovery). |
| `client.inclusions` / Client service | PK `(admission_id, inclusion_revision)`; verifier, dataset, first assigned piece ID, entry count and receipt reference. Admission-scoped FK from `admissions.current_inclusion_revision`. |
| `client.memberships` / Client service | Current reverse lookup: PK `(chain_key, verifier, data_set_id, piece_id)`; `UNIQUE(admission_id, entry_index)`; entry and inclusion FKs. Retire invalidated mappings before installing replacements. Derive historical ordered IDs from validated first-ID/count and immutable entries. |
| `client.removals` / Client service | Resolved `(admission_id, entry_index, inclusion_revision)`, exact chain/verifier/dataset/piece target, and immutable authorization/observation reference. Persist before handoff or issue; apply scheduling and processing outcomes to that binding. Retain the evidence used to release demand. |
| `payments.reconcile_due` / Payments | PK `(chain_key, fwss, payer, client_data_set_id)`; coalesced wake for dataset discovery. Payments resolves and observes shared account/rail scope through receipts or targeted state reads. |

The **inclusion revision** is the outcome revision that established the current successful inclusion. Carry it through subsequent evidence updates. Invalidation clears it; restoration or a changed binding uses the new outcome revision. Revalidate inclusions against the current branch token before applying their mappings.

Use [transaction outcome publication](execution-and-integrations.md#transaction-outcome-publication). Callers record applied revisions with their own domain writes. Retirement leaves the admission due until the replacement or current non-inclusion outcome is applied.

The SDK derives success and ordered IDs from transaction inclusion. Client service applies the corresponding inclusion, entry and membership records independently. Retain receipt evidence and removal targets needed by unresolved work and supported replay.

Transport allocation locks the sending account, checks attributed chain evidence and reservations, and reserves an available nonce. Sign outside the transaction and persist signed bytes under that assignment's live ownership generation. Resolve the old assignment and current operation intent before allocating a replacement transport nonce.

Before reconciliation reads, capture operation revision, ownership and branch token. Take the branch-row and operation locks and recheck those values before publishing. `ReadOutcome` returns `revalidation_required` until its saved branch is validated. Sender allocation takes the account lock before the request lock. These paths stay within Transaction management and Chain access.

Apply the [recovery principles](../engineering/principles.md#failure-must-not-create-an-unrecoverable-state) to durable records and restartable reads.

### Interest application and reconciliation

Client service owns admission entries and client-facing membership bindings. Apply initial demand and later retention changes through Storage's API in the transaction that records the corresponding Client decision. The caller holds the admission lock and validates the current outcome or exact removal evidence; Storage writes only its own interest records under the piece locks. There is no intermediate interest inbox or delivery revision.

`storage.interests` is keyed by `(admission_id, entry_index)` and indexed by piece. Retain immutable scope/CID and initial source attribution. Record only the retention requirement: `protect` for normal protection before or after confirmation; `retain_for_proving` for a deleting membership under the [deletion rules](#client-piece-deletion); or `released` for no demand. Retain applicable release evidence and its exact membership binding when release depends on on-chain deletion. Client service owns the membership lookup; routine confirmation does not update Storage.

If retention or the evidence Storage needs changes, make the piece due in the same transaction. Applying the same decision again leaves the interest unchanged. A stale outcome cannot change demand: check the current branch and outcome or exact removal binding before calling Storage, rather than ordering forwarded snapshots.

`storage.pieces.plan_after` schedules per-piece reconciliation:

```sql
CREATE TABLE storage.pieces (
    managed_scope uuid NOT NULL,
    piece_cid bytea NOT NULL,
    plan_after timestamptz,
    PRIMARY KEY (managed_scope, piece_cid)
);
CREATE INDEX pieces_due ON storage.pieces
    (plan_after, managed_scope, piece_cid)
    WHERE plan_after IS NOT NULL;
```

Under the piece lock, a planning-fact change makes `plan_after` due, preserving any earlier due time. A worker claims the due piece by taking that row lock, for example with `FOR UPDATE SKIP LOCKED`. After acquiring it, read current facts, publish only the remaining deficit, and clear or reschedule `plan_after` in the same transaction. There is no separate job or lease. A later writer takes the same lock and makes the piece due again.

### Initial protection records

Storage management records placement work, transfers, fixed commitment batches and exact backend memberships:

| Storage-management record | Required fields and constraints |
| --- | --- |
| `dataset_slots` and `datasets` | Slot PK `(managed_scope, sp_id)` selects the open backend dataset. Persist standalone creation intent and its nonce claim before signing. A dataset accepts batches only once its on-chain ID and creation meet the required finality. |
| `placements` | `id` PK; original piece scope/CID FK, destination SP/dataset, lifecycle and current backend membership. Derive typed direct representation from the stored key. Index by piece/destination; allow multiple placements of a piece at one SP. Each frozen authorization has its own placement IDs. |
| `transfers` | `id` PK; one piece via its placement FK, source/endpoint, destination, exact versioned pull parameters, provider replay scope/correlation, issue/outcome and due fields. Persist before issue. JWT renewal changes credentials, not transfer identity. Success supplies the exact staged source and retention deadline. |
| `batches` | `id` PK; destination/dataset, immutable unsigned ordered payload/digests and FWSS nonce claims, `frozen_at`, nullable signed authorization/exact provider request, observation and due fields. Persist signed bytes plus submission work before issue; once populated they are immutable. Each placement's first commitment deadline is retained across discovery/failover. |
| `batch_entries` | PK `(batch_id, entry_index)`; placement FK and CID; `UNIQUE(placement_id)` and `UNIQUE(batch_id, piece_cid)`. Entries assign placements to frozen batches. Retry the same batch/authorization; alternatives use new placements and can reuse parked bytes. |
| `staged_sources` | Source bound to scope and PieceCID: the selected provider and PieceCID from admission, or the exact provider staging reference from a successful pull. Retain the known retention deadline, refresh due time, and requested deadline for an unresolved refresh. Publish renewed deadlines under the piece lock; provider calls run outside it. |
| `placement_memberships` | Current reverse lookup binds exact placements. Transfer/batch records hold current outcomes/revisions and required receipt/first-ID/count evidence. Retire invalidated mappings before replacement. Keep the operation due until its membership mapping and outcome application commit; retry from saved evidence. |
| `removals` | Immutable external target, resolved placement and membership inclusion. Persist before issue, revalidate before retry and apply results to the recorded binding. |

Retry unresolved effects and unapplied outcomes when due. After known forks, discover resolved success/revert/conflict through stale branch tokens. Lifecycle and funding observations follow their freshness schedules. Reorg correction restores the service intent retained for the original authorization.

