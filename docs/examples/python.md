# Python Integration Example

## Dependencies

```bash
pip install x402 httpx eth-account
```

## x402 Pay-per-request

The x402 Python SDK wraps `httpx` and handles the `402 → sign → retry` flow automatically. Your wallet must hold USDC on Base or Solana mainnet.

```python
import asyncio
import os

import httpx
from eth_account import Account
from x402 import x402Client
from x402.http.clients import x402HttpxClient
from x402.mechanisms.evm import EthAccountSigner
from x402.mechanisms.evm.exact.register import register_exact_evm_client

BASE_URL = "https://api.numinouslabs.io"
PREDICTION_JOBS_URL = f"{BASE_URL}/api/v1/forecasters/prediction-jobs"

PAYLOAD = {"query": "Will Bitcoin exceed $150,000 before the end of Q1 2026?"}
POLL_INTERVAL = 5    # seconds
POLL_TIMEOUT = 180   # seconds


async def main() -> None:
    private_key = os.environ["BUYER_WALLET_PRIVATE_KEY"]  # EVM private key (hex)
    account = Account.from_key(private_key)

    client = x402Client()
    register_exact_evm_client(client, EthAccountSigner(account))

    print(f"Wallet: {account.address}")

    # Submit prediction job — x402 payment handled automatically
    async with x402HttpxClient(client, timeout=60) as http:
        resp = await http.post(PREDICTION_JOBS_URL, json=PAYLOAD)
        await resp.aread()

        if not resp.is_success:
            print(f"Failed: {resp.status_code} — {resp.text}")
            return

        data = resp.json()
        prediction_id = data["prediction_id"]
        print(f"Job created: {prediction_id}")

    # Poll for result
    elapsed = 0
    async with httpx.AsyncClient(timeout=10) as http:
        while elapsed < POLL_TIMEOUT:
            await asyncio.sleep(POLL_INTERVAL)
            elapsed += POLL_INTERVAL

            resp = await http.get(f"{PREDICTION_JOBS_URL}/{prediction_id}")
            data = resp.json()
            status = data["status"]
            print(f"  [{elapsed}s] status={status}")

            if status == "COMPLETED":
                result = data["result"]
                print(f"\nPrediction: {result['prediction']:.2%} probability")
                print(f"Forecaster: {result['forecaster_name']}")
                if result.get("metadata", {}).get("reasoning"):
                    print(f"Reasoning:  {result['metadata']['reasoning']}")
                return

            if status == "FAILED":
                print(f"Job failed: {data.get('error')}")
                return

    print("Timed out waiting for result.")


if __name__ == "__main__":
    asyncio.run(main())
```

## API Key

If you have a pre-issued API key, skip the x402 setup entirely and pass the key as a header.

```python
import asyncio
import httpx

BASE_URL = "https://api.numinouslabs.io"
PREDICTION_JOBS_URL = f"{BASE_URL}/api/v1/forecasters/prediction-jobs"

PAYLOAD = {"query": "Will Bitcoin exceed $150,000 before the end of Q1 2026?"}
POLL_INTERVAL = 5
POLL_TIMEOUT = 180


async def main() -> None:
    api_key = "YOUR_API_KEY"
    headers = {"X-API-Key": api_key}

    async with httpx.AsyncClient(timeout=60, headers=headers) as http:
        # Submit
        resp = await http.post(PREDICTION_JOBS_URL, json=PAYLOAD)
        resp.raise_for_status()

        data = resp.json()
        prediction_id = data["prediction_id"]
        print("Job created: {prediction_id}")

        # Poll
        elapsed = 0
        while elapsed < POLL_TIMEOUT:
            await asyncio.sleep(POLL_INTERVAL)
            elapsed += POLL_INTERVAL

            resp = await http.get(f"{PREDICTION_JOBS_URL}/{prediction_id}")
            data = resp.json()
            status = data["status"]
            print(f"  [{elapsed}s] status={status}")

            if status == "COMPLETED":
                result = data["result"]
                print(f"\nPrediction: {result['prediction']:.2%} probability")
                return

            if status == "FAILED":
                print(f"Job failed: {data.get('error')}")
                return

    print("Timed out waiting for result.")


if __name__ == "__main__":
    asyncio.run(main())
```
