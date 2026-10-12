# Chapter 4: Executing Swaps

## Core Idea
`swap` builds `{ to, data, value }` calldata for a quoted route — the SDK NEVER broadcasts. `swapBest` combines quote+swap. Deposit trades return `depositAddress` + `amountToSend` for manual transfers.

## Key Concepts
- **swap** — requires quote `id` + `fromAddress` + `receiver`; returns SwapResponseInterface
- **swapBest** — same params minus `id`; API finds the best route internally
- **swapDepositTrade** — deposit-based; returns deposit address + amount (no source wallet tx)
- **SwapResponseInterface.transaction** — `{ to, data, value, approvalAddress? }` ready for the signer
- **approvalAddress** — the spender to approve ERC-20s for; differs per provider — always read from the swap response
- **enableChecks** — balance/gas validation before building; default `true`
- **signature / publicKey / refundAddress** — Retrobridge auth / Bitcoin trades / deposit refund address

## Execution Models (4 types)

| Execution type | Returned data | User action |
|---|---|---|
| EVM (decentralized) | `to` (contract), `data` (calldata), `value` (native) | Contract call via wallet |
| EVM (deposit) | `to` (token contract or deposit addr), `data` (transfer encoding or `0x`), `value` | Transfer to deposit address |
| non-EVM (decentralized) | Blockchain-native tx object (TON, Tron, Solana, Sui) | Sign & send with the chain's native wallet/SDK |
| non-EVM (deposit) | `depositAddress` + `amountToSend` | Manual transfer, track via identifiers |

## Swap Request Parameters (beyond quote params)

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✅ | Trade ID from quote response |
| `fromAddress` | string | ✅ | Wallet that signs and sends |
| `receiver` | string | ✅ | Recipient on destination chain |
| `enableChecks` | boolean | | Balance/gas checks; default `true` |
| `signature` | string | | Wallet auth signature (Retrobridge) |
| `publicKey` | string | | Bitcoin wallet public key |
| `refundAddress` | string | | Refund address for failed deposit trades |

## Code Example — full EVM flow

```typescript
// Step 1 — quote
const quote = await sdk.quoteBest({
    srcTokenBlockchain: 'ETH',
    srcTokenAddress: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', // USDC
    srcTokenAmount: '500',
    dstTokenBlockchain: 'ARBITRUM',
    dstTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // USDC on Arbitrum
    fromAddress: '0xYourWallet',
    receiver: '0xYourWallet',
});

// Step 2 — build tx
const swapData = await sdk.swap({
    srcTokenBlockchain: 'ETH',
    srcTokenAddress: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
    srcTokenAmount: '500',
    dstTokenBlockchain: 'ARBITRUM',
    dstTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831',
    fromAddress: '0xYourWallet',
    receiver: '0xYourWallet',
    id: quote.id,
});

// Step 3 — send (ethers.js example)
const tx = await signer.sendTransaction({
    to: swapData.transaction.to,
    data: swapData.transaction.data,
    value: swapData.transaction.value ?? '0x0',
});
```

## Worked Example — deposit trade response (TRX → SOL via near_intents)

```json
{
  "transaction": {
    "depositAddress": "TGvLb21GVMXdKoWDdRJA5TbXrGdKaMdBXL",
    "amountToSend": "50",
    "exchangeId": "TGvLb21GVMXdKoWDdRJA5TbXrGdKaMdBXL"
  },
  "useRubicContract": false
}
```
→ The user transfers 50 TRX to `depositAddress` manually; then poll status by `id` only (no srcTxHash).

## Worked Example — swapBest one-call (no quote preview)

```typescript
const swapData = await sdk.swapBest({
    srcTokenBlockchain: 'ETH',
    srcTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE',
    srcTokenAmount: '0.5',
    dstTokenBlockchain: 'BSC',
    dstTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE',
    fromAddress: '0xYourWallet',
    receiver: '0xYourWallet',
});
const tx = await signer.sendTransaction({
    to: swapData.transaction.to,
    data: swapData.transaction.data,
    value: swapData.transaction.value,
});
```

## Key Takeaways
1. `value: swapData.transaction.value ?? '0x0'` — native-token swaps carry value; ERC-20 swaps don't
2. Approve tokens against `swapData.transaction.approvalAddress` — NOT the router; it differs per provider
3. Deposit trades have no `srcTxHash` — status polling uses `id` only
4. Non-EVM decentralized routes return a native tx object — build/send with the chain's own SDK (see Ch05 for TON/Tron/Solana wallet snippets)

## Anti-patterns
- **Approving a hardcoded spender**: provider-specific `approvalAddress` — read it from every swap response
- **Assuming EVM calldata on non-EVM chains**: TON/Tron/Solana/Sui routes return native transaction objects
- **Sending tx with a stale quote `id`**: expired quotes fail with code 1003 — re-quote first

## Connects To
- **Ch03 Quoting**: where the quote `id` comes from
- **Ch05 Status Tracking & Claim**: what happens after the tx is sent
- **Ch07 Error Handling**: expired quotes and approval errors
