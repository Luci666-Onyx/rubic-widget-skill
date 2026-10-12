# Rubic Patterns

## Quote → Swap → Track (core pattern)

**When to use**: every EVM swap integration.
**How**:
```typescript
const quote = await sdk.quoteBest(params);
const swapData = await sdk.swap({ ...params, id: quote.id, fromAddress, receiver });
const tx = await signer.sendTransaction({
    to: swapData.transaction.to,
    data: swapData.transaction.data,
    value: swapData.transaction.value ?? '0x0',
});
const finalStatus = await sdk.waitForStatus({ id: swapData.id, srcTxHash: tx.hash });
```
**Trade-offs**: full control and a previewable quote, at the cost of an extra round-trip. Use `swapBest` when you don't need the preview.

## Quote-expiry retry

**When to use**: quotes can expire between preview and confirm (error 1003).
**How**:
```typescript
for (let i = 0; i < maxRetries; i++) {
    try {
        const quote = await sdk.quoteBest(quoteParams);
        return await sdk.swap({ ...swapParams, id: quote.id });
    } catch (err) {
        if (err instanceof RubicApiError && err.code === 1003 && i < maxRetries - 1) continue;
        throw err;
    }
}
```
**Trade-offs**: max 2 retries; beyond that the user's params likely changed.

## Amount-change guard (±0.5%)

**When to use**: always, before executing a swap built from a stale quote.
**How**: compare `destinationWeiAmount` from quote vs swap; if outside [0.995×, 1.005×], surface a confirmation UI and re-issue the swap on approval.
**Trade-offs**: adds a user step; skipping it risks executing a materially worse rate.

## Approval flow

**When to use**: any ERC-20 source token.
**How**:
```typescript
const approveInfo = await sdk.checkApprove({
    blockchain: 'ETH',
    tokenAddress: usdcAddress,
    walletAddress,
    spenderAddress: swapData.transaction.approvalAddress!,
    amount: '500',
});
if (approveInfo.needApprove) {
    const tx = await signer.sendTransaction({
        to: approveInfo.transaction!.to,
        data: approveInfo.transaction!.data,
    });
    await tx.wait();
}
```
**Trade-offs**: an extra tx + gas; batch it once (approve max) to amortize future swaps.

## Deposit trade (no source wallet)

**When to use**: CEXes, hardware wallets, Bitcoin/non-EVM sources, or users who can't sign on the source chain.
**How**:
```typescript
const result = await sdk.quoteDepositTrades({ /* params */ });
const swapData = await sdk.swapDepositTrade({ /* params */ });
// user sends swapData.transaction.amountToSend to swapData.transaction.depositAddress
const status = await sdk.waitForStatus({ id: swapData.id }); // id only — no srcTxHash
```
**Trade-offs**: slower (manual transfer); status polling uses `id` only.

## READY_TO_CLAIM flow (Arbitrum Bridge)

**When to use**: when status resolves as READY_TO_CLAIM.
**How**:
```typescript
if (status.status === 'READY_TO_CLAIM') {
    const claimTx = await sdk.claim({ sourceTransactionHash: tx.hash, fromBlockchain: 'ETHEREUM' });
    await signer.sendTransaction(claimTx);
}
```
**Trade-offs**: requires a second user signature; never auto-treat as complete.

## Live chain/provider validation

**When to use**: before every quote, instead of hardcoding chain ids.
**How**:
```typescript
const chains = await sdk.getChains();
const ok = chains.some(c => c.name === 'ARBITRUM') && chains.some(c => c.name === 'BSC');
const providers = await sdk.getProviders();
const acrossOk = providers.crossChain.find(p => p.name === 'across')?.chains
    .some(c => c.from === 'ETH' && c.to === 'ARBITRUM');
```
**Trade-offs**: one extra GET per session (cache it).

## MCP agent swap (full autonomy)

**When to use**: AI-agent-driven swaps via the Rubic MCP server.
**How**: `rubic_get_supported_chains` → `rubic_search_tokens` → `rubic_quote_routes` → `rubic_simulate_swap` → `rubic_build_swap_tx` → `rubic_sign_and_broadcast_tx` → `rubic_track_status`.
**Trade-offs**: signing requires `EVM_WALLET_PRIVATE_KEY` locally; EVM-only signing.

## Custom HTTP client (edge runtimes)

**When to use**: Cloudflare Workers, Deno, React Native — anywhere without Node's axios.
**How**: implement `HttpClient` with fetch; throw `RubicApiError.fromApiResponse(data)` on non-ok responses.
**Trade-offs**: you own error parsing; the SDK's automatic error wrapping is bypassed.

## Error-code decoding

**When to use**: whenever a RubicApiError surfaces.
**How**: split the 6-digit code — first 3 digits = domain (001 params / 002 calculation / 003 swap / 004 contract / 999 unknown), last 3 = specific code. `2001` = no routes; `3001` = need approve; `3004` = insufficient native; `1003` = quote expired.
**Trade-offs**: `err.reason` is always more precise — log both code and traceId.

## Provider filtering

**When to use**: when you need to exclude flaky/expensive providers.
**How**: `nativeBlacklist: ['celer_bridge']` for native providers; `foreignBlacklist: { lifi: ['stargate'] }` for foreign providers inside an aggregator; `preferredProvider` to force one.
**Trade-offs**: filtering shrinks the route set and can eliminate the best price.
