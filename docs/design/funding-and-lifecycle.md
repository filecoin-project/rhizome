# Funding readiness and service lifecycle

Related: [architecture ownership/policy](../architecture.md#payments-lifecycle), [admission workflow](storage-workflows.md), and [execution/replay contract](execution-and-integrations.md).

## Ownership and accounting scope

Payments owns funding observations and readiness, and supplies them to Client service and Storage management. Client service applies admission policy; Storage management owns backend fulfillment. Transaction management executes their durable requests.

Observe each Filecoin Pay payer account at its payment-contract, token, and payer scope, retaining the chain reference. Datasets, rails, and operators can share the account's funds. FWSS operator allowances have their own token/payer/operator scope and aggregate usage. Account for competing uses of shared funds and allowances when assessing each admission.

Client-side and SP-side storage terms are separate commercial relationships. Assess MSP backend funding independently of client funding. Mixed client-side terms do not partition placement reuse; reuse can reduce backend costs while preserving the client's signed memberships and contract-enforced charges.

Coalesce client changes into current account/rail-discovery requests. Publish observations under account/rail-scoped ownership and chain-applicability checks. Retain each settlement intent and its actual receipt progress and amounts.

## Contract preflight

### Existing-dataset additions

FWSS `piecesAdded` validates authorization, updates payments from the resulting total leaf count, and may replenish lifecycle reserve. Payment failure reverts both the addition and nonce consumption. Simulate the exact intended operation at an identified chain view, with its sender, contracts, calldata, value, and authorization. This authoritative preflight covers rail/operator authority, settlement of current lockup, deposited funds for aggregate lockup, rate/lockup-allowance headroom, one-time-fee lockup, and applicable reserve replenishment.

The preflight result applies to that chain view and reserves no funds. Use it in place of a separate MSP balance calculation. The update path does not require a general operator-approval flag after rail creation.


### New dataset with initial pieces

Creation checks projected payer funds and operator approval/headroom, creates rails and initial lockup/reserve, then adds the initial pieces atomically. Preflight the combined operation. FWSS `352b614` selects immutable per-dataset storage terms (currency, precision, price); payment updates and scaled fees/reserves use those terms, and the pricing module supplies PDP lockup and lifecycle-reserve parameters. Terms-aware creation signing requires SDK/deployment integration.

Track ERC-20 deposit allowance, deposited Filecoin Pay funds and FWSS operator rate/lockup allowances separately. Clients maintain their own Filecoin Pay balances.

## Funding observations for ongoing policy

Use `getAccountInfoIfSettled(token, payer)` when a funding decision needs a projected account view. It returns:

| Field | Meaning / caution |
| --- | --- |
| `fundedUntilEpoch` | Epoch at which aggregate account lockup can no longer advance at the current rate, given current funds and lockup. |
| `currentFunds` | Funds held by the Filecoin Pay account. |
| `availableFunds` | Funds remaining after projected account lockup through the applicable epoch, shared across the account. |
| `currentLockupRate` | Current aggregate account rate; a proposed addition can increase it. |

These values describe the observed account rather than reserved capacity for a proposed addition. A zero-rate account returns the maximum-uint horizon sentinel even with no funds. `lockupLastSettledAt` can be stale because settlement is lazy. Payments retains observation scope, chain view/reference, time, and reason; choose freshness and polling from the decision budget. Exact preflight requires no separate account read unless another policy needs one.

## Admission gates and interruption semantics

| Condition | Before durable acceptance | After durable acceptance |
| --- | --- | --- |
| Exact preflight establishes a client funding/allowance failure | Reject through the normal API error path without creating an interest, accepted nonce claim, or client-facing operation. | Reconcile issued attempts. A conclusive admission failure withdraws only that admission's interests through the revision protocol. |
| Required client preflight is unavailable or cannot be interpreted reliably | Reject or defer for an unknown dependency result; do not classify it as confirmed client underfunding. | Preserve accepted intent and reconcile when the dependency returns. |
| Exact client preflight succeeds | Client funding eligibility passes. Apply the known-shortfall gates below using available information. | Reconcile actual execution and reorg outcomes under the durable request identity; preflight reserves no funding. |
| Original accepted request is replayed | Recover the admission before fresh-admission gates. | Return or recover its retained status and reuse its transactions and interests. |
| Available information shows insufficient MSP backend funding/allowances | Return a retryable admission failure, without extra reads or probes solely for this gate. | Preserve fulfillment and reconciliation until funding returns; the SP is not at fault. |
| Available information establishes fewer than three eligible SP destinations | Return a retryable admission failure, without discovery or probes solely for this gate. | Preserve accepted demand and continue placement planning and repair as eligibility changes. |
| Backend funding or three-SP capacity is unknown from available information | Admit if required validation, parking, Upload-SP selection, and other agreed gates pass. Retain the uncertainty without fetching information solely for this gate. | Discover execution needs and obstacles through normal fulfillment, assessment, and funding work; keep obligations durable. |

Preflight concerns the admission attempt. Ongoing client underfunding requires the still-unselected grace/hold, re-evaluation, and operator-approved termination policy; it does not automatically delete backend data or establish SP fault. Concurrent activity can consume shared funds and allowances after preflight. Request uniqueness and piece locks do not reserve them.

## Agreed admission policies

### Client funding

Use exact contract preflight as the sole client-side admission funding test.

### Known-shortfall gates for backend funding and provider capacity

Evaluate backend-funding and three-SP capacity gates solely from existing module views, ordinary observations and required-work evidence. Reject when sufficiently complete evidence establishes a shortfall; treat missing or partial evidence as unknown. Retain scope, attribution, freshness and completeness with observations.

Exact client preflight, suitable Upload-SP selection, and confirmation of [source availability](execution-and-integrations.md#mandatory-confirmation-retrieval-head) and [sufficient staging retention](execution-and-integrations.md#parking-refresh-and-retention) still apply. Continue normal assessment, monitoring, and backend-operation validation. Accepted obligations survive later shortfalls and uncertainty.

## Incoming-rail settlement

Payments persists the rail and requested target epoch, constructs `FilecoinPayV1.settleRail(railId, untilEpoch)` with zero value, and owns any semantic reduction of the target. Transaction management returns receipt/logs, actual included hash, and chain reference.

Decode logs from the expected payment contract for the expected rail:

| Receipt evidence | Payments interpretation |
| --- | --- |
| `RailSettled` | Record actual `settledUpTo`, gross amount, net payee credit, commission, and network fee. Compare actual progress with the target; successful settlement can be partial. Matching events suffice without an unconditional `getRail`. |
| `RailFinalized` | Record finalization. It can occur with or without `RailSettled` and does not establish historical payment amounts. |
| Success without `RailSettled` | A successful call may be a no-op. Use retained evidence or targeted state reads when needed; do not invent progress or amounts. |
| Revert/no progress | Another actor may have settled this permissionless rail, or the range may be unavailable. Payments decides whether to recheck or defer; Transaction management preserves the requested business target. |

Solidity return values are not receipt data. Read rail/proving state to choose a valid target or resolve ambiguous progress when existing evidence is insufficient; funding and termination decisions may independently require fresh state. Settlement cadence and gas trade-offs remain policy decisions.

## Contract exercise evidence

An isolated Anvil exercise compiled and deployed unmodified Filecoin Pay and used local fixtures for native-token account and rail logic. A positive deposit did not permit an unaffordable rate/lockup increase, and exhausted rate and lockup allowances reverted distinctly. Lazy settlement left stored time unchanged while projected available funds fell; a second rail shortened the shared account horizon. A zero-balance, zero-rate account returned the horizon sentinel, and an active-rail update failed after projected horizon expiry despite a positive deposit.

## Sources

- [FWSS addition/payment callback](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/modules/FWSSAuthorizationModule.sol)
- [FWSS payment update boundary](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/lib/LibStoragePayments.sol)
- [FWSS creation, reserve, and rate operations](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/lib/LibRails.sol)
- [Snapshot-specific price and lockup parameters](https://github.com/FilOzone/filecoin-services/blob/352b6146723759bd565b8f0d3606541ea792bc83/service_contracts/src/lib/PriceListUSDFC.sol)
- [Filecoin Pay funding, allowances, projections, and lockup enforcement](https://github.com/FilOzone/fws-payments/blob/f0a40fe287ecb08c2c20b828bdbadd2437988bba/src/FilecoinPayV1.sol)
