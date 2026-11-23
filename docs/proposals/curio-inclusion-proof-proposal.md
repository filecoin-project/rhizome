# Piece inclusion proofs in Curio

Status: proposal for Curio review.

## Purpose

Serve Merkle witnesses against a PieceCID's CommP from staged or committed data. The first [client-facing PDP round](../architecture.md#proof-witnesses-and-client-facing-proving) must work with only a parked Upload SP copy, before backend AddPieces.

The MSP [resolves challenges to piece-relative leaf indices](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/PDPVerifier.sol#L1179-L1189), verifies witnesses, and submits the proof. Curio needs no dataset ID, backend piece ID, or challenge seed. A witness does not prove dataset membership or retention.

## HTTP contract

`POST /pdp/piece/{pieceCid}/inclusion-proofs` takes one PieceCIDv2 over HTTPS with `Authorization: Bearer …` and `Content-Type: application/json`. It creates no reference, retention lease, or proof job.

```json
{"leafIndices":[1]}
```

`leafIndices` contains 1–32 distinct integers in `[0, 2^h)`, where `h` is the CID's tree height in edges. Indices count 32-byte leaves in the full Fr32 tree, not raw bytes. Padding-only leaves are valid here but do not demonstrate possession; use the [verifier's challenge range](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/Cids.sol#L93-L104) for PDP.

`200 OK` returns all witnesses in request order. Each `leaf` is the Fr32 leaf itself; `siblings` has exactly `h` nodes, nearest first, excluding the root. Encode nodes as `0x` plus 64 lowercase hex digits. All responses use `Cache-Control: no-store`.

Example for `bafkzcibcaablqfyjsvd3rrmqmd7n4f7i73ybzuxfq4p4za6lft2jeaaaaguxefq` (raw bytes 0 through 126):

```json
{"witnesses":[{
  "leafIndex":1,
  "leaf":"0x8084888c9094989ca0a4a8acb0b4b8bcc0c4c8ccd0d4d8dce0e4e8ecf0f4f83c",
  "siblings":[
    "0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f",
    "0x8cb3cba6a9d5a5181ad2bbd6020329499585a4679d074800de21d1e3dbec8e24"
  ]
}]}
```

Curio verifies each branch before returning it. The MSP independently checks response size, count, order, indices, node widths, and branch depth, then [rebuilds the expected CommP](https://github.com/FilOzone/pdp/blob/4d6b30f95845ca5ef35f9430a3a3f435af54d291/src/Proofs.sol#L15-L62). Start at the leaf. At each level the index's low bit selects the node's side (0 left, 1 right); hash `left || right` with SHA-256, clear the top two bits of the last byte, and shift the index right. Reuse a verified witness for duplicate challenge slots.

| HTTP status | Meaning |
| --- | --- |
| `400` | Invalid input, including unsupported CID/height or invalid indices. |
| `401` | Missing or invalid witness credential. |
| `404` | No authorized live source; do not reveal another service's piece. |
| `413` | Request body too large. |
| `429` | Caller quota exhausted. |
| `503` | Temporary failure; distinguish `witness_not_ready`, capacity exhaustion, and read or verification failure. |

Errors return a JSON `code`, no witnesses or storage details. On fallback, keep the original round deadline and attribute the failure to its provider.

## Authorization

The MSP signs a short-lived JWT with a dedicated witness key. Curio maps each trusted issuer to a public key, signing algorithm, and service label. Require expiry, provider audience, witness-read scope, and the path's exact PieceCIDv2. Reject anonymous access. Witness and general PDP keys are not interchangeable; reuse verification code, not [`JWTAuth`'s trust policy](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/pdp/auth.go#L34-L104).

Require a live staged reference or readable committed placement authorized for the configured service and exact CID. Preserve that authorization through handoff; expiry or removal of the last qualifying source ends access. Shared bytes or caches confer no rights; a service may be authorized to read an existing copy without proving upload provenance.

For restricted data, also require a valid Keysmith read presentation in a separate `Authorization: FOC-Grant …` field, as in [Curio's private download handler](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/market/retrieval/piecehandler.go#L128-L188). Internal [`ReadForProof`](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/lib/cachedreader/hashspace_stage.go#L129-L164) and `open-pieces` placement are not authorization. Apply the same checks on warm and cold caches. Witnesses expose data bytes; do not log tokens, leaves, or branches.

## Staging readiness and bounded execution

Reuse Curio's [PDPv0 staged cache task](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/tasks/pdpv0/task_save_cache.go#L246-L271) for large pieces: it can run before AddPieces and [verifies the snapshot](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/tasks/pdpv0/task_save_cache.go#L81-L160). Propose `witnessReady` on the authenticated [parking retention response](../design/execution-and-integrations.md#parking-refresh-and-retention), true only when authorized reads and any required verified snapshot are usable. Require readiness and sufficient retention before admission at a witness-capable Upload SP. HEAD checks reachability and length only.

Build one tree for a small piece. For large pieces, adapt Curio's [cached proof primitive](https://github.com/filecoin-project/curio/blob/fea0f8680ede3a995c1c0c1073cc90a2a392e4a6/lib/proof/cached_proof.go#L89-L184) to reuse the upper tree and read each challenged section once per batch. Do not scan large pieces, unseal on demand, or fetch from another provider. Keep the source readable through cleanup or handoff, or fail the request.

Proposed wire caps: 4 KiB request, 128 KiB response, height at most 50. Independently bound storage/cache reads, memory, concurrency per service and globally, and elapsed time including queueing. The MSP caps each attempt to leave time for fallback and submission. Set operational limits from measurements before rollout.

## Review and acceptance

Agree on service access rules, readiness, and operating limits with Curio. Verify against the target Curio build and PDP verifier:

- Complete the first PDP round from a parked copy before backend AddPieces, including a large piece, restart during cache preparation, and concurrent upload/proof load. Measure preparation and witness latency.
- Verify identical witnesses after commitment and during cleanup or handoff; check boundary and padding positions, reject wrong CID/index/depth and tampering, and submit a proof to the target verifier.
- Reject credentials across key domains, invalid JWT claims, cross-service access, and missing or invalid private grants, including on warm caches. Confirm `404` does not disclose existence.
- Exercise maximum batches, cache misses, timeouts, and saturation. Check resource bounds, cancellation, and fallback within the original deadline.
