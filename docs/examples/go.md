# Go Integration Example

## Dependencies

```bash
go get github.com/coinbase/x402/go
go get github.com/ethereum/go-ethereum/crypto
```

## x402 Pay-per-request

The x402 Go SDK wraps `net/http` and handles the `402 → sign → retry` flow automatically via `DoWithPayment`. Your wallet must hold USDC on Base mainnet.

```go
package main

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "time"

    x402 "github.com/coinbase/x402/go"
    x402http "github.com/coinbase/x402/go/http"
    evmclient "github.com/coinbase/x402/go/mechanisms/evm/exact/client"
    evmsigner "github.com/coinbase/x402/go/signers/evm"
)

const (
    baseURL           = "https://api.numinouslabs.io"
    predictionJobsURL = baseURL + "/api/v1/forecasters/prediction-jobs"
    pollInterval      = 5 * time.Second
    pollTimeout       = 180 * time.Second
)

func main() {
    privateKeyHex := os.Getenv("BUYER_WALLET_PRIVATE_KEY") // hex, with or without 0x

    signer, err := evmsigner.NewClientSignerFromPrivateKey(privateKeyHex)
    if err != nil {
        panic(fmt.Sprintf("invalid private key: %v", err))
    }
    fmt.Printf("Wallet: %s\n", signer.Address())

    // Create x402 client and register EVM payment scheme
    client := x402.Newx402Client().Register("eip155:*", evmclient.NewExactEvmScheme(signer))
    httpClient := x402http.Newx402HTTPClient(client)

    // Submit prediction job — x402 payment handled automatically
    payload := map[string]string{"query": "Will Bitcoin exceed $150,000 before the end of Q1 2026?"}
    body, _ := json.Marshal(payload)

    ctx := context.Background()
    req, _ := http.NewRequestWithContext(ctx, "POST", predictionJobsURL, bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    fmt.Println("Submitting prediction job...")
    resp, err := httpClient.DoWithPayment(ctx, req)
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()

    respBody, _ := io.ReadAll(resp.Body)
    if resp.StatusCode != http.StatusAccepted {
        fmt.Printf("Failed: %d — %s\n", resp.StatusCode, string(respBody))
        return
    }

    var submit struct {
        PredictionID string `json:"prediction_id"`
    }
    json.Unmarshal(respBody, &submit)
    fmt.Printf("Job created: %s\n", submit.PredictionID)

    // Poll for result
    pollURL := fmt.Sprintf("%s/%s", predictionJobsURL, submit.PredictionID)
    deadline := time.Now().Add(pollTimeout)
    stdClient := &http.Client{Timeout: 10 * time.Second}

    for time.Now().Before(deadline) {
        time.Sleep(pollInterval)

        pollResp, _ := stdClient.Get(pollURL)
        var data map[string]any
        json.NewDecoder(pollResp.Body).Decode(&data)
        pollResp.Body.Close()

        status := data["status"].(string)
        fmt.Printf("  status=%s\n", status)

        if status == "COMPLETED" {
            result := data["result"].(map[string]any)
            fmt.Printf("\nPrediction: %.1f%% probability\n", result["prediction"].(float64)*100)
            fmt.Printf("Forecaster: %s\n", result["forecaster_name"].(string))
            if meta, ok := result["metadata"].(map[string]any); ok {
                if reasoning, ok := meta["reasoning"].(string); ok {
                    fmt.Printf("Reasoning:  %s\n", reasoning)
                }
            }
            return
        }
        if status == "FAILED" {
            fmt.Printf("Job failed: %v\n", data["error"])
            return
        }
    }
    fmt.Println("Timed out waiting for result.")
}
```

## API Key

If you have a pre-issued API key, use the standard `net/http` client with the key header — no x402 packages needed.

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

const (
    baseURL           = "https://api.numinouslabs.io"
    predictionJobsURL = baseURL + "/api/v1/forecasters/prediction-jobs"
    apiKey            = "YOUR_API_KEY"
    pollInterval      = 5 * time.Second
    pollTimeout       = 180 * time.Second
)

func main() {
    client := &http.Client{Timeout: 60 * time.Second}

    payload := map[string]string{"query": "Will Bitcoin exceed $150,000 before the end of Q1 2026?"}
    body, _ := json.Marshal(payload)

    req, _ := http.NewRequest("POST", predictionJobsURL, bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-API-Key", apiKey)

    resp, err := client.Do(req)
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()

    var submit struct {
        PredictionID string `json:"prediction_id"`
    }
    json.NewDecoder(resp.Body).Decode(&submit)
    fmt.Printf("Job created: %s\n", submit.PredictionID)

    // Poll
    pollURL := fmt.Sprintf("%s/%s", predictionJobsURL, submit.PredictionID)
    deadline := time.Now().Add(pollTimeout)

    for time.Now().Before(deadline) {
        time.Sleep(pollInterval)

        pollResp, _ := http.Get(pollURL)
        var data map[string]any
        json.NewDecoder(pollResp.Body).Decode(&data)
        pollResp.Body.Close()

        status := data["status"].(string)
        fmt.Printf("  status=%s\n", status)

        if status == "COMPLETED" {
            result := data["result"].(map[string]any)
            fmt.Printf("\nPrediction: %.1f%% probability\n", result["prediction"].(float64)*100)
            return
        }
        if status == "FAILED" {
            fmt.Printf("Job failed: %v\n", data["error"])
            return
        }
    }
    fmt.Println("Timed out waiting for result.")
}
```
