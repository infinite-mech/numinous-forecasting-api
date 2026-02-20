# TypeScript Integration Example

## Dependencies

```bash
npm install @x402/fetch @x402/evm viem
```

## x402 Pay-per-request

The `@x402/fetch` package wraps the native `fetch` API and handles the `402 → sign → retry` flow automatically. Your wallet must hold USDC on Base mainnet.

```typescript
import { wrapFetchWithPaymentFromConfig } from "@x402/fetch";
import { ExactEvmScheme } from "@x402/evm";
import { privateKeyToAccount } from "viem/accounts";

const BASE_URL = "https://api.numinouslabs.io";
const PREDICTION_JOBS_URL = `${BASE_URL}/api/v1/forecasters/prediction-jobs`;

const PAYLOAD = { query: "Will Bitcoin exceed $150,000 before the end of Q1 2026?" };
const POLL_INTERVAL_MS = 5_000;
const POLL_TIMEOUT_MS = 180_000;

async function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function main(): Promise<void> {
  const privateKey = process.env.BUYER_WALLET_PRIVATE_KEY as `0x${string}`;
  const account = privateKeyToAccount(privateKey);

  console.log(`Wallet: ${account.address}`);

  const fetchWithPayment = wrapFetchWithPaymentFromConfig(fetch, {
    schemes: [{ network: "eip155:*", client: new ExactEvmScheme(account) }],
  });

  // Submit prediction job — x402 payment handled automatically
  const submitResp = await fetchWithPayment(PREDICTION_JOBS_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(PAYLOAD),
  });

  if (!submitResp.ok) {
    console.error(`Failed: ${submitResp.status} — ${await submitResp.text()}`);
    return;
  }

  const { prediction_id } = await submitResp.json();
  console.log(`Job created: ${prediction_id}`);

  // Poll for result
  const deadline = Date.now() + POLL_TIMEOUT_MS;
  while (Date.now() < deadline) {
    await sleep(POLL_INTERVAL_MS);

    const pollResp = await fetch(`${PREDICTION_JOBS_URL}/${prediction_id}`);
    const data = await pollResp.json();
    console.log(`  status=${data.status}`);

    if (data.status === "COMPLETED") {
      const { prediction, forecaster_name, metadata } = data.result;
      console.log(`\nPrediction: ${(prediction * 100).toFixed(1)}% probability`);
      console.log(`Forecaster: ${forecaster_name}`);
      if (metadata?.reasoning) {
        console.log(`Reasoning:  ${metadata.reasoning}`);
      }
      return;
    }

    if (data.status === "FAILED") {
      console.error(`Job failed: ${data.error}`);
      return;
    }
  }

  console.error("Timed out waiting for result.");
}

main().catch(console.error);
```

## API Key

If you have a pre-issued API key, no extra packages are needed — just the native `fetch`. Pass the key as a header.

```typescript
const BASE_URL = "https://api.numinouslabs.io";
const PREDICTION_JOBS_URL = `${BASE_URL}/api/v1/forecasters/prediction-jobs`;

const API_KEY = "YOUR_API_KEY";
const PAYLOAD = { query: "Will Bitcoin exceed $150,000 before the end of Q1 2026?" };
const POLL_INTERVAL_MS = 5_000;
const POLL_TIMEOUT_MS = 180_000;

async function main(): Promise<void> {
  const headers = {
    "Content-Type": "application/json",
    "X-API-Key": API_KEY,
  };

  // Submit
  const submitResp = await fetch(PREDICTION_JOBS_URL, {
    method: "POST",
    headers,
    body: JSON.stringify(PAYLOAD),
  });

  if (!submitResp.ok) throw new Error(`Submit failed: ${submitResp.status}`);

  const { prediction_id } = await submitResp.json();
  console.log(`Job created: ${prediction_id}`);

  // Poll
  const deadline = Date.now() + POLL_TIMEOUT_MS;
  while (Date.now() < deadline) {
    await new Promise((r) => setTimeout(r, POLL_INTERVAL_MS));

    const pollResp = await fetch(`${PREDICTION_JOBS_URL}/${prediction_id}`);
    const data = await pollResp.json();
    console.log(`  status=${data.status}`);

    if (data.status === "COMPLETED") {
      console.log(`\nPrediction: ${(data.result.prediction * 100).toFixed(1)}% probability`);
      return;
    }

    if (data.status === "FAILED") {
      console.error(`Job failed: ${data.error}`);
      return;
    }
  }

  console.error("Timed out waiting for result.");
}

main().catch(console.error);
```
