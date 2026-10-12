# Chapter 2: Configuration

## Core Idea
`SDK.create(SdkParams, httpClient?)` — set referrer (required), apiKey, integrator addresses, timeout; swap the default axios client for fetch or a custom axios instance.

## Key Concepts
- **SdkParams.referrer** (required) — sent as `referer` in every request body
- **SdkParams.apiKey** — default `''`; higher rate limits; from Rubic BD
- **integratorAddress.crossChain** — default `0x3fFF...DbEE`; fee recipient for cross-chain
- **integratorAddress.onChain** — default `0x3b9C...0d4`; fee recipient for on-chain
- **timeout** — default `10000` ms (default HTTP client only)
- **HttpClient interface** — `{ get, post }` — replaceable for Workers/Deno/RN

## Reference Tables

### SdkParams

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `referrer` | string | ✅ | — | Integration identifier, sent as `referer` in every request |
| `apiKey` | string | | `''` | API key from Rubic BD |
| `integratorAddress.crossChain` | string | | `0x3fFF...DbE` | Fee recipient for cross-chain swaps |
| `integratorAddress.onChain` | string | | `0x3b9C...0d4` | Fee recipient for on-chain swaps |
| `timeout` | number | | `10000` | Request timeout in ms (default client only) |

### HttpClient interface

```typescript
interface HttpClient {
    get<T>(url: string, options?: {
        headers?: Record<string, string>;
        params?: Record<string, string | number | boolean | ReadonlyArray<string | number | boolean>>;
    }): Promise<T>;
    post<T>(url: string, body: object, options?: {
        headers?: Record<string, string>;
    }): Promise<T>;
}
```

## Code Examples

### Minimal (no API key)

```typescript
const sdk = await SDK.create({ referrer: 'my-dapp' });
```

### Native fetch client (Cloudflare Workers / Deno / React Native)

```typescript
const fetchClient: HttpClient = {
    async get(url, options = {}) {
        const searchParams = new URLSearchParams(
            Object.entries(options.params ?? {}).map(([k, v]) => [k, String(v)])
        );
        const fullUrl = searchParams.size ? `${url}?${searchParams}` : url;
        const res = await fetch(fullUrl, { headers: options.headers });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
    },
    async post(url, body, options = {}) {
        const res = await fetch(url, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json', ...options.headers },
            body: JSON.stringify(body),
        });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
    },
};
const sdk = await SDK.create({ referrer: 'my-app' }, fetchClient);
```

### Custom axios instance (interceptors/retries)

```typescript
const axiosInstance = axios.create({ timeout: 20_000 });
axiosInstance.interceptors.response.use(
    res => res.data,
    err => Promise.reject(err)
);
const sdk = await SDK.create({ referrer: 'my-app' }, axiosInstance);
```

## Key Takeaways
1. With a custom HTTP client, YOU parse API errors — use `RubicApiError.fromApiResponse(data)` on non-ok responses (see Ch07)
2. A custom axios instance MUST unwrap `res.data` in an interceptor, or the SDK receives the full axios response object
3. `apiKey` comes from Rubic BD (Telegram `t.me/RubicPartnership`) — not self-serve
4. Integrator fee addresses must be whitelisted by Rubic before they work

## Anti-patterns
- **Leaving default integrator addresses in production**: without your whitelisted address, the default fixed fee in native tokens is deducted from every transaction
- **Using axios interceptors that return the whole response** — breaks the SDK silently

## Connects To
- **Ch01 Getting Started**: where SDK.create fits in the flow
- **Ch08 Monetization**: how integrator fees actually pay out
