# Execution and integration contracts

This document supplies admission execution contracts to the [storage model](storage-model.md) and [admission workflow](storage-workflows.md).

## Source-backed authorization boundary

The FWSS and PDP contracts define the following rules for admissions, meaning accepted signed AddPieces requests, including requests that also create a dataset.

| Fact | Consequence for MSP |
| --- | --- |
| FWSS stores `clientNonces[payer][nonce]` in contract state | Scope each nonce to the chain, FWSS contract, and payer. |
| CreateDataSet consumes `clientDataSetId` in the same map | Creation and addition share one nonce namespace. A combined request uses different values for its creation ID and addition nonce. |
| AddPieces signs `clientDataSetId`, `nonce`, ordered `Cid[]`, and indexed piece metadata under an EIP-712 domain | Preserve the complete domain and signed payload, including order and metadata encoding. |
| Existing-dataset HTTP requests route by on-chain `dataSetId`; the signature names the client-chosen `clientDataSetId` | Resolve the route's payer and client ID from the expected FWSS relationship before binding the operation. |
| Synapse generates a fresh nonce unless the signer supplies one | Each fresh nonce identifies a separate authorization. |
| PDP assigns consecutive piece IDs and invokes FWSS in the same transaction | On success, entry `i` gets piece ID `firstAdded + i`. A callback revert rolls back the entire addition. |
| FWSS records the dataset ID and next piece ID as the nonce outcome | Match nonce consumption to the signed request before recording success. |

Use the dataset's validated payer in the nonce key, including when a session key or dataset authorizer signs for that payer. The exact contract preflight determines contract authorization.

## Request and entry identities

### Logical request key

For an AddPieces authorization, use the logical key:

`(chain identity, FWSS verifying-contract address, payer, client nonce)`

Persist one admission for that key. The [FWSS nonce claims (`tx.fwss_nonce_claims`)](storage-model.md#acceptance-records) enforce uniqueness and identify the owning admission across operation kinds and datasets. Client service uses those claims to resolve the key.

Bind that key to:

- EIP-712 domain name/version, chain ID, and verifying contract.
- Exact ordered signed piece CID bytes, client-chosen dataset ID, nonce, and metadata keys/values with their signed representation and order.
- The routed/expected on-chain dataset and its validated payer/client-ID relationship, or the exact dataset-creation authorization when creating it in the same transaction.
- Signed authorization bytes and the exact intended contract call, including the applicable PDP verifier/listener and value.

The authorization digest identifies the signed meaning. Treat equivalent transport encodings and separately valid signatures for identical bound semantics as the same request, and execute the original accepted authorization. Compare metadata in its signed encoding and order and preserve the signed piece order.

Validate identity, authority, payload correlation, and the required admission checks before claiming a nonce, keeping preflight and RPC outside the acceptance transaction. The uniqueness constraint decides concurrent valid attempts. The loser compares its request with the committed admission and follows the replay rules below.

### Per-entry admission and interest

Each accepted ordered entry is identified by:

`(logical request identity, zero-based entry index)`

Client service retains this identity before on-chain piece IDs exist. Storage management uses it for the entry's provisional interest, all later revisions, and promotion of that same interest.

After confirmation, retain the entry's association with `(chain, PDP verifier, on-chain client-facing dataset ID, assigned piece ID)` so membership removal can withdraw the correct interest.

A fresh nonce creates a distinct addition even for the same PieceCID and dataset. Repeated PieceCIDs within one request also stay distinct: `[P, P]` creates two interests, two assigned piece IDs, and independently removable memberships, each with its own metadata. Share compatible source evidence and backend placements; deduplicate content lookups and coordination-lock keys while preserving every signed entry and its ordered outcome.

Curio rejects duplicate subpieces in one HTTP addition. MSP submits the client-facing call through Transaction management and applies Curio's rule to backend batches. Reuse compatible placements, and put any required distinct placements of the same piece at one SP in separate compatible batches. Keep the signed client request, entry outcomes, memberships, and payment semantics intact.

### Revision delivery

Each entry's source revision orders Client service updates for that entry only. Applying an interest update, recording its revision/delivery identity, and leaving required reconciliation discoverable is atomic at Storage management's boundary.

Use the same key for provisional acceptance, confirmation, definitive failure, and later removal. A failure or removal applies only to its own entry identity. Use the model's piece coordination rows to serialize competing placement decisions.

## Replay behavior

After validating caller authority and correlating the submitted request, apply these rules:

| Situation | Behavior |
| --- | --- |
| Same logical key and same bound request already accepted | Recover the existing admission and its saved response/status identity, bypassing new-admission gates and reusing its transaction and interests. |
| Same logical key but different signed semantics, route, or creation envelope | Reject the conflict and leave the original unchanged. |
| Fresh key with the same PieceCID | Run [new-admission checks](funding-and-lifecycle.md#agreed-admission-policies); reuse compatible backend protection. |
| Original signed request was rejected before durable acceptance | Retry the checks; no accepted nonce claim exists. |
| Accepted operation has no known external result | Reconcile the same operation and return its actual unresolved status. |
| Accepted operation has a conclusive terminal result | Return its retained result. |

After an uncertain attempt, the client must resend the original signed `extraData` and ordered body. Calling high-level `commit()` again can generate a fresh nonce, and if the original attempt created a dataset whose ID the client never learned, a later commit can create another dataset. The SDK needs a way to replay the signed request; neither the transaction hash nor the PieceCID replaces that request. Cancellation and HTTP timeouts end the wait; accepted obligations remain.

## Transaction-hash response contract

On the PDP-compatible POST route for existing-dataset addition, return `201` with a `Location` ending in the first real transaction hash. Synapse extracts `txHash` from that header and retains `extraData` and the piece count for chain-based waiting and recovery.

After durable admission, wait outside the acceptance transaction for Transaction management to persist the signed transaction bytes, their hash, and durable submission work. Return that hash only once these records exist. If the deadline expires first, use the HTTP error path and leave the accepted work and interests intact. The concrete timeout/error mapping belongs to API integration.

Use the durable operation record to recover after a crash or lost response. Replay returns the saved first hash when available. Keep that hash as a stable status lookup alias for the logical operation, even after fee replacement. Retain attempt hashes and use the SDK's `confirmedTxHash` field when the actual confirmed hash differs.

The POST acknowledges accepted transaction work; SDK storage success uses the completion path below. Preserve operation-specific status schemas, including the ordered assigned piece IDs and distinct combined-creation result fields. Keep durable request records and aliases for unresolved work and the supported replay window.

## AddPieces inclusion and reorgs

Successful inclusion in the observed canonical chain is the AddPieces confirmation boundary:

| Outcome | Completion condition |
| --- | --- |
| Client-facing SDK success | Client-to-MSP AddPieces has succeeded on chain. The SDK waits on the returned transaction hash and recovers its ordered result from chain evidence. |
| Backend placement completion | The backend SP's AddPieces has succeeded on chain. |

Both complete at successful inclusion without an additional confirmation-depth wait; backend completion needs no first PDP proof. MSP confirms parking before acceptance. The SDK follows transaction inclusion independently of MSP's local outcome application and backend replication.

Proceed with dependent work at inclusion. If a reorg removes it, keep accepted demand and issued operations, refresh chain observations, and preserve completed physical transfers:

- **Client-facing addition:** if its client nonce is unconsumed in the new chain view, resubmit the original authorization under the same admission. Transaction management handles sending-account nonces and replacement.
- **Backend addition:** resolve the previous operation against the new view, then resume the required placement and commit flow, reusing parked bytes where possible. Preserve operation-to-membership correlation through recovery.

Continue reconciliation after SDK success. Other chain actions use their own confidence rules.

### SDK transaction waiting

The SDK's `waitForAddPieces` and `waitForCreateDataSetAddPieces` wait on chain receipts and recover using FWSS nonce state. Preserve this completion path for MSP: return the real transaction hash, and let the SDK derive success and assigned IDs from chain evidence.

## Correlating success and recovering an unknown outcome

Prefer Transaction management's persisted attempts and matching receipt/logs for MSP-submitted client-facing work. Validate expected chain/contract, dataset, entry count, ordered contents, and the applicable authorization correlation before applying success. Keep the chain reference and action-specific confidence.

FWSS encodes an AddPieces nonce outcome `v` as follows:

- `datasetId = v & ((1 << 128) - 1)`
- `nextPieceId = v >> 128`
- For an expected request of `N` entries: `firstAdded = nextPieceId - N`, and entry `i` maps to `firstAdded + i`.

Validate the encoding, range, and context before using the candidate mapping. Zero upper bits on a consumed creation-only nonce are not an AddPieces outcome. Use the historical nonce result or matching receipt rather than the current dataset piece count.

When a nonzero nonce outcome might belong to a different signed request, recover the matching transaction/receipt or other authoritative evidence of the expected contents. Use targeted discovery as needed; dataset and count alone do not establish the match. Retain unresolved or conflicting outcomes until the evidence supports a decision. Conflict recovery stays bound to the original authorization.

A zero outcome establishes only that the nonce has no successful consumption in that chain view; its transaction may still be pending. Reconcile the original authorization against refreshed observations after a reorg, keeping its accepted identity and physical transfers.

## Combined dataset creation and addition

The SDK's first commit can create a dataset and add its first pieces through `POST /pdp/data-sets/create-and-add`. Persist both signed authorizations, the complete ordered body, both FWSS nonce claims, the transaction request, and the entry handoffs atomically. Creation claims the `clientDataSetId`; addition claims the nonce. Both use the same payer/FWSS namespace and must be distinct.

Until the transaction succeeds, identify each entry by the accepted request and its index. On success, attach the on-chain dataset and piece IDs to those entries. The verifier creates the dataset and appends pieces in a single transaction; a revert rolls back both actions.

Compare the entire combined envelope on replay, including the creation payload and its dataset metadata and payee. Use the real-hash response and the SDK's combined-operation result and wait contract.

FWSS supports `CreateDataSetWithStorageTerms` and its terms-aware creation `extraData`. Bind the selected `StorageTermId` and complete creation variant so execution and replay use the same terms. Synapse's signing helper exposes the legacy creation shape; signing the terms-aware variant requires SDK integration.

## Upload preflight and destination handoff

For a standalone introduction to the approach, see the [direct tus upload proposal](../proposals/tus-upload-proposal.md). This section defines the current design.

Synapse calls the authenticated MSP `POST /uploads/preflight` before sending bytes. Preflight accepts an optional expected PieceCID or raw length, works before either is known, and checks staging limits. MSP selects an Upload SP and persists a client-bound upload record without allocating a resource at the SP. Replaying the same client-scoped idempotency key and parameters recovers that upload identity; different parameters conflict. Preflight authorizes staging only; it reserves no funds and accepts no storage obligation.

Return an MSP `uploadId`, the provider's absolute HTTPS `creationUrl`, a scoped `uploadGrant`, `maxSize`, and `creationExpiresAt`. The grant is an MSP-signed bearer JWT verified by the SP using MSP's public keys. Bind issuer, provider audience, operation, expiry, upload ID, client, size limit, and any known size or expected PieceCID. Use the CID's encoded raw size when present. Credential renewal preserves the upload identity and does not extend parking.

The SDK creates the resource at the provider with tus 1.0.0 Creation, uploads directly with PATCH, and uses resource HEAD to recover its durable accepted offset. Unknown lengths use `creation-defer-length`; the provider enforces the size limit throughout and enforces any known size exactly. Preserve the full resource and result URLs, including deployment prefixes. Expose browser CORS on the required routes and send credentials only to authorized provider origins. Resumption requires replayable input.

The provider persists at most one resource per MSP upload ID, bound to its grant, before returning `201 Location`. Equivalent creation retries recover the resource without resetting its offset; conflicting parameters fail and expired resources cannot be recreated under the same ID. Persist acknowledged bytes and offsets, and on completion durably schedule restartable CommP computation, expected-CID validation, and retrieval publication.

The provider advertises a repeatable, authenticated result lookup on creation and resource HEAD responses. MSP can also look up that result at the selected provider by its own upload ID without the client's resource URL or bearer token. The result reports `receiving`, `processing`, `parked`, `failed`, or `expired`; `parked` names the upload ID, resulting PieceCID, and staging retention deadline. The MSP issuer and upload ID resolve to the exact provider staging reference. Completing transfer or processing does not consume the handle. No client finalize request is required. Creation With Upload, parallel partial uploads, migration during upload, and already-present deduplication are outside this flow.

The SDK sends signed AddPieces to MSP, with one `uploadId` per ordered entry alongside the signed body. IDs are transport metadata, not FWSS authorization fields. For a new admission, match each ID to the authenticated client, its selected SP, and the parked result for that signed PieceCID. A batch may span SPs; duplicate signed entries remain distinct and may share the same source. Resolve endpoints from the stored provider binding, never a client-supplied retrieval URL. For an accepted replay, use the retained attribution and do not consult or require a live upload resource.

Provider support for grant enforcement, idempotent creation and the piece result is an application contract, not a feature of [tus 1.0.0](https://tus.io/protocols/resumable-upload).

## Parked-source evidence

The authenticated tus result correlates an upload with a parked PieceCID and retention deadline. The result, retrieval HEAD, and a parking refresh are distinct evidence: identity, reachability, and retention respectively. Curio's existing reference and indexing APIs do not establish these facts together. Verify that the deployed retrieval route can read staged bytes before membership; publishing a reference alone does not guarantee open-pieces placement succeeded.

### Mandatory confirmation: retrieval HEAD

Before durable admission, MSP performs HEAD on the selected Upload SP's configured full `/piece/<expected-v2-CID>` retrieval URL. Require HTTP 200 and a valid `Content-Length` that exactly matches the raw size encoded in the signed expected PieceCID. The response must come from the selected SP's direct source endpoint and confirm current access rather than reuse stale cached headers.

Bound end-to-end time and response-header size, send `Accept-Encoding: identity`, require compatible content encoding, and read only the headers. Timeout, an unattributed response, or a failed response check leaves the source unconfirmed. Use HEAD as the source check; neither reference lookup nor upload/indexing evidence substitutes for it, and there is no implicit fallback to GET or a reference-creating POST.

Confirm the selected Upload SP. Repeated entries for the same source and PieceCID may share one correlated observation. Accepted replay recovers the existing operation without repeating admission checks.

HEAD confirms readable access and advertised length under the trusted Upload-SP model. Upload correlation and retention rely on their own evidence, and destination pulls verify the full content.

If initial HEAD does not confirm the source, reject or defer before durable acceptance, leaving interests, transaction work, and nonce claims uncreated. Keep attribution and distinguish unavailable or unknown evidence from a confirmed identity or length mismatch. Source loss after acceptance triggers fulfillment and reconciliation while demand remains in force.

### Parking refresh and retention

Curio must expose an authenticated MSP-to-SP parking refresh endpoint for existing, uncommitted staging. Address the exact staging reference established by an upload result or a successful pull item, and supply its expected PieceCID and requested absolute retention deadline. The provider atomically verifies the binding and extends the GC protection monotonically, returning the effective guaranteed deadline after committing. Repeating the same request must not allocate a reference or move its deadline further. If the reference is missing, expired, or already claimed for deletion, return that result rather than claiming it was renewed. The guarantee must serialize with cleanup on the provider; it does not protect against SP or disk failure.

Before acceptance, establish enough remaining retention for each selected source to cover handoff, queueing, transfer or commitment, and recovery; refresh if the observed deadline is too close. If there is not enough proven time after accounting for observation age and clock uncertainty, defer or reject rather than accepting on HEAD alone. Persist the exact staging reference, evidence attribution and time, and guaranteed deadline with the admission. Storage management takes over durable refresh scheduling, including for pulled staging, while accepted work still relies on that reference. Renew before the safety margin is exhausted; a failed or uncertain refresh cannot advance the known deadline. If availability can no longer be relied on, replan from other sources without withdrawing demand.

Credential expiry, unfinished tus resource expiry, and parked-piece retention are separate. HEAD and polling do not renew parking. Curio currently makes unclaimed PDP references GC-eligible from creation after `max(2, PDPUnclaimedUploadKeepHours)` hours (default two), except where protected by qualifying pending AddPieces work. The refresh contract must integrate with that cleanup; nominal retention is not a guarantee of time remaining at admission.

## Provider pulls

MSP authorizes the destination SP's pull through the same MSP JWT trust and verification contract as tus uploads. Issue a pull-scoped token for the destination, not a forwarded client upload grant. Bind the stable transfer ID and exact pull parameters, including the expected PieceCID(s), source URL(s), and any byte limits, directly or through a request digest. Reissue expired credentials without changing the transfer ID or parameters. The pull authorizes staging, not a PDP membership, and does not require an FWSS signature or a frozen AddPieces batch. The subsequent commitment keeps its own authorization and nonce.

Curio must recover pull status by stable transfer ID scoped to the MSP issuer and destination service. Repeating the same parameters returns the existing operation; different parameters conflict. Apply success to each matching item, not from aggregate status, and return the exact staging reference and retention deadline for each parked piece. Use a new transfer identity for materially changed work. The pinned Curio pull handler keys work on signed `extraData` and performs exact AddPieces simulation; replacing that authorization and lookup is a Curio integration dependency.

Destination SPs must be able to fetch and verify the piece from the configured HTTPS source within their network and URL policies. The JWT authorizes the pull request at the destination, not the source GET. Initially use the existing public retrieval GET; a source requiring credentials needs a separate scoped read contract with the same direct data path.

## Source and SDK evidence

Contract and Curio findings come from source review of Curio `0073d178`, Synapse `f57ff11`, FWSS `352b614`, and PDP. Local Synapse exercises showed:

- With fixture PieceCIDs, an explicit nonce reproduced the authorization. Changing nonce, client dataset ID, piece order, or the encoding of empty metadata invalidated the original signature. Default signing generated distinct nonces, and duplicate entries retained their positions.
- The legacy streaming SDK discarded the cross-origin upload host. The tus SDK must preserve and use the provider's URLs.

## Source references

- [Synapse signing and ABI extraData](https://github.com/FilOzone/synapse-sdk/blob/f57ff119966b483205263b513b1220dd869415b6/packages/synapse-core/src/typed-data/sign-add-pieces.ts)
- [Synapse EIP-712 types/domain](https://github.com/FilOzone/synapse-sdk/blob/f57ff119966b483205263b513b1220dd869415b6/packages/synapse-core/src/typed-data/type-definitions.ts)
- [Synapse compact metadata representation](https://github.com/FilOzone/synapse-sdk/blob/f57ff119966b483205263b513b1220dd869415b6/packages/synapse-core/src/utils/compact-piece-metadata.ts)
- [Synapse HTTP addition/status contract](https://github.com/FilOzone/synapse-sdk/blob/f57ff119966b483205263b513b1220dd869415b6/packages/synapse-core/src/sp/add-pieces.ts)
- [Synapse combined authorization](https://github.com/FilOzone/synapse-sdk/blob/f57ff119966b483205263b513b1220dd869415b6/packages/synapse-core/src/typed-data/sign-create-dataset-add-pieces.ts)
- [Synapse combined HTTP/wait contract](https://github.com/FilOzone/synapse-sdk/blob/f57ff119966b483205263b513b1220dd869415b6/packages/synapse-core/src/sp/create-dataset-add-pieces.ts)
- [Synapse high-level commit and cancellation](https://github.com/FilOzone/synapse-sdk/blob/f57ff119966b483205263b513b1220dd869415b6/packages/synapse-sdk/src/storage/context.ts)
- [FWSS shared nonce storage](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/storage/FWSSStorage.sol)
- [FWSS AddPieces authorization/outcome](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/modules/FWSSAuthorizationModule.sol)
- [FWSS dataset creation identity](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/modules/FWSSDataSetModule.sol)
- [FWSS signature hashing/authorizer boundary](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/lib/LibSignatureVerification.sol)
- [FWSS public nonce lookup](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/modules/FWSSViewModule.sol)
- [PDP atomic addition and ID assignment](https://github.com/FilOzone/pdp/blob/3d486ae9ef6a5040acd55a0430307aa2802103c5/src/PDPVerifier.sol)
- [Curio addition conversion, duplicate checks, and response](https://github.com/filecoin-project/curio/blob/0073d178dcf430db37def3bb55fc279da348121c/pdp/handlers_add.go)
- [Curio existing pull authorization and replay](https://github.com/filecoin-project/curio/blob/0073d178dcf430db37def3bb55fc279da348121c/pdp/handlers_pull.go)
- [Synapse chain-only wait and nonce recovery](https://github.com/FilOzone/synapse-sdk/blob/f57ff119966b483205263b513b1220dd869415b6/packages/synapse-core/src/sp/wait-for-receipt.ts)
- [Curio open-pieces placement](https://github.com/filecoin-project/curio/blob/0073d178dcf430db37def3bb55fc279da348121c/tasks/openpieces/task_place.go)
- [Curio retrieval resolution](https://github.com/filecoin-project/curio/blob/0073d178dcf430db37def3bb55fc279da348121c/lib/cachedreader/hashspace_stage.go)
- [Curio piece HTTP serving](https://github.com/filecoin-project/curio/blob/0073d178dcf430db37def3bb55fc279da348121c/market/retrieval/piecehandler.go)
- [Curio unclaimed-reference cleanup](https://github.com/filecoin-project/curio/blob/0073d178dcf430db37def3bb55fc279da348121c/tasks/pdpv0/task_piece_gc.go)
- [Curio retention defaults](https://github.com/filecoin-project/curio/blob/0073d178dcf430db37def3bb55fc279da348121c/deps/config/types.go)
