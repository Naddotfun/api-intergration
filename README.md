# Nad.fun API Integration Guide

This document describes the Nad.fun v2 API integration surface and the Nad.fun v2 token creation contract flow.

The API sections use the v2 response schema (`quote_*`, `version`, and `V2_*` market types). The contract sections are based on `nadfun-contract-v2`, especially `INadFunRouter` and `NadFunRouter`.

---

## Base URLs

| Network | Base URL |
|---|---|
| Mainnet | `https://api.nad.fun` |
| Testnet | `https://dev-api.nadapp.net` |

All examples use `{BASE_URL}` as a placeholder for one of the URLs above.

---

## Common Rules

### Address Format

- Token and account addresses use EVM address format.
- Addresses must be 42-character strings including the `0x` prefix.
- Invalid token addresses usually return `400` with an error such as `{"error":"Invalid token ID"}`.

### Numeric Values

- Prices, amounts, volumes, reserves, and supplies are returned as strings to preserve decimal precision.
- Integrators should parse these fields with decimal or bigint libraries, not JavaScript `number` for financial calculations.

### Error Format

```json
{
  "error": "error message"
}
```

Some internal errors are intentionally masked as `Internal server error`.

### API Key and Rate Limit

External requests can be sent without an API key, but the rate limit is lower.

| Request origin | `X-API-Key` | Rate limit |
|---|---:|---:|
| `nad.fun`, `nadapp.net`, `*.nad.fun`, `*.symphony.io`, `localhost:*` | Not required | None |
| External origin or no Origin header | Not provided | `10 req/min` |
| External origin or no Origin header | Provided | `100 req/min` |

```http
X-API-Key: nadfun_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Rate limited responses may include these headers:

| Header | Description |
|---|---|
| `X-RateLimit-Limit` | Allowed requests per minute |
| `X-RateLimit-Remaining` | Remaining requests in the current window |
| `X-RateLimit-Window` | Currently `1m` |
| `X-RateLimit-Upgrade` | Present when no API key is used |

---

## Authentication and API Key Issuance

API keys are issued from an authenticated wallet session. The full flow is:

1. Request a nonce from `POST /auth/nonce`.
2. Sign the returned nonce string with the wallet.
3. Create a session with `POST /auth/session`.
4. Use the session cookie to create an API key with `POST /api-key`.
5. Send the issued key in `X-API-Key` on API requests.

The full API key is returned only once at creation time. Store it securely.

### 1. Request Auth Nonce

#### `POST /auth/nonce`

```json
{
  "address": "0x1234567890abcdef1234567890abcdef12345678"
}
```

Success `200`:

```json
{
  "nonce": "Account:\n\n0x1234567890abcdef1234567890abcdef12345678\n\nURI: https://testnet.nad.fun\n\nVersion: 1\n\nChain ID: 10143\n\nNonce: 550e8400-e29b-41d4-a716-446655440000\n\nIssued At: 2026-05-08T00:00:00Z"
}
```

Sign the exact `nonce` string returned by the server. Do not edit or normalize whitespace.

### 2. Create Session

#### `POST /auth/session`

```json
{
  "signature": "0x1234567890abcdef...",
  "nonce": "Account:\n\n0x1234567890abcdef1234567890abcdef12345678\n\nURI: https://testnet.nad.fun\n\nVersion: 1\n\nChain ID: 10143\n\nNonce: 550e8400-e29b-41d4-a716-446655440000\n\nIssued At: 2026-05-08T00:00:00Z",
  "chain_id": 10143,
  "wallet_address": null
}
```

| Field | Required | Description |
|---|---:|---|
| `signature` | Yes | EVM signature of the exact nonce string |
| `nonce` | Yes | The full nonce message returned by `/auth/nonce` |
| `chain_id` | Yes | Must match the target API environment chain ID |
| `wallet_address` | No | Optional smart-wallet address field |

Success `200` sets an HTTP-only session cookie and returns account information:

```http
Set-Cookie: <SESSION_COOKIE_NAME>=<SESSION_ID>; HttpOnly; Secure; Path=/; SameSite=None; Max-Age=86400
```

```json
{
  "account_info": {
    "account_id": "0x1234567890abcdef1234567890abcdef12345678",
    "nickname": "0x1234...5678",
    "bio": "",
    "image_uri": "https://storage.nadapp.net/default/1.png"
  }
}
```

### 3. Create API Key

#### `POST /api-key`

Requires the session cookie created by `/auth/session`.

```json
{
  "name": "My Integration",
  "description": "External service integration",
  "expires_in_days": 365
}
```

| Field | Required | Description |
|---|---:|---|
| `name` | Yes | Human-readable API key name |
| `description` | No | Optional description |
| `expires_in_days` | No | Expiration in days. Omit or send `null` for no expiration |

Success `200`:

```json
{
  "id": 7185139933124608001,
  "api_key": "nadfun_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "key_prefix": "nadfun_xxxxx",
  "name": "My Integration"
}
```

Notes:

- API keys use the format `nadfun_` plus 32 alphanumeric characters.
- Each wallet account can create up to 5 API keys.
- The `owner_address` is taken from the authenticated session. Clients do not need to send it.
- The full `api_key` is not returned by list endpoints. It is only returned by this creation response.

### 4. Manage API Keys

#### `GET /api-key`

Lists API keys owned by the authenticated session account.

```json
{
  "api_keys": [
    {
      "id": 7185139933124608001,
      "key_prefix": "nadfun_xxxxx",
      "name": "My Integration",
      "description": "External service integration",
      "owner_address": "0x1234567890abcdef1234567890abcdef12345678",
      "is_active": true,
      "created_at": "2026-05-08T00:00:00Z",
      "expires_at": "2027-05-08T00:00:00Z",
      "last_used_at": null,
      "request_count": 0
    }
  ],
  "total": 1
}
```

#### `DELETE /api-key/:id`

Deletes an API key owned by the authenticated session account.

```json
{
  "success": true
}
```

### 5. Use API Key

```bash
curl "{BASE_URL}/trade/market/0x1234567890abcdef1234567890abcdef12345678" \
  -H "X-API-Key: nadfun_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

When the key is invalid, expired, inactive, or malformed, the API returns `401`.

### Auth and API Key Types

```ts
interface AuthNonceRequest {
  address: string;
}

interface AuthNonceResponse {
  nonce: string;
}

interface AuthSessionRequest {
  signature: string;
  nonce: string;
  chain_id: number;
  wallet_address?: string | null;
}

interface AuthSessionResponse {
  account_info: AccountInfo;
}

interface CreateApiKeyRequest {
  name: string;
  description?: string | null;
  owner_address?: string | null;
  expires_in_days?: number | null;
}

interface CreateApiKeyResponse {
  id: number;
  api_key: string;
  key_prefix: string;
  name: string;
}

interface ApiKeyInfo {
  id: number;
  key_prefix: string;
  name: string;
  description: string | null;
  owner_address: string | null;
  is_active: boolean;
  created_at: string;
  expires_at: string | null;
  last_used_at: string | null;
  request_count: number;
}

interface ApiKeyListResponse {
  api_keys: ApiKeyInfo[];
  total: number;
}
```

`owner_address` is accepted by the server-side request type, but API key creation overwrites it with the authenticated session address.

---

## Token Creation Flow

1. Upload the token image with `POST /metadata/image`.
2. Use the returned `image_uri` to upload token metadata with `POST /metadata/metadata`.
3. Use the returned `metadata_uri` to mine a vanity salt with `POST /token/salt`.
4. Call the v2 contract `NadFunRouter.create()` or `NadFunRouter.createWithNative()`.
5. After the creation transaction is indexed, query the token through `/token/:token`, `/token/metadata/:token_id`, and `/trade/*`.

---

## Common Response Types

### `AccountInfo`

```ts
interface AccountInfo {
  account_id: string;
  nickname: string;
  bio: string;
  image_uri: string;
}
```

### `TokenInfo`

V2 API shape for `token_info`.

```ts
type TokenVersion = "V1" | "V2";

interface TokenInfo {
  token_id: string;
  name: string;
  symbol: string;
  image_uri: string;
  description: string | null;
  is_graduated: boolean;
  is_nsfw: boolean;
  twitter: string | null;
  telegram: string | null;
  website: string | null;
  created_at: number;
  creator: AccountInfo;
  is_cto: boolean;
  version: TokenVersion;
}
```

If the creator has connected X, `creator.nickname` and `creator.image_uri` prefer the X handle and X image.

| Field | Type | Description |
|---|---|---|
| `token_id` | string | Token contract address |
| `name` | string | Token name |
| `symbol` | string | Token symbol |
| `image_uri` | string | Token image URL |
| `description` | string or null | Token description |
| `is_graduated` | boolean | Whether the token has graduated from bonding curve to DEX |
| `is_nsfw` | boolean | Whether the token image or metadata is marked NSFW |
| `twitter` | string or null | X URL |
| `telegram` | string or null | Telegram URL |
| `website` | string or null | Website URL |
| `created_at` | number | Creation Unix timestamp in seconds |
| `creator` | AccountInfo | Creator account information |
| `is_cto` | boolean | CTO status flag |
| `version` | `"V1"` or `"V2"` | Token version. V2 tokens support multi-quote markets |

### `MarketInfo`

V2 API shape for `market_info`.

```ts
type MarketType = "CURVE" | "DEX" | "V2_CURVE" | "V2_DEX";

interface MarketInfo {
  market_type: MarketType;
  token_id: string;
  quote_id: string;
  market_id: string;
  reserve_native: string;
  reserve_quote: string;
  reserve_token: string;
  token_price: string;
  native_price: string;
  quote_price: string;
  price: string;
  price_usd: string;
  price_native: string;
  total_supply: string;
  volume: string;
  ath_price: string;
  ath_price_usd: string;
  ath_price_native: string;
  holder_count: number;
}
```

| Field | Type | Description |
|---|---|---|
| `market_type` | `"CURVE"`, `"DEX"`, `"V2_CURVE"`, or `"V2_DEX"` | Current market type |
| `token_id` | string | Token contract address |
| `quote_id` | string | Quote asset address. V1 tokens use WMON |
| `market_id` | string | Curve or DEX pool address |
| `reserve_native` | string | Legacy reserve field. For V1 it is MON reserve; for V2 it mirrors quote reserve |
| `reserve_quote` | string | Quote asset reserve |
| `reserve_token` | string | Project token reserve |
| `token_price` | string | Token price in USD. Same value as `price_usd` in the V2 schema |
| `native_price` | string | Legacy price field. For V1 it is MON/USD; for V2 use `quote_price` |
| `quote_price` | string | Quote asset price in USD |
| `price` | string | Token price in quote asset |
| `price_usd` | string | Token price in USD |
| `price_native` | string | Legacy native price field. For V1 it matches `price`; for V2 use `price` with `quote_id` |
| `total_supply` | string | Total token supply |
| `volume` | string | Cumulative trading volume |
| `ath_price` | string | All-time high price in USD. Same value as `ath_price_usd` in the V2 schema |
| `ath_price_usd` | string | All-time high price in USD |
| `ath_price_native` | string | Legacy native ATH field. For V2, prefer quote-aware fields when available |
| `holder_count` | number | Token holder count |

### Token and Market Response Types

```ts
interface TokenResponse {
  token_info: TokenInfo;
}

interface TokenMetadataResponse {
  token_info: TokenInfo;
  market_info: MarketInfo;
}

interface MarketResponse {
  market_info: MarketInfo;
}
```

### `SwapInfo`

```ts
type SwapType = "BUY" | "SELL";

interface SwapInfo {
  event_type: SwapType;
  native_amount: string;
  quote_amount: string;
  token_amount: string;
  native_price: string;
  quote_price: string;
  value: string;
  transaction_hash: string;
  created_at: number;
}
```

| Field | Type | Description |
|---|---|---|
| `event_type` | `"BUY"` or `"SELL"` | Swap direction |
| `native_amount` | string | Legacy amount field. For V1 it is MON amount; for V2 it mirrors quote amount |
| `quote_amount` | string | Quote asset amount used in the swap |
| `token_amount` | string | Project token amount |
| `native_price` | string | Legacy price field. For V1 it is MON/USD; for V2 use `quote_price` |
| `quote_price` | string | Quote asset price in USD at execution time |
| `value` | string | Swap value in USD at execution time |
| `transaction_hash` | string | Transaction hash |
| `created_at` | number | Swap Unix timestamp in seconds |

### `BalanceInfo`

```ts
interface BalanceInfo {
  balance: string;
  token_price: string;
  native_price: string;
  created_at: number;
}
```

| Field | Type | Description |
|---|---|---|
| `balance` | string | Token balance |
| `token_price` | string | Token price in USD |
| `native_price` | string | Legacy native price field |
| `created_at` | number | Balance row creation Unix timestamp in seconds |

### Trading Response Types

```ts
interface TokenSwap {
  account_info: AccountInfo;
  swap_info: SwapInfo;
}

interface TokenSwapResponse {
  swaps: TokenSwap[];
  total_count: number;
}

interface TokenHolder {
  account_info: AccountInfo;
  balance_info: BalanceInfo;
}

interface TokenHolderResponse {
  holders: TokenHolder[];
  total_count: number;
}

interface BarResponse {
  k: string;
  t: number[];
  c: string[];
  o: string[];
  h: string[];
  l: string[];
  v: string[];
  s: string;
}

interface MetricItem {
  timeframe: string;
  percent: number;
  transactions: { buy: number; sell: number; total: number };
  volume: { buy: string; sell: string; total: string };
  makers: { buy: number; sell: number; total: number };
}

interface MetricsBatchResponse {
  metrics: MetricItem[];
}
```

### Upload and Salt Types

```ts
interface UploadImageResponse {
  is_nsfw: boolean;
  image_uri: string;
}

interface UploadMetadataRequest {
  image_uri: string;
  name: string;
  symbol: string;
  description?: string | null;
  website?: string | null;
  twitter?: string | null;
  telegram?: string | null;
}

interface TokenMetadata {
  name: string;
  symbol: string;
  description: string | null;
  image_uri: string;
  website: string | null;
  twitter: string | null;
  telegram: string | null;
  is_nsfw: boolean;
}

interface UploadMetadataResponse {
  metadata_uri: string;
  metadata: TokenMetadata;
}

interface MineSaltRequest {
  creator: string;
  name: string;
  symbol: string;
  metadata_uri: string;
}

interface MineSaltResponse {
  salt: string;
  address: string;
}
```

---

## 1. Get Token Info

### `GET /token/:token`

Returns token metadata and creator information.

#### Path Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `token` | string | Yes | Token contract address |

#### Success `200`

```json
{
  "token_info": {
    "token_id": "0x1234567890abcdef1234567890abcdef12345678",
    "name": "Sample Token",
    "symbol": "SAMPLE",
    "image_uri": "https://storage.nadapp.net/coin/uuid",
    "description": "Sample token description",
    "is_graduated": false,
    "is_nsfw": false,
    "twitter": "https://x.com/example",
    "telegram": "https://t.me/example",
    "website": "https://example.com",
    "created_at": 1754927984,
    "creator": {
      "account_id": "0xabcdef1234567890abcdef1234567890abcdef12",
      "nickname": "Creator",
      "bio": "Creator bio",
      "image_uri": "https://storage.nadapp.net/profile/uuid"
    },
    "is_cto": false,
    "version": "V2"
  }
}
```

---

## 2. Get Token Metadata and Market

### `GET /token/metadata/:token_id`

Returns token metadata and current market data in one response.

#### Path Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `token_id` | string | Yes | Token contract address |

#### Success `200`

```json
{
  "token_info": {
    "token_id": "0x1234567890abcdef1234567890abcdef12345678",
    "name": "Sample Token",
    "symbol": "SAMPLE",
    "image_uri": "https://storage.nadapp.net/coin/uuid",
    "description": "Sample token description",
    "is_graduated": false,
    "is_nsfw": false,
    "twitter": "https://x.com/example",
    "telegram": "https://t.me/example",
    "website": "https://example.com",
    "created_at": 1754927984,
    "creator": {
      "account_id": "0xabcdef1234567890abcdef1234567890abcdef12",
      "nickname": "Creator",
      "bio": "Creator bio",
      "image_uri": "https://storage.nadapp.net/profile/uuid"
    },
    "is_cto": false,
    "version": "V2"
  },
  "market_info": {
    "market_type": "V2_CURVE",
    "token_id": "0x1234567890abcdef1234567890abcdef12345678",
    "quote_id": "0x5a4E0bFDeF88C9032CB4d24338C5EB3d3870BfDd",
    "market_id": "0x27063a38eC0D3281D354090EB92e669Ed1eB956C",
    "reserve_native": "12345.67",
    "reserve_quote": "12345.67",
    "reserve_token": "987654321.0",
    "token_price": "0.00000028",
    "native_price": "1.25",
    "quote_price": "1.25",
    "price": "0.000000224",
    "price_usd": "0.00000028",
    "price_native": "0.000000224",
    "total_supply": "1000000000000000000000000000",
    "volume": "50000.0",
    "ath_price": "0.00000035",
    "ath_price_usd": "0.00000035",
    "ath_price_native": "0.00000028",
    "holder_count": 142
  }
}
```

---

## 3. Get Market Data

### `GET /trade/market/:token_id`

Returns current market state, price, reserves, volume, and holder count.

#### Path Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `token_id` | string | Yes | Token contract address |

#### Success `200`

```json
{
  "market_info": {
    "market_type": "V2_CURVE",
    "token_id": "0x1234567890abcdef1234567890abcdef12345678",
    "quote_id": "0x5a4E0bFDeF88C9032CB4d24338C5EB3d3870BfDd",
    "market_id": "0x27063a38eC0D3281D354090EB92e669Ed1eB956C",
    "reserve_native": "12345.67",
    "reserve_quote": "12345.67",
    "reserve_token": "987654321.0",
    "token_price": "0.00000028",
    "native_price": "1.25",
    "quote_price": "1.25",
    "price": "0.000000224",
    "price_usd": "0.00000028",
    "price_native": "0.000000224",
    "total_supply": "1000000000000000000000000000",
    "volume": "50000.0",
    "ath_price": "0.00000035",
    "ath_price_usd": "0.00000035",
    "ath_price_native": "0.00000028",
    "holder_count": 142
  }
}
```

---

## 4. Get Chart Data

### `GET /trade/chart/:token_id`

Returns OHLCV candlestick data.

#### Path Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `token_id` | string | Yes | Token contract address |

#### Query Parameters

| Name | Type | Required | Default | Description |
|---|---|---:|---|---|
| `resolution` | string | No | `5` | `1`, `5`, `15`, `30`, `60`, `1H`, `240`, `4H`, `D`, `1D`, `W`, `1W`, `M`, `1M` |
| `from` | integer | Yes | - | Start Unix timestamp. Current implementation validates it but uses `to` and `countback` for the query |
| `to` | integer | Yes | - | End Unix timestamp. Query condition is `time_stamp < to` |
| `countback` | integer | No | `500` | Maximum candles to return. Range: `1..3000` |
| `chart_type` | string | No | `price` | `price`, `price_usd`, `market_cap`, `market_cap_usd` |

#### Chart Types

| Value | Description |
|---|---|
| `price` | Token price in quote asset |
| `price_usd` | Token price in USD |
| `market_cap` | Market cap in quote asset |
| `market_cap_usd` | Market cap in USD |

#### Success `200`

```json
{
  "k": "price",
  "t": [1751460000, 1751460060, 1751460120],
  "c": ["0.00124", "0.00125", "0.00126"],
  "o": ["0.00123", "0.00124", "0.00125"],
  "h": ["0.00125", "0.00126", "0.00127"],
  "l": ["0.00122", "0.00123", "0.00124"],
  "v": ["1000.0", "1200.0", "1500.0"],
  "s": "ok"
}
```

#### No Data `200`

```json
{
  "k": "price",
  "t": [],
  "c": [],
  "o": [],
  "h": [],
  "l": [],
  "v": [],
  "s": "no_data"
}
```

---

## 5. Get Trading Metrics

### `GET /trade/metrics/:token_id`

Returns transaction counts, volume, maker counts, and price-change percentage for multiple timeframes.

#### Path Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `token_id` | string | Yes | Token contract address |

#### Query Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `timeframes` | string | Yes | Comma-separated values. Supported: `1`, `5`, `15`, `30`, `60`, `240`, `1D` |

#### Success `200`

```json
{
  "metrics": [
    {
      "timeframe": "1",
      "percent": 5.25,
      "transactions": {
        "buy": 150,
        "sell": 85,
        "total": 235
      },
      "volume": {
        "buy": "750000.50",
        "sell": "484567.39",
        "total": "1234567.89"
      },
      "makers": {
        "buy": 45,
        "sell": 32,
        "total": 77
      }
    }
  ]
}
```

#### Example

```bash
curl "{BASE_URL}/trade/metrics/0x1234567890abcdef1234567890abcdef12345678?timeframes=1,5,15,30,60,240,1D"
```

---

## 6. Get Swap History

### `GET /trade/swap-history/:token_id`

Returns paginated token swap history.

#### Path Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `token_id` | string | Yes | Token contract address |

#### Query Parameters

| Name | Type | Required | Default | Description |
|---|---|---:|---|---|
| `page` | integer | No | `1` | Minimum `1` |
| `limit` | integer | No | `10` | Range: `1..100` |
| `direction` | string | No | `DESC` | `ASC` or `DESC` |
| `volume_ranges` | string | No | - | `small` (`$0-$1000`), `medium` (`$1000-$10000`), `large` (`$10000+`), comma-separated |
| `account_id` | string | No | - | Filter swaps by account |
| `trade_type` | string | No | `ALL` | `BUY`, `SELL`, or `ALL` |

#### Success `200`

```json
{
  "swaps": [
    {
      "account_info": {
        "account_id": "0xabcdef1234567890abcdef1234567890abcdef12",
        "nickname": "Trader",
        "bio": "",
        "image_uri": "https://storage.nadapp.net/profile/uuid"
      },
      "swap_info": {
        "event_type": "BUY",
        "native_amount": "1000000000000000000",
        "quote_amount": "1000000000000000000",
        "token_amount": "500000000000000000000",
        "native_price": "1.25",
        "quote_price": "1.25",
        "value": "1.25",
        "transaction_hash": "0x...",
        "created_at": 1754927984
      }
    }
  ],
  "total_count": 100
}
```

---

## 7. Get Holders

### `GET /trade/holder/:token_id`

Returns a paginated token holder list.

#### Path Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `token_id` | string | Yes | Token contract address |

#### Query Parameters

| Name | Type | Required | Default | Description |
|---|---|---:|---|---|
| `page` | integer | No | `1` | Minimum `1` |
| `limit` | integer | No | `10` | Range: `1..100` |
| `direction` | string | No | `DESC` | Included in the shared pagination params |

#### Success `200`

```json
{
  "holders": [
    {
      "account_info": {
        "account_id": "0xabcdef1234567890abcdef1234567890abcdef12",
        "nickname": "Holder",
        "bio": "",
        "image_uri": "https://storage.nadapp.net/profile/uuid"
      },
      "balance_info": {
        "balance": "1000000000000000000000",
        "token_price": "0.00000028",
        "native_price": "1.25",
        "created_at": 1754927984
      }
    }
  ],
  "total_count": 150
}
```

---

## 8. Upload Image

### `POST /metadata/image`

Uploads a token image and checks whether it is NSFW.

#### Request

- Content-Type: `image/jpeg`, `image/png`, `image/webp`, or `image/svg+xml`
- Body: raw binary bytes
- File limit: `5MB`

The current implementation detects the actual image format from magic bytes. The `Content-Type` header is read, but format detection is based on file bytes.

#### Supported Formats

| Format | MIME |
|---|---|
| JPEG | `image/jpeg` |
| PNG | `image/png` |
| WebP | `image/webp` |
| SVG | `image/svg+xml` |

#### Success `200`

```json
{
  "is_nsfw": false,
  "image_uri": "https://storage.nadapp.net/coin/550e8400-e29b-41d4-a716-446655440000"
}
```

#### Example

```bash
curl -X POST "{BASE_URL}/metadata/image" \
  -H "Content-Type: image/png" \
  --data-binary @token.png
```

---

## 9. Upload Metadata

### `POST /metadata/metadata`

Stores token metadata JSON in R2 and the database. This endpoint must be called after `/metadata/image`.

#### Request

```json
{
  "image_uri": "https://storage.nadapp.net/coin/550e8400-e29b-41d4-a716-446655440000",
  "name": "Sample Token",
  "symbol": "SAMPLE",
  "description": "A sample token for demonstration purposes",
  "website": "https://example.com",
  "twitter": "https://x.com/example",
  "telegram": "https://t.me/example"
}
```

#### Validation

| Field | Required | Rule |
|---|---:|---|
| `image_uri` | Yes | Must start with `ALLOWED_IMAGE_DOMAIN`. Default: `https://storage.nadapp.net/` |
| `name` | Yes | Trimmed length 1-32, no newlines |
| `symbol` | Yes | 1-10 characters, ASCII alphanumeric only |
| `description` | No | Maximum 500 characters |
| `website` | No | If present, must start with `https://` |
| `twitter` | No | If present, must start with `https://x.com/` |
| `telegram` | No | If present, must start with `https://t.me/` |

Important: the NSFW result for `image_uri` is cached in Redis after image upload. If the cache expires, metadata upload fails. Call this endpoint soon after uploading the image.

#### Success `200`

```json
{
  "metadata_uri": "https://storage.nadapp.net/metadata/550e8400-e29b-41d4-a716-446655440000.json",
  "metadata": {
    "name": "Sample Token",
    "symbol": "SAMPLE",
    "description": "A sample token for demonstration purposes",
    "image_uri": "https://storage.nadapp.net/coin/550e8400-e29b-41d4-a716-446655440000",
    "website": "https://example.com",
    "twitter": "https://x.com/example",
    "telegram": "https://t.me/example",
    "is_nsfw": false
  }
}
```

---

## 10. Mine Salt

### `POST /token/salt`

Mines a `bytes32` salt so that the v2 token clone address ends with the configured vanity suffix.

#### Request

```json
{
  "creator": "0x742d35Cc6634C0532925a3b844Bc9e7595f70143",
  "name": "My Token",
  "symbol": "MTK",
  "metadata_uri": "https://storage.nadapp.net/metadata/94a412d2-b599-4bb0-b026-b14c4036c58c.json"
}
```

#### Validation

| Field | Required | Rule |
|---|---:|---|
| `creator` | Yes | EVM address |
| `name` | Yes | 1-32 characters, no newlines |
| `symbol` | Yes | 1-10 characters, alphanumeric |
| `metadata_uri` | Yes | Must start with `ALLOWED_IMAGE_DOMAIN` |

#### Success `200`

```json
{
  "salt": "0x000000000000000000000000000000000000000000000000000000000000a3f5",
  "address": "0x742d35Cc6634C0532925a3b844Bc9e7595f7777"
}
```

Failures use the common error format. The current runtime maps max-iteration failure to `500` with `{"error":"Internal server error"}`.

#### Implementation Notes

- API server environment variables:
  - `BONDING_CURVE`: CREATE2 deployer address
  - `TOKEN_IMPLEMENT`: EIP-1167 clone implementation address
  - `VANITY_ADDRESS_SUFFIX`: desired hex suffix. Current environment example: `7777`
- Maximum iterations: `10,000,000`
- Parallel mining chunk size: `10,000`
- CREATE2 address calculation uses EIP-1167 minimal proxy init code.
- The starting salt includes request values and a random UUID. The API does not guarantee the same salt for repeated calls with identical input.
- The returned `salt` produces a deterministic address when the deployer and implementation are unchanged.

---

## 11. v2 Contract Token Creation

The old `BondingCurveRouter.TokenCreationParams`, `amountOut`, and `actionId` flow is not used in v2. v2 uses `NadFunRouter` as the single user-facing router.

### Deployment Addresses

Based on `nadfun-contract-v2/deploy.md`:

| Name | Address |
|---|---|
| `WMON` | `0x5a4E0bFDeF88C9032CB4d24338C5EB3d3870BfDd` |
| `LV_MON` | `0xBe3fa50514D9617ce645a02B34F595541AF02b6b` |
| `V2_NAD_FUN_ROUTER` | `0x75588668999cA0557b78046b8a5E86b47b9234ec` |
| `V2_BONDING_CURVE` | `0x27063a38eC0D3281D354090EB92e669Ed1eB956C` |
| `V2_PROTOCOL_MANAGER` | `0x2F98030aBD7c59e3E5Dc6b4b66b6008821d0fB41` |
| `V2_TOKEN_REGISTRY` | `0x2Bc127be900aD290E703Cd2C71eB0EDCa162C898` |
| `V2_CREATOR_FEE_VAULT` | `0xfEB12B7698e296C57BBF9f0c9b38B3e908285A99` |
| `V2_BURN_VAULT` | `0xFA707fe7d2c2894bf0436c7B73947cBA9E888017` |
| `V2_LP_VAULT` | `0x2acD9C75fe16c909237D9e6f080210D26c8c956D` |
| `V2_GIFT_VAULT` | `0xC112EB5C40FC9A22425300D232A31d00FF840ad0` |

Addresses can differ by environment. Confirm the latest production addresses before using them in production.

### `INadFunRouter.CreateParams`

```solidity
struct CreateParams {
    string name;
    string symbol;
    string tokenURI;
    address quoteToken;
    uint16 creatorFeeRate;
    IBondingCurve.VaultAllocation[] vaults;
    bytes32 salt;
    ITokenRegistry.DexType dexType;
    uint256 buyQuoteAmount;
    uint256 deadline;
}
```

Field notes:

| Field | Notes |
|---|---|
| `quoteToken` | Must be registered in `ProtocolManager`. Examples: WMON, LVMON |
| `creatorFeeRate` | BPS. Default allowlist is `100`, `300`, `500`, representing 1%, 3%, 5% |
| `buyQuoteAmount` | Quote token amount used for the optional initial buy. Use `0` for no initial buy |
| `deadline` | Expiration Unix timestamp. Reverts if `deadline < block.timestamp` |

### `IBondingCurve.VaultAllocation`

```solidity
struct VaultAllocation {
    address vault;
    uint16 bps;
    bytes setupData;
}
```

- The total `bps` across vaults must be `10000`.
- Up to 5 vaults can be used.
- To send creator fees directly to a recipient, use `CreatorFeeVault` and set `setupData = abi.encode(recipient)`.

### `ITokenRegistry.DexType`

```solidity
enum DexType {
    UniswapV2,
    UniswapV3,
    UniswapV4
}
```

The current Nad.fun v2 pair path uses `UniswapV2`.

### Creation Functions

```solidity
function create(CreateParams calldata params)
    external
    returns (address token, uint256 tokenOut);

function createWithNative(CreateParams calldata params)
    external
    payable
    returns (address token, uint256 tokenOut);
```

### No Initial Buy

Set `buyQuoteAmount = 0`. The creator receives no initial buy tokens and `tokenOut = 0`.

```solidity
uint256 deployFee = protocolManager.deployFee(WMON);

IERC20(WMON).approve(address(nadFunRouter), deployFee);

IBondingCurve.VaultAllocation[] memory vaults = new IBondingCurve.VaultAllocation[](1);
vaults[0] = IBondingCurve.VaultAllocation({
    vault: V2_CREATOR_FEE_VAULT,
    bps: 10000,
    setupData: abi.encode(creator)
});

INadFunRouter.CreateParams memory params = INadFunRouter.CreateParams({
    name: "My Token",
    symbol: "MTK",
    tokenURI: metadataUri,
    quoteToken: WMON,
    creatorFeeRate: 500,
    vaults: vaults,
    salt: salt,
    dexType: ITokenRegistry.DexType.UniswapV2,
    buyQuoteAmount: 0,
    deadline: block.timestamp + 300
});

(address token, uint256 tokenOut) = nadFunRouter.create(params);
```

### Initial Buy

Set `buyQuoteAmount` to the quote token amount to spend on the initial buy. In v2, clients do not calculate and pass `amountOut` during creation. The router and bonding curve calculate `tokenOut` and return it.

```solidity
uint256 buyQuoteAmount = 1 ether;
uint256 deployFee = protocolManager.deployFee(WMON);
uint256 totalQuote = deployFee + buyQuoteAmount;

IERC20(WMON).approve(address(nadFunRouter), totalQuote);

INadFunRouter.CreateParams memory params = INadFunRouter.CreateParams({
    name: "My Token",
    symbol: "MTK",
    tokenURI: metadataUri,
    quoteToken: WMON,
    creatorFeeRate: 500,
    vaults: vaults,
    salt: salt,
    dexType: ITokenRegistry.DexType.UniswapV2,
    buyQuoteAmount: buyQuoteAmount,
    deadline: block.timestamp + 300
});

(address token, uint256 tokenOut) = nadFunRouter.create(params);
```

### Create with Native MON

Use `createWithNative` when `quoteToken` is WMON or LVMON and the user wants to fund creation with native MON.

```solidity
uint256 buyQuoteAmount = 1 ether;
uint256 deployFee = protocolManager.deployFee(WMON);
uint256 totalRequired = deployFee + buyQuoteAmount;

INadFunRouter.CreateParams memory params = INadFunRouter.CreateParams({
    name: "My Token",
    symbol: "MTK",
    tokenURI: metadataUri,
    quoteToken: WMON,
    creatorFeeRate: 500,
    vaults: vaults,
    salt: salt,
    dexType: ITokenRegistry.DexType.UniswapV2,
    buyQuoteAmount: buyQuoteAmount,
    deadline: block.timestamp + 300
});

(address token, uint256 tokenOut) =
    nadFunRouter.createWithNative{value: totalRequired}(params);
```

### v2 Changes Summary

| Old flow | v2 flow |
|---|---|
| `BondingCurveRouter` and `DexRouter` | Single `NadFunRouter` |
| Pass `amountOut` during token creation | Pass `buyQuoteAmount`; receive `tokenOut` |
| `actionId` | Removed |
| Single native quote assumption | `quoteToken` parameter supports multiple quote assets |
| Creator fee embedded in token transfer behavior | Plain ERC20 plus `FeeCollector` and vault distribution |
| Post-graduation external router dependency | `NadFunPair` plus `NadSwapAdapter` |

---

## 12. v2 Contract Trading Reference

`NadFunRouter` automatically routes by token lifecycle phase.

```solidity
function buy(BuyParams calldata params) external returns (uint256 amountOut);
function buyWithNative(BuyWithNativeParams calldata params) external payable returns (uint256 amountOut);
function buyWithPermit(BuyWithPermitParams calldata params) external returns (uint256 amountOut);

function sell(SellParams calldata params) external returns (uint256 amountOut);
function sellToNative(SellToNativeParams calldata params) external returns (uint256 amountOut);
function sellWithPermit(SellWithPermitParams calldata params) external returns (uint256 amountOut);
function sellToNativeWithPermit(SellToNativeWithPermitParams calldata params) external returns (uint256 amountOut);

function exactOutBuy(ExactOutBuyParams calldata params) external returns (uint256 amountIn);
function exactOutBuyWithNative(ExactOutBuyWithNativeParams calldata params) external payable returns (uint256 amountIn);
function exactOutSell(ExactOutSellParams calldata params) external returns (uint256 amountOut);
function exactOutSellToNative(ExactOutSellToNativeParams calldata params) external returns (uint256 amountOut);
```

View functions:

```solidity
function isGraduated(address token) external view returns (bool);
function getAmountOut(address token, uint256 amountIn, bool isBuy) external view returns (uint256);
function getAmountIn(address token, uint256 amountOut, bool isBuy) external view returns (uint256);
function getBondingCurveAmountOut(address token, uint256 amountIn, bool isBuy) external view returns (uint256);
function getBondingCurveAmountIn(address token, uint256 amountOut, bool isBuy) external view returns (uint256);
function getDexAmountOut(address token, uint256 amountIn, bool isBuy) external view returns (uint256);
function getDexAmountIn(address token, uint256 amountOut, bool isBuy) external view returns (uint256);
```

---

## Client Checklist

- Use the `image_uri` returned by the upload API when creating metadata.
- The current metadata request type allows `description` to be nullable, but product integrations should send a non-empty description.
- API keys are created through wallet session authentication and the full key is returned only once.
- The `address` returned by `/token/salt` is a predicted address before creation. After creation, verify the transaction receipt and indexed API data.
- In v2 token creation, do not pass `amountOut`. Pass `buyQuoteAmount` and use the returned `tokenOut`.
- Parse all price and amount strings with decimal-safe tooling.
- Strict parsers must handle v2 fields such as `quote_id`, `reserve_quote`, `quote_price`, `quote_amount`, `version`, `V2_CURVE`, and `V2_DEX`.
