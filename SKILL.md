---
name: backpac-agentic-settlement
description: Complete instructions for agents on how to construct zero-knowledge on-chain transaction workflows using the Cairn CLI or raw REST APIs. Covers EIP-4361 authentication, Proof of Intent (PoI) creation, intent broadcasting, and Proof of Transport verification.
---

# Backpac Agentic Settlement Skill

This skill empowers AI agents to seamlessly orchestrate, execute, and verify blockchain transactions across disparate networks using Backpac's Assured RPC Gateway. Instead of dealing with raw probability models like mempools and RPC timeouts, agents can interact with a deterministic settlement execution engine.

Backpac translates non-deterministic on-chain systems into exactly three simplified states:
1. `PENDING` (Intent received, awaiting ingestion)
2. `CONFIRMED` (Intent has landed on-chain, awaiting finality threshold)
3. `FINALIZED` (Zero Risk, immutable settlement)

## 1. Authentication (EIP-4361 Sign-In with Ethereum)

Agents must authenticate via a Wallet Identity using the EIP-4361 specification. To do this, you use a decentralized identifier (DID) (e.g. `did:key:z6Mk...`).

### Using Cairn CLI

```bash
# 1. Generate challenge payload
cairn auth challenge --wallet 0xAGENT_STR --chain eip155:1 --did did:key:z6Mk...

# 2. Connect using the generated challenge nonce and a valid ECDSA signature
cairn auth connect \
  --wallet 0xAGENT_STR \
  --chain eip155:1 \
  --did did:key:z6Mk... \
  --nonce <CHALLENGE_NONCE> \
  --signature <HEX_SIGNATURE>
```

*Note: If local keys are available, use `--key-file ~/.backpac/keys.pem` and the CLI will auto-sign.*

### Using Raw API

```http
POST /v1/agents/challenge
{
  "wallet": "0xAGENT_STR",
  "chainId": "eip155:1",
  "aud": "api.backpac.xyz",
  "did": "did:key:z6Mk..."
}

POST /v1/agents/connect
{
  "wallet": "0xAGENT_STR",
  "chainId": "eip155:1",
  "aud": "api.backpac.xyz",
  "did": "did:key:z6Mk...",
  "nonce": "<NONCE_FROM_CHALLENGE>",
  "signature": "<HEX_SIGNATURE>"
}
```

*Response will contain a `jwt` token which must be placed in `Authorization: Bearer <TOKEN>`.*

## 2. Proof of Intent (PoI)

Before broadcasting any transaction, create an overarching tracking session known as a Proof of Intent.

### Using Cairn CLI (PoI)

```bash
cairn poi create --ttl 600
```

This returns a JSON object containing the `id`.

### Using Raw API (PoI)

```http
POST /v1/pois
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "ttl": 600
}
```

## 3. Emitting Execution Intents

Once you have a PoI, you can dispatch raw transaction payloads to Backpac. Backpac acts as a standard JSON-RPC gateway but injects institutional orchestration rules around it.

### Using Cairn CLI (Intents)

When using the CLI, it automatically wraps your RPC payload into the exact format.

```bash
cairn intent send \
  --method eth_sendRawTransaction \
  --params '["0xSIGNED_TX_BYTES"]' \
  --poi-id <POI_ID_FROM_PREVIOUS_STEP> \
  --confidence 0.95
```

### Using Raw API (Intents)

You must send your request directly to the root RPC path (`/`) but include specific Assured Headers.

```http
POST /
Authorization: Bearer <TOKEN>
X-Backpac-Poi-Id: <POI_ID_FROM_PREVIOUS_STEP>
X-Backpac-Confirmations: 24
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "eth_sendRawTransaction",
  "params": ["0xSIGNED_TX_BYTES"]
}
```

*Response will yield the transaction hash (`result: "0x..."`) if successfully ingested. If the system fails to pick it up, standard JSON-RPC errors apply.*

## 4. Intent Tracing and Webhooks

To confirm whether a transaction successfully settled without parsing complex active RPC traces, Backpac provides both polling and Server-Sent Events (SSE) streaming capabilities.

### Using Cairn CLI (Tracing)

```bash
# Block execution until resolution (Polling)
cairn intent wait <INTENT_ID> --timeout 120 --interval 2

# Stream live status updates (Server-Sent Events)
cairn watch intent <INTENT_ID>

# Stream live global agent notifications
cairn watch agent
```

### Using Raw API (Tracing)

```http
# Standard Polling
GET /v1/intents/:intent_id
Authorization: Bearer <TOKEN>

# Event Streaming (SSE)
GET /v1/intents/:intent_id/stream
Authorization: Bearer <TOKEN>
```

Check the `status` field for `FINALIZED`.

## 5. Settlement Receipt: Proof of Transport (PoT)

Once a transaction has `FINALIZED`, Backpac creates a cryptographic bundle attesting to its occurrence and state transition. This bundle is signed using Backpac's own institutional Ed25519 identity matrix.

### Using Cairn CLI (PoT)

```bash
# Verify the JWKS signature and local payload hash
cairn proof verify <INTENT_ID>

# Or fetch the raw verified bundle for storage
cairn proof get <INTENT_ID> --verify-signature --raw
```

The CLI automatically downloads the JWKS (JSON Web Key Set), matches the versions, and computes the cryptographic verification locally against the payload hash.

## 6. Receiver Verification (Trustless Settlement)

If you are the recipient of an intent, you can verify its validity and finality in a single step using the `receive` command. This is critical for agents acting as automated merchants or oracles.

```bash
# Verify intent for a specific recipient and minimum value
cairn receive \
  --poi-id <POI_ID> \
  --recipient <YOUR_DID_OR_ADDRESS> \
  --expect-value 1.5 \
  --require-finalized
```

This command returns Exit Code `0` only if the intent is strictly finalized and all context matches. If the intent is still pending, it returns Exit Code `3`.

### Using Raw API (PoT)

As an agent building an audit trail, fetch the proof:

```http
GET /v1/proofs/:intent_id
Authorization: Bearer <TOKEN>
```

In your own environment, you should independently fetch `GET /.well-known/jwks.json` to extract the Backpac public key, split the signature string payload, construct the canonical string `[protocol]:[version]:{intentId}:{accountId}:{payloadHash}:{metadataHash}`, and perform an Ed25519 verification.

## 6. Automated x402 Payments

Backpac utilizes a "credits-first" execution model. When an agent's SLA volume is depleted, the gateway responds with `HTTP 402 Payment Required` and an `L402` or `Www-Authenticate` header containing the billing challenge. 
The Cairn CLI automatically intercepts these challenges. Set the `CAIRN_AUTO_PAY=1` environment variable to enable the CLI to map the challenge to an EIP-712 payment authorization, sign it using the local wallet, and automatically replay the transaction.
