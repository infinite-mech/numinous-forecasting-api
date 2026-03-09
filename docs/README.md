# Numinous Forecasts API

## Overview

The Numinous Forecasts API enables programmatic access to Numinous's probabilistic forecasting system. Given a question about a future event, the API returns a probability estimate between 0 and 1 reflecting the likelihood that the event resolves positively.

Forecasts are generated asynchronously. A request creates a **prediction job** that runs in the background. Polling the job endpoint returns the result once complete.

### Base URL

```
https://api.numinouslabs.io
```

---

## Authentication

The `POST /api/v1/forecasters/prediction-jobs` endpoint supports two authentication methods.

### Option 1 — x402 Pay-per-request

Access is granted by paying a small USDC fee per request via the [x402 protocol](https://x402.org). No account or API key is required — just a funded crypto wallet.

When a request arrives without a valid payment signature, the server responds with `402 Payment Required` and a `PAYMENT-REQUIRED` header containing payment instructions (price, accepted networks, destination wallets). An x402-compatible client handles this automatically: it reads the payment requirements, signs a payment payload with your wallet, and retries the request with a `PAYMENT-SIGNATURE` header.

**Accepted networks**

| Network | CAIP-2 Identifier | Token |
|---|---|---|
| Base | `eip155:8453` | USDC |
| Solana | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` | USDC |

See [examples/python.md](examples/python.md), [examples/typescript.md](examples/typescript.md), and [examples/go.md](examples/go.md) for working integrations.

### Option 2 — API Key

Authenticate by including your API key as a request header. The key bypasses x402 — no wallet or payment is required.

```
X-API-Key: your_api_key
```

API keys are self-serve. Create and revoke keys at [eversight.numinouslabs.io/api-keys](https://eversight.numinouslabs.io/api-keys). Each account can hold up to 5 active keys.

---

## Endpoints

### POST `/api/v1/forecasters/prediction-jobs`

Creates a new prediction job. The request is accepted immediately and the forecast is computed asynchronously. Use the returned `prediction_id` to poll for results.

**Request**

```
POST /api/v1/forecasters/prediction-jobs
Content-Type: application/json
PAYMENT-SIGNATURE: <x402 payment proof>   ← x402 auth
X-API-Key: <your_api_key>                 ← API key auth (alternative)
```

**Request Body**

The body supports two mutually exclusive modes: **structured mode** and **query mode**.

#### Optional Parameters

These apply to both modes.

| Field | Type | Required | Description |
|---|---|---|---|
| `agent_version_id` | `string (UUID)` | No | Pin the forecast to a specific miner agent version. When omitted, pool-based miner selection is used automatically. Version IDs can be found on the [Numinous leaderboard](https://leaderboard.numinouslabs.io) — navigate to any miner's page and copy the version ID from the agent code viewer. |

> **Tip:** Open any miner on the [leaderboard](https://leaderboard.numinouslabs.io), select a code version from the dropdown, and click **Copy Version ID**.

#### Structured Mode

Recommended when you have a well-defined event with known resolution criteria and a cutoff date. Produces the most precise results because the forecaster receives an unambiguous event specification with no intermediate parsing step.

| Field | Type | Required | Description |
|---|---|---|---|
| `title` | `string` | Yes | Event title phrased as a clear yes/no question. |
| `description` | `string` | Yes | Resolution criteria describing exactly how the event resolves. |
| `cutoff` | `string` | Yes | ISO 8601 datetime after which no further predictions are accepted. |
| `topics` | `array[string]` | No | Topic categories (e.g. `["crypto", "finance"]`). Improves routing and accuracy. Defaults to empty. |

```json
{
  "title": "Will Bitcoin exceed $150,000 before March 31, 2026?",
  "description": "Resolves YES if the Bitcoin spot price on any major exchange exceeds $150,000 USD at any point before 2026-03-31T23:59:59Z.",
  "cutoff": "2026-03-31T23:59:59Z",
  "topics": ["crypto", "finance"]
}
```

#### Query Mode

Ideal for rapid exploration or natural language pipelines. Submit a plain-language question and the API extracts the title, resolution criteria, cutoff, and topics automatically before running the forecast.

| Field | Type | Required | Description |
|---|---|---|---|
| `query` | `string` | Yes | A natural language question about a future event. |

```json
{
  "query": "Will Bitcoin exceed $150,000 before the end of Q1 2026?"
}
```

**Response** — `202 Accepted`

```json
{
  "prediction_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "status": "PENDING"
}
```

| Field | Type | Description |
|---|---|---|
| `prediction_id` | `string (UUID)` | Unique identifier for the prediction job. Use this to retrieve results. |
| `status` | `string` | Initial job status. Always `PENDING` on creation. |

**Error Responses**

| Status | Description |
|---|---|
| `402 Payment Required` | No valid payment signature or API key provided. |
| `500 Internal Server Error` | An error occurred while submitting the job. |

---

### GET `/api/v1/forecasters/prediction-jobs/{prediction_id}`

Retrieves the current status and result of a prediction job.

**Authentication:** None. The `prediction_id` UUID acts as an access token — keep it private.

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `prediction_id` | `string (UUID)` | The prediction job ID returned from `POST /prediction-jobs`. |

**Response** — `200 OK`

```json
{
  "prediction_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "status": "COMPLETED",
  "created_at": "2026-02-17T18:00:00Z",
  "result": {
    "prediction": 0.72,
    "forecaster_name": "pool_based_miner_forecaster",
    "forecasted_at": "2026-02-17T18:00:45Z",
    "metadata": {
      "miner_uid": 42,
      "miner_hotkey": "5F4tQyWrhfGVcNhoqeiNsR6KjD4wMZ2kfhLj4oHYuyHbZAc3",
      "pool": "GLOBAL_BRIER",
      "version_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "agent_name": "my_forecasting_agent",
      "version_number": 7,
      "raw_prediction": 0.72,
      "event_title": "Will Bitcoin exceed $150,000 before March 31, 2026?",
      "event_cutoff": "2026-03-31T23:59:59",
      "reasoning": "Based on current market momentum and historical BTC cycles, a 72% probability reflects strong bullish sentiment tempered by macro uncertainty."
    },
    "parsed_fields": null
  },
  "error": null
}
```

**Response Fields**

| Field | Type | Description |
|---|---|---|
| `prediction_id` | `string (UUID)` | Job identifier. |
| `status` | `string` | Current job status. See status values below. |
| `created_at` | `string (ISO 8601)` | When the job was created. |
| `result` | `object \| null` | Present when `status` is `COMPLETED`. |
| `result.prediction` | `float` | Probability estimate in `[0.0, 1.0]`. A value of `0.72` means 72% probability of YES resolution. |
| `result.forecaster_name` | `string` | Identifier of the forecaster that produced the result. |
| `result.forecasted_at` | `string (ISO 8601)` | When the forecast was generated. |
| `result.metadata` | `object \| null` | Forecaster metadata. For the pool-based forecaster includes: `miner_uid`, `miner_hotkey`, `pool`, `version_id`, `agent_name`, `version_number`, `raw_prediction`, `event_title`, `event_cutoff`, and optionally `reasoning`. |
| `result.parsed_fields` | `object \| null` | Fields extracted from the natural language query. Present only in **query mode** — always `null` in structured mode. Contains `title`, `description`, `cutoff`, and `topics`. |
| `error` | `string \| null` | Error description. Present when `status` is `FAILED`. |

**Job Status Values**

| Status | Description |
|---|---|
| `PENDING` | Job accepted, waiting to be processed. |
| `RUNNING` | Forecast is being computed. |
| `COMPLETED` | Forecast complete. `result` is populated. |
| `FAILED` | Forecast failed. `error` is populated. |

**Error Responses**

| Status | Description |
|---|---|
| `404 Not Found` | No job found for the given `prediction_id`. |

---

## Usage Flow

```
1. POST /api/v1/forecasters/prediction-jobs
   → 402 if no auth → pay via x402 client or include X-API-Key header
   → 202 { prediction_id, status: "PENDING" }

2. Poll GET /api/v1/forecasters/prediction-jobs/{prediction_id}
   → Repeat until status is COMPLETED or FAILED
   → Read result.prediction
```

A reasonable polling interval is 5 seconds. Jobs typically complete within 30–120 seconds depending on forecaster load.

---

## Client Examples

- [Python](examples/python.md)
- [TypeScript](examples/typescript.md)
- [Go](examples/go.md)
