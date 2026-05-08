# Nad.fun Integration V2

이 문서는 현재 `api-server` 라우터/타입 기준의 외부 연동 문서입니다. 컨트랙트 생성 플로우는 `nadfun-contract-v2`의 `INadFunRouter` / `NadFunRouter` 기준으로 작성했습니다.

---

## Base URLs

| Network | Base URL |
|---|---|
| Mainnet | `https://api.nad.fun` |
| Testnet | `https://dev-api.nad.fun` |

모든 예시는 `{BASE_URL}`을 위 URL 중 하나로 치환해서 사용합니다.

---

## 공통 규칙

### 주소 형식

- 모든 토큰/계정 주소는 EVM 주소 형식입니다.
- `0x` prefix 포함 42자 문자열이어야 합니다.
- 유효하지 않은 주소는 대부분 `400`과 `{"error":"Invalid token ID"}` 또는 유사 메시지를 반환합니다.

### 숫자 형식

- 가격, 수량, 볼륨, 공급량처럼 정밀도가 필요한 값은 문자열로 반환됩니다.
- 클라이언트에서는 `number` 대신 decimal/bigint 라이브러리 사용을 권장합니다.

### 에러 형식

```json
{
  "error": "error message"
}
```

일부 내부 오류는 상세 사유 대신 `Internal server error`로 마스킹됩니다.

### API Key 및 Rate Limit

외부 Origin 요청은 API Key 없이도 가능하지만 Rate Limit이 낮습니다.

| 요청 출처 | `X-API-Key` | Rate Limit |
|---|---:|---:|
| `nad.fun`, `nadapp.net`, `*.nad.fun`, `*.symphony.io`, `localhost:*` | 불필요 | 없음 |
| 외부 Origin / Origin 없음 | 없음 | `10 req/min` |
| 외부 Origin / Origin 없음 | 있음 | `100 req/min` |

```http
X-API-Key: nadfun_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Rate limit 응답에는 다음 헤더가 포함될 수 있습니다.

| Header | Description |
|---|---|
| `X-RateLimit-Limit` | 분당 허용 요청 수 |
| `X-RateLimit-Remaining` | 남은 요청 수 |
| `X-RateLimit-Window` | 현재 `1m` |
| `X-RateLimit-Upgrade` | API Key 미사용 시 안내 |

---

## 전체 생성 플로우

1. `POST /metadata/image`로 토큰 이미지를 업로드합니다.
2. 반환된 `image_uri`로 `POST /metadata/metadata`를 호출해 `metadata_uri`를 받습니다.
3. `POST /token/salt`로 vanity 주소용 `salt`와 예상 토큰 주소를 받습니다.
4. v2 컨트랙트 `NadFunRouter.create()` 또는 `NadFunRouter.createWithNative()`를 호출합니다.
5. 생성 트랜잭션이 인덱싱된 뒤 API에서 `/token/:token`, `/token/metadata/:token_id`, `/trade/*` 조회가 가능합니다.

---

## 공통 응답 타입

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

현재 API 응답의 `token_info` 기준입니다.

```ts
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
  hackathon_info?: HackathonInfo | null;
}
```

`creator.nickname` / `creator.image_uri`는 X 연동 정보가 있으면 X 정보를 우선 사용합니다.

| Field | Type | Description |
|---|---|---|
| `token_id` | string | 토큰 컨트랙트 주소 |
| `name` | string | 토큰 이름 |
| `symbol` | string | 토큰 심볼 |
| `image_uri` | string | 토큰 이미지 URL |
| `description` | string \| null | 토큰 설명 |
| `is_graduated` | boolean | bonding curve에서 DEX로 졸업했는지 여부 |
| `is_nsfw` | boolean | NSFW 이미지/메타데이터 여부 |
| `twitter` | string \| null | X 링크 |
| `telegram` | string \| null | Telegram 링크 |
| `website` | string \| null | 웹사이트 링크 |
| `created_at` | number | 생성 Unix timestamp, seconds |
| `creator` | AccountInfo | creator 계정 정보 |
| `is_cto` | boolean | CTO 상태 여부 |
| `hackathon_info` | HackathonInfo \| null | 해커톤 등록 정보. `/token/:token`에서 포함될 수 있음 |

### `MarketInfo`

현재 API 응답의 `market_info` 기준입니다.

```ts
type MarketType = "CURVE" | "DEX";

interface MarketInfo {
  market_type: MarketType;
  token_id: string;
  market_id: string;
  reserve_native: string;
  reserve_token: string;
  token_price: string;
  native_price: string;
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
| `market_type` | `"CURVE"` \| `"DEX"` | 현재 마켓 타입 |
| `token_id` | string | 토큰 컨트랙트 주소 |
| `market_id` | string | curve 또는 DEX pool 주소 |
| `reserve_native` | string | 현재 API 호환 필드. quote/native 리저브 |
| `reserve_token` | string | 프로젝트 토큰 리저브 |
| `token_price` | string | 토큰 USD 가격 |
| `native_price` | string | 현재 API 호환 필드. quote/native USD 가격 |
| `price` | string | quote/native 기준 토큰 가격 |
| `price_usd` | string | USD 기준 토큰 가격 |
| `price_native` | string | `price`와 동일한 호환 필드 |
| `total_supply` | string | 총 공급량 |
| `volume` | string | 누적 거래량 |
| `ath_price` | string | ATH 가격. 현재 API에서는 USD 기준 값 |
| `ath_price_usd` | string | ATH USD 가격 |
| `ath_price_native` | string | ATH quote/native 가격 |
| `holder_count` | number | 홀더 수 |

주의: v2 컨트랙트는 multi quote asset을 지원하지만, 현재 이 API 타입은 `quote_id`, `reserve_quote`, `quote_price`, `version`을 직렬화하지 않습니다. 현재 코드 기준 `market_type`도 `CURVE` 또는 `DEX`로 반환됩니다.

---

## 1. 토큰 기본 정보 조회

### `GET /token/:token`

토큰의 기본 메타데이터와 creator 정보를 조회합니다.

#### Path Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `token` | string | Yes | 토큰 컨트랙트 주소 |

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
    "hackathon_info": null
  }
}
```

---

## 2. 토큰 메타데이터 + 마켓 조회

### `GET /token/metadata/:token_id`

토큰 기본 정보와 현재 마켓 정보를 함께 조회합니다.

#### Path Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `token_id` | string | Yes | 토큰 컨트랙트 주소 |

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
    "hackathon_info": null
  },
  "market_info": {
    "market_type": "CURVE",
    "token_id": "0x1234567890abcdef1234567890abcdef12345678",
    "market_id": "0x27063a38eC0D3281D354090EB92e669Ed1eB956C",
    "reserve_native": "12345.67",
    "reserve_token": "987654321.0",
    "token_price": "0.00000028",
    "native_price": "1.25",
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

## 3. 마켓 데이터 조회

### `GET /trade/market/:token_id`

현재 마켓 상태, 가격, 리저브, 볼륨, 홀더 수를 조회합니다.

#### Path Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `token_id` | string | Yes | 토큰 컨트랙트 주소 |

#### Success `200`

```json
{
  "market_info": {
    "market_type": "CURVE",
    "token_id": "0x1234567890abcdef1234567890abcdef12345678",
    "market_id": "0x27063a38eC0D3281D354090EB92e669Ed1eB956C",
    "reserve_native": "12345.67",
    "reserve_token": "987654321.0",
    "token_price": "0.00000028",
    "native_price": "1.25",
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

## 4. 차트 데이터 조회

### `GET /trade/chart/:token_id`

OHLCV 캔들 데이터를 조회합니다.

#### Path Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `token_id` | string | Yes | 토큰 컨트랙트 주소 |

#### Query Parameters

| Name | Type | Required | Default | Description |
|---|---|---:|---|---|
| `resolution` | string | No | `5` | `1`, `5`, `15`, `30`, `60`, `1H`, `240`, `4H`, `D`, `1D`, `W`, `1W`, `M`, `1M` |
| `from` | integer | Yes | - | 시작 Unix timestamp. 현재 구현에서는 유효성 검사용이며 조회 조건에는 `to`/`countback`이 사용됩니다. |
| `to` | integer | Yes | - | 종료 Unix timestamp. `time_stamp < to` 조건 |
| `countback` | integer | No | `500` | 반환할 최대 캔들 수. `1..3000` |
| `chart_type` | string | No | `price` | `price`, `price_usd`, `market_cap`, `market_cap_usd` |

#### Chart Type

| Value | Description |
|---|---|
| `price` | quote token 기준 토큰 가격 |
| `price_usd` | USD 기준 토큰 가격 |
| `market_cap` | quote token 기준 시가총액 |
| `market_cap_usd` | USD 기준 시가총액 |

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

## 5. 거래 지표 조회

### `GET /trade/metrics/:token_id`

여러 timeframe의 거래 수, 거래량, maker 수, 가격 변동률을 한 번에 조회합니다.

#### Path Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `token_id` | string | Yes | 토큰 컨트랙트 주소 |

#### Query Parameters

| Name | Type | Required | Description |
|---|---|---:|---|
| `timeframes` | string | Yes | comma-separated. `1`, `5`, `15`, `30`, `60`, `240`, `1D` |

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

## 6. Swap History 조회

### `GET /trade/swap-history/:token_id`

토큰 거래 내역을 페이지네이션으로 조회합니다.

#### Query Parameters

| Name | Type | Required | Default | Description |
|---|---|---:|---|---|
| `page` | integer | No | `1` | 최소 `1` |
| `limit` | integer | No | `10` | `1..100` |
| `direction` | string | No | `DESC` | `ASC` 또는 `DESC` |
| `volume_ranges` | string | No | - | `small`, `medium`, `large`, comma-separated |
| `account_id` | string | No | - | 특정 계정 거래 필터 |
| `trade_type` | string | No | `ALL` | `BUY`, `SELL`, `ALL` |

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
        "token_amount": "500000000000000000000",
        "native_price": "1.25",
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

## 7. Holder 조회

### `GET /trade/holder/:token_id`

토큰 홀더 목록을 조회합니다.

#### Query Parameters

| Name | Type | Required | Default | Description |
|---|---|---:|---|---|
| `page` | integer | No | `1` | 최소 `1` |
| `limit` | integer | No | `10` | `1..100` |
| `direction` | string | No | `DESC` | 현재 PaginationParams에 포함 |

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

## 8. 이미지 업로드

### `POST /metadata/image`

토큰 이미지를 업로드하고 NSFW 여부를 검사합니다.

#### Request

- Content-Type: `image/jpeg`, `image/png`, `image/webp`, `image/svg+xml`
- Body: raw binary bytes
- File limit: `5MB`

현재 구현은 실제 포맷을 magic bytes로 판별합니다. Content-Type은 읽지만 포맷 판별 자체는 파일 바이트 기준입니다.

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

## 9. 메타데이터 업로드

### `POST /metadata/metadata`

이미지 업로드 결과를 사용해 토큰 메타데이터 JSON을 R2와 DB에 저장합니다.

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
| `image_uri` | Yes | `ALLOWED_IMAGE_DOMAIN`으로 시작. 기본 `https://storage.nadapp.net/` |
| `name` | Yes | trim 후 1-32자, 줄바꿈 불가 |
| `symbol` | Yes | 1-10자, ASCII alphanumeric only |
| `description` | No | 최대 500자 |
| `website` | No | 값이 있으면 `https://`로 시작 |
| `twitter` | No | 값이 있으면 `https://x.com/`로 시작 |
| `telegram` | No | 값이 있으면 `https://t.me/`로 시작 |

중요: `image_uri`의 NSFW 결과는 이미지 업로드 후 Redis에 캐시됩니다. 캐시가 만료되면 메타데이터 업로드가 실패하므로 이미지 업로드 직후 호출해야 합니다.

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

## 10. Salt 마이닝

### `POST /token/salt`

v2 token clone 주소가 특정 suffix로 끝나도록 `bytes32 salt`를 찾습니다.

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
| `name` | Yes | 1-32자, 줄바꿈 불가 |
| `symbol` | Yes | 1-10자, alphanumeric |
| `metadata_uri` | Yes | `ALLOWED_IMAGE_DOMAIN`으로 시작 |

#### Success `200`

```json
{
  "salt": "0x000000000000000000000000000000000000000000000000000000000000a3f5",
  "address": "0x742d35Cc6634C0532925a3b844Bc9e7595f7777"
}
```

#### 구현 기준

- API 서버 환경변수:
  - `BONDING_CURVE`: CREATE2 deployer로 사용
  - `TOKEN_IMPLEMENT`: EIP-1167 clone implementation 주소
  - `VANITY_ADDRESS_SUFFIX`: 원하는 hex suffix. 현재 환경 예시는 `7777`
- 최대 반복 횟수: `10,000,000`
- 병렬 탐색 chunk size: `10,000`
- CREATE2 주소는 EIP-1167 minimal proxy init code 기준으로 계산됩니다.
- 마이닝 시작 salt에는 요청값과 랜덤 UUID가 섞입니다. 따라서 같은 입력으로 호출해도 API가 매번 같은 salt를 반환한다는 보장은 없습니다.
- 반환된 `salt` 자체와 컨트랙트 deployer/implementation이 같으면 결과 주소는 deterministic입니다.

---

## 11. v2 컨트랙트 생성 연동

기존 문서의 `BondingCurveRouter.TokenCreationParams` / `amountOut` / `actionId` 방식은 v2에서 사용하지 않습니다. v2는 `NadFunRouter`가 단일 user-facing router입니다.

### 주요 배포 주소

`nadfun-contract-v2/deploy.md` 기준:

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

환경별 주소가 다를 수 있으므로 production integration에서는 최신 배포 주소를 별도 확인해야 합니다.

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

필드 주의사항:

| Field | Notes |
|---|---|
| `quoteToken` | `ProtocolManager`에 등록된 quote token이어야 합니다. 예: WMON, LVMON |
| `creatorFeeRate` | BPS 단위입니다. 기본 allowlist는 1%, 3%, 5%에 해당하는 `100`, `300`, `500`입니다. |
| `buyQuoteAmount` | 초기 매수에 사용할 quote token 수량입니다. 초기 매수가 없으면 `0` |
| `deadline` | 만료 Unix timestamp입니다. `deadline < block.timestamp`이면 revert |

### `IBondingCurve.VaultAllocation`

```solidity
struct VaultAllocation {
    address vault;
    uint16 bps;
    bytes setupData;
}
```

- `bps` 합계는 `10000`이어야 합니다.
- 최대 5개 vault를 사용할 수 있습니다.
- creator fee를 단순 수령 주소로 보내려면 `CreatorFeeVault`를 쓰고 `setupData = abi.encode(recipient)`를 전달합니다.

### `ITokenRegistry.DexType`

```solidity
enum DexType {
    UniswapV2,
    UniswapV3,
    UniswapV4
}
```

현재 v2 NadFun pair 경로는 `UniswapV2` 타입을 사용합니다.

### 생성 함수

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

`buyQuoteAmount = 0`으로 생성합니다. 이 경우 creator는 초기 매수 토큰을 받지 않고 `tokenOut = 0`입니다.

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

`buyQuoteAmount`에 초기 매수에 사용할 quote token 수량을 넣습니다. v2에서는 `amountOut`을 클라이언트가 직접 계산해 넣지 않습니다. 라우터/본딩커브가 `tokenOut`을 계산해 반환합니다.

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

### Native MON으로 생성

`quoteToken`이 WMON 또는 LVMON이고 native MON을 사용하려면 `createWithNative`를 호출합니다.

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

### v2 변경 요약

| 기존 방식 | v2 방식 |
|---|---|
| `BondingCurveRouter` / `DexRouter` | `NadFunRouter` 단일 라우터 |
| `amountOut`을 생성 파라미터로 입력 | `buyQuoteAmount` 입력, `tokenOut` 반환 |
| `actionId` | 제거 |
| 단일 native quote 가정 | `quoteToken` 파라미터로 multi quote 지원 |
| creator fee 로직이 token transfer에 내장 | plain ERC20 + `FeeCollector` + vault distribution |
| DEX 졸업 후 외부 router 의존 | `NadFunPair` + `NadSwapAdapter` 경로 |

---

## 12. v2 거래 컨트랙트 참고

`NadFunRouter`는 토큰의 lifecycle에 따라 자동 라우팅합니다.

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

조회 함수:

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

## 클라이언트 체크리스트

- `metadata.image_uri`가 아니라 업로드 API의 `image_uri`를 그대로 사용합니다.
- `/metadata/metadata`의 `description`은 현재 API 타입상 nullable이지만, 제품 정책상 비어 있지 않게 보내는 것을 권장합니다.
- `/token/salt` 결과의 `address`는 생성 전 예상 주소입니다. 최종 생성 후에는 트랜잭션 receipt와 API 인덱싱 결과를 함께 확인합니다.
- v2 생성 시 `amountOut`을 파라미터로 넣지 않습니다. `buyQuoteAmount`만 넣고 `tokenOut` 반환값을 사용합니다.
- 모든 금액/가격 문자열은 decimal로 파싱합니다.
- strict enum parser를 쓰는 클라이언트는 향후 API가 `quote_*` / v2 market type 필드를 추가해도 깨지지 않게 unknown field와 enum 확장을 허용하는 구조를 권장합니다.
