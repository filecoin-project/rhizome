# Piece inclusion proofs in Curio

Status: proposal for Curio review; not an implemented or agreed API.

## Purpose

Give an authorized MSP a Merkle witness for any requested leaf of a piece Curio can read, whether the piece is staged or committed. The proof shows that a leaf at a particular position belongs to the piece's CommP. It does not prove inclusion in a backend dataset or transaction, establish retention, or count a staged copy toward the MSP's protection target.

The client-facing AddPieces transaction can land before any backend AddPieces transaction. Curio must be able to serve the first client-facing PDP round from the parked Upload SP copy alone. The MSP resolves each chain challenge to a PieceCID and a piece-relative position, verifies the witness, and submits the PDP proof. Curio does not need the challenge seed, the MSP's dataset ID, or a backend dataset or piece ID. See Rhizome's [proving requirements](../architecture.md#proof-witnesses-and-client-facing-proving).

## HTTP contract

`POST /pdp/piece/{pieceCid}/inclusion-proofs` over HTTPS, with `Authorization: Bearer …` and `Content-Type: application/json`. The path takes a PieceCIDv2; no v1 alias. Each request addresses one piece. This is a read operation: retrying it does not create a reference, extend retention, or start a proof job.

Request body:

```json
{"leafIndices":[1]}
```

`leafIndices` is a nonempty array of distinct, zero-based JSON integers. An index counts 32-byte leaves in the full Fr32-padded commitment tree, not raw bytes or leaves across a dataset. Decode the CID's root and tree height `h` (edges from leaf to root); require `0 <= index < 2^h`. This includes leaves wholly in the zero-padding region. The MSP uses the verifier's actual challenge range for PDP and does not use padding-only witnesses as evidence of data possession.

`200 OK` returns one witness per requested index in request order, or no witnesses on failure. `leaf` is the actual Fr32 leaf, not a hash of it. `siblings` contains exactly `h` nodes, nearest the leaf first, without the root. Each node is `0x` followed by 64 lowercase hex digits. Responses use `Cache-Control: no-store`, including errors.

Example for `bafkzcibcaablqfyjsvd3rrmqmd7n4f7i73ybzuxfq4p4za6lft2jeaaaaguxefq` (127 raw bytes with values 0 through 126):

```json
{
  "witnesses": [
    {
      "leafIndex": 1,
      "leaf": "0x8084888c9094989ca0a4a8acb0b4b8bcc0c4c8ccd0d4d8dce0e4e8ecf0f4f83c",
      "siblings": [
        "0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f",
        "0x8cb3cba6a9d5a5181ad2bbd6020329499585a4679d074800de21d1e3dbec8e24"
      ]
    }
  ]
}
```

Curio checks each branch against the requested CID's root before returning it. The MSP independently checks the response size, count, order, indices, node widths, and exact depth against its original request and expected CID. At each level, the current index's low bit chooses the current node's side (0 left, 1 right); hash `left || right` with SHA-256, clear the top two bits of the digest's last byte, and shift the index right. Accept only if the final value equals the expected CommP. The MSP can deduplicate equal `(PieceCID, index)` requests and then copy a verified witness into every matching PDP challenge slot.

| HTTP status | Meaning |
| --- | --- |
| `400` | Invalid body, unsupported PieceCID or tree height, or duplicate, out-of-range, or too many indices. |
| `401` | Missing or invalid witness credential. |
| `404` | No authorized live source for this CID. Do not distinguish an absent piece from one held for another service. |
| `413` | Request body exceeds the byte limit. |
| `429` | Caller quota exhausted. |
| `503` | Temporary service or source failure. Return `witness_not_ready` when an authorized live source is still preparing; distinguish this from capacity exhaustion and a read or verification failure. |

Errors return a JSON `code` without storage details. The MSP can try another provider within the original round deadline and retains the original failure's attribution even if fallback succeeds.

## Authorization

The MSP signs a short-lived bearer JWT with its private key. Register the public key through Curio's [existing PDP service registration](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/web/api/webrpc/pdp.go#L74-L119) and reuse `JWTAuth`, which looks up the key by the token's `service_name`. Explicitly enable JWT verification on the witness route: the [PDP service defaults to `NullAuth`](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/pdp/handlers.go#L105-L134). Reject anonymous access and require expiry, the configured Curio endpoint as audience, a witness-read scope, and the exact PieceCIDv2.

The token authenticates the requester; separately authorize the exact PieceCIDv2 against that service's live staged reference or committed, readable placement. Deduplication may share physical bytes but must not create read authority; a provider may authorize an already-present copy without binding it to a specific upload. Carry the service binding across staging-to-commitment handoff; expiry or removal of the last qualifying source ends access. Curio resolves internal references; the MSP never supplies them.

For read-restricted data, also require a Keysmith `FOC-Grant` presentation that authorizes read access to the piece under Curio's private-download policy. Send it in a separate `Authorization: FOC-Grant …` field. The service JWT and the ability to open a piece with Curio's internal proving reader do not grant this access. Apply the same checks on warm and cold caches. Resolve private staging policy before publishing readiness; a copy in `open-pieces` is not evidence of permission.

Witnesses disclose piece bytes in Fr32 form. The ability to make many requests can disclose the full piece, so grant witness access only to services allowed to read the underlying data. Do not log tokens, leaves, or branches.

## Staging readiness and bounded execution

Reuse Curio's PDPv0 staged cache task for pieces too large for a bounded in-memory tree. Do not wait for backend AddPieces. Publish readiness only when the authorized read path and any required verified snapshot are usable, not merely when a reference exists or a cache task has been scheduled. The proposed admission signal is `witnessReady` on the authenticated [parking retention response](../design/execution-and-integrations.md#parking-refresh-and-retention): for a witness-capable Upload SP, the MSP requires readiness along with sufficient retention before new admission. HEAD remains a reachability/length check. Readiness is a current observation, not a guarantee against later failure.

Build a small full tree once per requested piece; for larger pieces reuse the upper snapshot tree across indices and read only the distinct challenged sections. Do not scan a large piece, unseal it on demand, or fetch it from another provider on a witness request. Keep a source readable for the in-flight request across cleanup or handoff, or fail without a partial response. A cache alone is not a live source.

Proposed wire caps: 32 indices, 4 KiB request body, 128 KiB response body, and tree height at most 50. Bound storage reads, cache reads, working memory, and per-service and global concurrency independently of wire bytes. Curio enforces an end-to-end handler timeout including queue and storage time; the MSP cancels at its shorter per-attempt deadline when needed to preserve time for fallback and submission. Agree on numeric time, read, memory and concurrency budgets from measurements before rollout; the wire caps are not performance targets.

## Upstream evidence

- Curio's [PDP HTTP routes](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/pdp/handlers.go#L210-L236) do not expose witnesses. The [PDPv0 proving task](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/tasks/pdpv0/task_prove.go#L582-L669) starts from dataset and piece IDs. Its [cached-proof primitive](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/lib/proof/cached_proof.go#L89-L184) already accepts a PieceCIDv2 and local leaf position, but reads the full snapshot layer and rebuilds the upper tree per call.
- [Upload publication](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/pdp/handlers_upload.go#L105-L121) marks staged PDP references above the cache threshold. The [PDPv0 cache scheduler](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/tasks/pdpv0/task_save_cache.go#L246-L271) claims those references without an AddPieces gate; its [worker](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/tasks/pdpv0/task_save_cache.go#L81-L160) verifies the snapshot against the requested commitment before marking completion. This is a preparation path, not a readiness or latency guarantee.
- [JWT authentication](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/pdp/auth.go#L34-L104) identifies a service; [AddPieces](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/pdp/handlers_add.go#L96-L106) filters staging references by service. The internal [`ReadForProof`](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/lib/cachedreader/hashspace_stage.go#L129-L164) can open restricted pieces by CID without caller context. HTTP [`/download`](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/market/retrieval/piecehandler.go#L128-L188) separately validates a Keysmith grant against the piece's FEE envelope. Use the latter auth boundary before returning leaf bytes, not the internal reader as authorization.
- PDP [resolves the piece and local offset before verification](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/PDPVerifier.sol#L1179-L1189). The [Merkle verifier](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/Proofs.sol#L15-L62) checks exact branch depth and position-dependent order; [PieceCID leaf-count logic](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/Cids.sol#L93-L104) excludes fully padded leaves from the modern verifier's challenge range. Confirm the target deployment separately.

## Review and acceptance

Agree with Curio on how service authority attaches to shared, staged and committed data, how the retention response reports readiness, and what provider limits the MSP can rely on. Keep the integration open until the following pass against the target Curio build and PDP verifier:

- Complete a client-facing proof round with only a parked Upload SP copy and no backend AddPieces submitted, including a large piece that requires a snapshot. Measure preparation from upload completion and request latency against the first applicable proof window, under concurrent upload and proof load. Include a cold process restart during preparation.
- Request the same positions after commitment and across concurrent staging cleanup or handoff. Verify branches for nonzero indices, piece boundaries and padding; reject wrong CID, index, depth and tampered nodes. Submit a proof built from the response to the target verifier.
- Exercise a valid private-grant request and reject a missing, invalid, or wrong-piece FOC-Grant even with a valid service JWT. Reject wrong JWT issuer, audience, scope, expiry, and cross-service access on warm and cold caches. A missing piece and another service's private piece have the same response.
- Exercise maximum-size batches, cache misses, request timeouts, and saturation. Confirm bounded reads and memory, cancellation, and fallback within the same proof-round deadline.
