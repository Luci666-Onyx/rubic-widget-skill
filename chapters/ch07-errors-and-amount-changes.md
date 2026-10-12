# Chapter 7: Error Handling & Amount Changes

## Core Idea
All API errors are 6-digit codes: `error domain` (3 digits) + `specific code` (3 digits). The SDK wraps them in `RubicApiError` (code, reason, data, traceId). Separately, quote-vs-swap amount drift is the INTEGRATOR's responsibility to detect (±0.5% threshold).

## Key Concepts
- **Error code format** — 6-digit = domain(3) + specific(3); e.g. domain 002 (calculation) + code 001 (no routes) = **2001**
- **RubicApiError** — typed subclass of Error: `name`, `message` (= reason), `code`, `reason`, `data`, `traceId`
- **Error domains** — 001 PARAMS_ERRORS, 002 CALCULATION_ERRORS, 003 SWAP_ERRORS, 004 RUBIC_CONTRACT_ERRORS, 999 UNKNOWN_ERRORS
- **AmountChangeWarning** — quote vs swap drift beyond ±0.5%; integrator must re-confirm with user
- **traceId** — internal request ID; include in bug reports
- **Polling timeout** — plain `Error('Polling timeout')`, NOT a RubicApiError

## Error Domains

| Domain | Code | Description |
|---|---|---|
| PARAMS_ERRORS | 001 | Request validation and input parameters |
| CALCULATION_ERRORS | 002 | Route discovery and quote calculation |
| SWAP_ERRORS | 003 | Swap execution |
| RUBIC_CONTRACT_ERRORS | 004 | Rubic smart contracts |
| UNKNOWN_ERRORS | 999 | Unexpected errors |

## Params errors (001)

| Code | Name | Description |
|---|---|---|
| 001 | REQUIRED_RECEIVER | Receiver address required |
| 002 | EQUAL_TOKENS | Source and destination tokens identical |
| 003 | DIFFERENT_QUOTES | Quote data mismatch |
| 004 | MISS_ID | Required identifier missing |
| 005 | NO_REQUIRED_FIELD | Missing required field(s) |
| 006 | NOT_CORRECT_WALLET_ADDRESS | Invalid wallet address |
| 999 | WRONG_OR_MISSED_FIELD | Invalid/malformed field |

## Calculation errors (002)

| Code | Name | Description |
|---|---|---|
| 001 | NO_ROUTES | No routes found for the params |
| 002 | BLOCKCHAIN_TEMPORARILY_DOWN | Blockchain unavailable |
| 003 | PROVIDER_RATE_LIMIT | Provider rate limit hit |
| 004 | MAX_AMOUNT | Amount exceeds provider limits |
| 005 | MIN_AMOUNT | Amount below provider limits |
| 006 | MAX_DECIMALS | Token decimals exceed limit |
| 007 | NO_AUTH_WALLET | Wallet authorization required |
| 999 | UNKNOWN | Unknown calculation error |

## Swap execution errors (003)

| Code | Name | Description |
|---|---|---|
| 001 | NEED_APPROVE | ERC20 approval required |
| 002 | NEED_PERMIT2_APPROVE | Permit2 approval required |
| 003 | NOT_ENOUGH_BALANCE | Insufficient token balance |
| 004 | NOT_ENOUGH_NATIVE_BALANCE | Insufficient native for gas |
| 005 | SIMULATION_FAILED | Tx simulation failed |
| 006 | UNSUPPORTED_RECEIVER | Receiver not supported |
| 007 | NO_DATA | Execution data missing |
| 100 | WRONG_ARB_BRIDGE_HASH | Invalid Arbitrum bridge tx hash |
| 999 | UNKNOWN | Unknown swap error |

## SDK error codes (common, from SDK docs)

| Code | Reason | Typical cause |
|---|---|---|
| 1001 | Insufficient liquidity | Amount too large for route |
| 1002 | Minimum amount not reached | Below provider minimum |
| 1003 | Trade expired | Quote `id` expired — re-quote |
| 1004 | Unsupported token pair | No provider supports the pair |
| 1101 | Invalid `fromAddress` | Wallet address format wrong |
| 1102 | Invalid `receiver` | Receiver format wrong |
| 1103 | Balance too low | Not enough tokens |
| 1104 | Insufficient gas | Not enough native token |
| 2001 | Invalid blockchain | Unsupported chain |
| 2002 | Invalid token address | Bad address for the chain |

> SDK docs mark these "illustrative" — always check `err.reason` for the exact message.

## Error response formats

- **Validation errors (PARAMS_ERRORS)**: may contain MULTIPLE issues — `{ code, reason }` array
- **Standard errors**: `{ error: { code, reason, data? }, id }` — the `id` lets you track partial executions

## Code Example — catching errors

```typescript
import { SDK, RubicApiError } from '@cryptorubic/sdk-lite';

try {
    const quote = await sdk.quoteBest({ /* ... */ });
} catch (err) {
    if (err instanceof RubicApiError) {
        console.error(`[${err.code}] ${err.reason}`);
        if (err.traceId) console.error('Trace ID:', err.traceId);
    } else {
        throw err; // network error, timeout
    }
}
```

## Worked Example — quote-expiry retry loop

```typescript
async function swapWithRetry(quoteParams, swapParams, maxRetries = 2) {
    for (let i = 0; i < maxRetries; i++) {
        try {
            const quote = await sdk.quoteBest(quoteParams);
            return await sdk.swap({ ...swapParams, id: quote.id });
        } catch (err) {
            if (err instanceof RubicApiError && err.code === 1003 && i < maxRetries - 1) {
                console.warn('Trade expired, re-quoting...');
                continue;
            }
            throw err;
        }
    }
}
```

## Worked Example — amount-change detection (±0.5%)

```typescript
const changePercent = 0.5;
const acceptablePercent = new BigNumber(changePercent).dividedBy(100);

const oldAmount = new BigNumber(oldWeiAmount);
const newAmount = new BigNumber(newWeiAmount);

const upperBound = oldAmount.multipliedBy(acceptablePercent.plus(1));
const lowerBound = oldAmount.multipliedBy(new BigNumber(1).minus(acceptablePercent));

const isOutOfRange = newAmount.lt(lowerBound) || newAmount.gt(upperBound);

if (isOutOfRange) {
    throw new AmountChangeWarning(oldWeiAmount, newWeiAmount);
}

// Client handling:
if (err instanceof AmountChangeWarning) {
    const rateChangeInfo = {
        oldAmount: Token.fromWei(err.oldAmount, trade.to.decimals),
        newAmount: Token.fromWei(err.newAmount, trade.to.decimals),
        tokenSymbol: trade.to.symbol,
    };
    const allowSwap = await onRateChange(rateChangeInfo); // user confirm UI
    if (allowSwap) { /* retry swap */ }
}
```

### Recommended amount-change flow

| Step | Action |
|---|---|
| 1 | Call quote, store toTokenAmount |
| 2 | User confirms, initiates swap |
| 3 | Call swap, receive updated toTokenAmount |
| 4 | Compare quote vs swap amount |
| 5 | If deviation > threshold (±0.5%), show confirmation UI |
| 6 | If approved, send a NEW swap request |
| 7 | If rejected, cancel execution |

## Why amounts change

| Reason | Description |
|---|---|
| Market movement | Prices change between quote and swap |
| Liquidity updates | DEX/bridge liquidity shifts |
| Provider recalculation | Providers recompute output during execution |

## Custom HTTP client error forwarding

```typescript
const myClient: HttpClient = {
    async post(url, body, options) {
        const res = await fetch(url, { method: 'POST', /* ... */ });
        const data = await res.json();
        if (!res.ok) throw RubicApiError.fromApiResponse(data);
        return data;
    },
    // ... get
};
```

## Key Takeaways
1. Decode errors as domain+code: `2001` = calculation domain + no-routes; `3001` = swap domain + need-approve
2. `err.data` carries context (spender address for 3001, gasLimit/gasPrice for 3004)
3. The API NEVER signals amount drift — the integrator compares quote vs swap amounts manually
4. A new swap request is required after user approval of a changed amount
5. Include `err.traceId` in any support ticket

## Anti-patterns
- **Auto-accepting changed amounts**: strongly discouraged; users must explicitly approve deviations > ±0.5%
- **Confusing Polling timeout with API errors**: one is a plain Error, the other is RubicApiError
- **Ignoring traceId**: fastest path to support resolution

## Connects To
- **Ch03/Ch04**: quote expiry (1003) and approval (3001) in context
- **Ch05 Status**: REVERT/READY_TO_CLAIM handling
