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

---

## Installation

Cairn is distributed via multiple package managers and as a pre-compiled binary for maximum cross-platform compatibility.

### 1. Via Cargo (Rust Package Manager)

If you have Rust installed, you can install Cairn directly from [crates.io](https://crates.io/crates/cairn-cli):

```bash
cargo install cairn-cli
```

### 2. Via Homebrew (macOS / Linux)

Add the Backpac tap and install the CLI:

```bash
brew tap backpac-inc/cairn
brew install cairn-cli
```

### 3. Pre-compiled Binaries

Download the latest static binary for your operating system from the [official releases](https://github.com/Backpac-Inc/cairn-cli/releases).

---

## Global Flags & Modifiers

When executing Cairn commands, these global flags can be used to control output format and behavior:

| Flag | Env Var | Description |
| :--- | :--- | :--- |
| `--output <FORMAT>` | `CAIRN_OUTPUT` | Switch output format (`json` or `text`). |
| `--quiet`, `-q` | | Suppresses stdout, returning only execution exit codes. |
| `--api-url <URL>` | `BACKPAC_API_URL` | Temporarily overrides API location. |
| `--jwt <TOKEN>` | `BACKPAC_JWT` | Submits the provided token instead of local state. |

---

## 1. Authentication (EIP-4361 Sign-In with Ethereum)

Agents must authenticate via a Wallet Identity using the EIP-4361 specification.

### CLI Commands (`auth`)

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `auth challenge` | `--wallet`, `--chain`, `--did` | Request an EIP-4361 signing challenge payload. |
| `auth connect` | `--wallet`, `--chain`, `--did`, `--nonce`, `--signature`, `[--key_file]` | Submit challenge signature. Saves JWT to `credentials.json`. Auto-signs if `--key_file` provided. |
| `auth refresh` | None | Re-mints JWT using the currently valid JWT. Updates `credentials.json`. |
| `auth status` | None | Checks token health, expiry, and scopes. |

---

## 2. Identity Management (`identity`)

Anchor and manage Decentralized Identifiers (DIDs) on-chain.

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `identity register` | `--did`, `--wallet`, `--public-key`, `[--display-name]` | Anchor a DID to the wallet. |
| `identity rotate` | `--did`, `--current-key`, `--new-key` | Rotate the Ed25519 signing key for the DID. |
| `identity get` | `<did>` | Look up state and current key for a registered DID. |

---

## 3. Settlement Workflow: The SENDER

### I. Initialize Settlement (`poi`)

Create a PoI which acts as the overarching session:

```bash
# Create PoI with 10 minute TTL
POI_ID=$(cairn poi create --ttl 600 | jq -r .id)
```

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `poi create` | `[--chain]`, `[--network]`, `[--parent]`, `[--max-depth]`, `[--ttl]`, `[--metadata]` | Create a new Proof of Intent tracking record. |
| `poi get` | `<poi_id>` | Retrieve an existing PoI. |

### II. Dispatch Intent (`intent`)

Submit the transaction payload bound to the PoI.

```bash
cairn intent send \
  --method eth_sendRawTransaction \
  --params '["0xSignedTxData"]' \
  --poi-id $POI_ID \
  --confidence 0.99
```

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `intent send` | `--method`, `--params`, `[--poi-id]`, `[--confidence]`, `[--id]` | Submit JSON-RPC payload. Binds to PoI if `--poi-id` provided. |
| `intent status` | `<intent_id>` | Check state of a specific execution intent. |
| `intent verify` | `<intent_id>`, `[--receiver-did]`, `[--min-confidence]` | Receiver-side verification endpoint for a settled PoI. |
| `intent list` | `[--status]`, `[--since]`, `[--limit]` | List and filter authenticated agent's intents. |

### III. Wait for Finality

Block until the transaction moves to a `FINALIZED` state:

```bash
cairn intent wait <INTENT_ID> --timeout 120
```

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `intent wait` | `<intent_id>`, `[--interval]`, `[--timeout]` | Client-side poll until status is `FINALIZED`, `ABORTED`, or `EXPIRED`. |

---

## 4. Settlement Workflow: The RECEIVER

### I. The Trustless Handshake (`receive`)

Verify that a PoI exists, is finalized (SETTLED), and matches the expected payment context:

```bash
cairn receive \
  --poi-id <POI_ID> \
  --recipient did:key:z6Mk... \
  --expect-value "1.5" \
  --require-finalized
```

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `receive` | `--poi-id`, `[--from]`, `[--recipient]`, `[--expect-value]`, `[--require-finalized]` | Combined fetch and verify step for receiver agents. Validates sender, recipient, and value contexts. |

### II. Proof Audit (`proof`)

Fetch the full cryptographic Proof of Transport (PoT) bundle for long-term audit logs:

```bash
# Verify the JWKS signature and local payload hash
cairn proof verify <INTENT_ID>

# Fetch raw JSON bundle after local signature verification
cairn proof get <INTENT_ID> --verify-signature --raw
```

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `proof get` | `<intent_id>`, `[--include-telemetry]`, `[--include-children]`, `[--verify-signature]`, `[--raw]` | Fetches the structured cryptographic PoT bundle. |
| `proof verify` | `<intent_id>` | Fetches bundle and performs local signature verification. |

---

## 5. Monitoring & SSE (`watch`)

Stream live state transitions for a specific intent or account-wide events.

```bash
cairn watch intent <INTENT_ID>
cairn watch agent
```

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `watch intent` | `<intent_id>` | Stream live SSE state transitions for a specific intent. |
| `watch agent` | None | Stream live SSE account-wide agent events and notifications. |

---

## 6. Automated x402 Payments

Enable automatic L402 challenge resolution. If an RPC call returns a 402, Cairn will sign and pay the credit invoice using your local wallet:

```bash
export CAIRN_AUTO_PAY=1
cairn intent send ...
```

---

## 7. CLI Exit Codes (Agent Contract)

Programmatic behavior must strictly rely on exit codes mapping HTTP responses to system boundaries.

| Condition / Meaning | Exit Code | HTTP Mapping |
| :--- | :---: | :--- |
| Success | `0` | `200`, `201`, `204` |
| General Error (Serialization, IO) | `1` | `500` or OS-level errors |
| Value Mismatch | `2` | PoI context validation failure |
| Not Finalized | `3` | `require-finalized` set but state is pending |
| Forbidden / Unauthorized | `4` | `401 Unauthorized` / `403 Forbidden` |
| Not Found | `5` | `404 Not Found` |
| Conflict | `6` | `409 Conflict` |
| Intent Expired | `7` | `410 Gone` (Expired) |
| Intent Aborted | `8` | `410 Gone` (Aborted) |
| Chain Depth Exceeded | `9` | Payload constraint check |
| Operation Timeout | `10` | `intent wait` duration exceeded |
| Signature Verification Error | `11` | Cryptographic match failure |
| PoT Not Ready | `12` | Proof missing signatures/payloads |
| Network Error | `13` | Unreachable endpoint |
| Token Expired | `14` | `401 Unauthorized` (Token Expired) |
| Challenge Expired | `15` | Auth challenge timeout |
| Insufficient Funds / x402 | `16` | `402 Payment Required` |
