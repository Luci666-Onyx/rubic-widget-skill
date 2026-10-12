# Chapter 5: Status Tracking, Claims & Refunds

## Core Idea
After broadcasting, poll with `waitForStatus` (or `getStatusExtended` for one-shot checks) until a terminal status. `READY_TO_CLAIM` (Arbitrum Bridge) needs a manual on-chain claim; Celer refunds need a manual refund transaction.

## Key Concepts
- **getStatusExtended** — one-shot status fetch: `{ id, srcTxHash? }` → StatusResponseInterface
- **waitForStatus** — polling helper: interval 3s default, timeout 300s default, `onStatusUpdate` callback
- **srcTxHash requirement** — decentralized providers: `id` + `srcTxHash`; deposit-based providers: `id` only
- **READY_TO_CLAIM** — terminal; Arbitrum Bridge requires manual claim via `sdk.claim()`
- **claim** — builds the Arbitrum Bridge claim transaction (ETHEREUM ↔ ARBITRUM)
- **celerRefund** — builds the refund transaction for failed Celer Bridge swaps
- **Terminal statuses** — SUCCESS, READY_TO_CLAIM, FAIL, REVERT, REVERTED, NOT_FOUND, INDETERMINATE

## Transaction Statuses

| Status | Terminal | Meaning |
|---|---|---|
| `PENDING` | | Received, waiting for confirmation |
| `LONG_PENDING` | | Taking longer than expected, still processing |
| `WAITING_FOR_TRUSTLINE` | | XRP/Stellar trustline pending |
| `WAITING_FOR_REFUND_TRUSTLINE` | | Trustline needed before refund |
| `SUCCESS` | ✅ | Completed on destination chain |
| `READY_TO_CLAIM` | ✅ | Funds ready to claim (Arbitrum Bridge, manual) |
| `FAIL` | ✅ | Failed on destination chain |
| `REVERT` | ✅ | Reverted — collect funds |
| `REVERTED` | ✅ | Reverted (provider variant); refund may have been issued |
| `NOT_FOUND` | ✅ | Invalid id or too early |
| `INDETERMINATE` | ✅ | Status cannot be determined |

## Code Example — full status handling

```typescript
try {
    const finalStatus = await sdk.waitForStatus(
        { id: swapData.id, srcTxHash: tx.hash },
        { interval: 5000, timeout: 300_000 }
    );

    switch (finalStatus.status) {
        case 'SUCCESS':
            showSuccess(finalStatus.destinationTxHash!);
            break;
        case 'READY_TO_CLAIM':
            showClaimButton(finalStatus);
            break;
        case 'FAIL':
        case 'REVERT':
        case 'REVERTED':
            showError('Swap failed. Please try again.');
            break;
        case 'NOT_FOUND':
        case 'INDETERMINATE':
            showError('Status unknown. Check the explorer.');
            break;
    }
} catch (err) {
    if (err instanceof Error && err.message === 'Polling timeout') {
        showError('Swap is taking too long. Check the explorer manually.');
    } else {
        throw err;
    }
}
```

## Worked Example — READY_TO_CLAIM → claim()

```typescript
if (finalStatus.status === 'READY_TO_CLAIM') {
    const claimTx = await sdk.claim({
        sourceTransactionHash: tx.hash,
        fromBlockchain: 'ETHEREUM',
    });
    await signer.sendTransaction(claimTx);
}
```

## Worked Example — deposit trade polling (no srcTxHash)

```typescript
const depositData = await sdk.swapDepositTrade({ /* params */ });
const finalStatus = await sdk.waitForStatus(
    { id: depositData.id },    // srcTxHash not needed
    { interval: 10_000, timeout: 1_800_000 }
);
```

## REST status endpoint

```
GET https://api-v2.rubic.exchange/api/info/status?srcTxHash=0x...
→ { status: "SUCCESS", destinationTxHash: "0x...", destinationNetworkTitle: "POLYGON", destinationNetworkChainId: 137 }
```

## Refunds

| Provider | Refund model |
|---|---|
| Most providers | Automatic — track status until resolved |
| **Celer Bridge** | Manual — call the Celer refund endpoint / contact support |

## Key Takeaways
1. `waitForStatus` rejects with plain `Error('Polling timeout')` — NOT a RubicApiError — catch it separately
2. Deposit-based trades (ChangeNOW, Exolix): poll with `id` only
3. `READY_TO_CLAIM` is SUCCESS-adjacent but needs the user to sign a claim transaction — never auto-treat as done
4. Refunds are provider-specific: automatic for most, manual for Celer

## Anti-patterns
- **Treating PENDING as failed**: cross-chain routes legitimately run 1–30 minutes
- **Missing READY_TO_CLAIM**: the user's funds sit unclaimed unless you surface the claim flow
- **Polling deposit trades with srcTxHash**: they don't have one — pass only `id`

## Connects To
- **Ch04 Executing Swaps**: the tx hash this chapter polls on
- **Ch07 Error Handling**: polling timeout vs API error distinction
