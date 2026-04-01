# Cairn CLI

Cairn (`cairn`) is Backpac's agent-native command-line tool designed for autonomous settlement workflows.

Built entirely in Rust for maximum speed, memory safety, and cross-platform compatibility, Cairn operates completely headless (without interactive prompts) by default to favor machine readability and agent instrumentation.

## Deterministic Settlement

Cairn abstracts away the non-deterministic nature of blockchains (local mempools, gas spikes, reorgs) into a structured state machine. Agents treat transactions as **Intents** that are either `PENDING`, `CONFIRMED`, or `FINALIZED`.

## Features

- **Agent Identity Management**: DID-anchored authentication via EIP-4361 (Sign-in with Ethereum).
- **PoI Engine**: Create, link, and trace structured Proofs of Intent across disparate networks.
- **RPC Gateway**: Broadcast zero-knowledge transactions securely over Backpac's endpoint matrix.
- **SSE Real-Time Monitoring**: Stream live state transitions for intents and globally emitted agent events.
- **Trustless Verification**: A dedicated `receive` command for counterparty agents to verify payment finality and context without custom logic.
- **Native OpenWallet Standard (OWS)**: Integrated OWS support directly into the CLI. Effortlessly manage storage, encryption, and signatures for multiple disparate wallets natively, bypassing third-party wallet applications.
- **Agent Policy Engine**: Locally govern, gate, and simulate transaction execution bounds via programmable Agentic Guardrails (`policy`).
- **x402 Automation**: Native 402 detection and EIP-712 wallet integration for automatic payment resolution.

## Installation

### Prerequisites

- Rust 1.70 or higher
- Git

### Build from source

```bash
git clone https://github.com/Backpac-Inc/cairn-cli
cd cairn-cli

# Build release optimized binary
cargo build --release

# Place in your PATH
mv target/release/cairn ~/.local/bin/
```

## Quick Start: Configuration & Auth

### 1. Configure the environment

Set your preferred execution chain and network. These can also be overridden via env vars (`CAIRN_CHAIN`, `CAIRN_NETWORK`) or global flags.

```bash
cairn config set chain ethereum
cairn config set network mainnet
```

### 2. Initialize a Named Wallet (OWS)

Cairn natively supports the OpenWallet Standard. Before authenticating, create a named wallet profile. This name (`--name`) is used to resolve the wallet and perform cryptographic signatures during the challenge response.

```bash
cairn wallet create --name Agent1 --length 12
```

### 3. Authenticate

Cairn uses EIP-4361 (Sign-In with Ethereum). You must first request a challenge nonce, then connect and sign the payload.

You can authenticate explicitly using the `--wallet` flag, or automatically resolve your identity using a named OWS profile via `--ows`:

```bash
# Option A: Explicit Wallet Address
NONCE=$(cairn auth challenge \
  --wallet 0xYOUR_WALLET \
  --chain eip155:1 \
  --did did:key:z6MkYOURDID | jq -r .nonce)

cairn auth connect \
  --wallet 0xYOUR_WALLET \
  --chain eip155:1 \
  --did did:key:z6MkYOURDID \
  --nonce $NONCE \
  --key-file ~/.backpac/mykey.pem

# Option B: Native OWS Resolution (recommended)
NONCE=$(cairn auth challenge \
  --ows Agent1 \
  --chain eip155:1 \
  --did did:key:z6MkYOURDID | jq -r .nonce)

cairn auth connect \
  --ows Agent1 \
  --chain eip155:1 \
  --did did:key:z6MkYOURDID \
  --nonce $NONCE
```

*Your successful login JWT is securely encrypted and stored at `~/.backpac/credentials.json`.*

#### Credential Encryption

Because Cairn handles sensitive authentication data, it requires an encryption key for local storage. This encryption key is derived using a multi-stage fallback strategy:

1. **Environment Variable (`CAIRN_STORAGE_KEY`)**: Useful for CI/CD environments.
2. **Interactive TTY Prompt**: By default, Cairn will interactively prompt for a secure passphrase.
3. **Machine UID Fallback**: If `CAIRN_NON_INTERACTIVE=1` is set, or if running headless, Cairn will derive a deterministic fallback key based on the host machine's unique hardware signature.

## Agent Workflow: The SENDER

The Sender agent is responsible for initializing a Proof of Intent (PoI) and fulfilling it via one or more execution intents.

### 1. Check Pre-requisites

Ensure the agent has sufficient balance to execute. Through native OWS integration, `cairn balance` can simultaneously abstract away RPC node configurations and check multiple wallets out-of-the-box:

```bash
cairn balance
```

### 2. Initialize Settlement

Create a PoI which acts as the overarching session:

```bash
POI_ID=$(cairn poi create --ttl 600 | jq -r .id)
```

### 3. Dispatch Intent

Submit the transaction payload bound to the PoI.

```bash
cairn intent send \
  --method eth_sendRawTransaction \
  --params '["0xSignedTxData"]' \
  --poi-id $POI_ID \
  --confidence 0.99
```

### 4. Wait for Finality

Block until the transaction moves to a `FINALIZED` state:

```bash
cairn intent wait <INTENT_ID> --timeout 120
```

---

## Agent Workflow: The RECEIVER

The Receiver agent uses Cairn to verify that a counterparty has fulfilled their promise trustlessly.

### 1. The Trustless Handshake

Verify that a PoI exists, is finalized (SETTLED), and matches the expected payment context (recipient and value):

```bash
cairn receive \
  --poi-id <POI_ID> \
  --recipient did:key:z6Mk... \
  --expect-value "1.5" \
  --require-finalized
```

*Cairn returns **Exit Code 0** if and only if all conditions are met.*

### 2. Proof Audit & Storage

Fetch the full cryptographic Proof of Transport (PoT) bundle for long-term audit logs:

```bash
# Fetch raw JSON bundle after local signature verification
cairn proof get <INTENT_ID> --verify-signature --raw
```

---

## Advanced Features

### SSE Monitoring

Stream live state transitions for a specific intent or all account-wide events:

```bash
cairn watch intent <INTENT_ID>
cairn watch agent
```

### Automatic x402 Payments

Enable automatic L402 challenge resolution. If an RPC call returns a 402, Cairn will sign and pay the credit invoice using your local wallet:

```bash
export CAIRN_AUTO_PAY=1
cairn intent send ...
```

## Global Flags & Modifiers

| Flag | Env Var | Description |
| :--- | :--- | :--- |
| `--output <FORMAT>` | `CAIRN_OUTPUT` | Switch output format (`json` or `text`). |
| `--quiet`, `-q` | | Suppresses stdout, returning only execution exit codes. |
| `--api-url <URL>` | `BACKPAC_API_URL` | Temporarily overrides API location. |
| `--jwt <TOKEN>` | `BACKPAC_JWT` | Submits the provided token instead of local state. |

## Command Hierarchy

### `auth`

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `auth challenge` | `--wallet`, `--chain`, `--did` | Request an EIP-4361 signing challenge payload. |
| `auth connect` | `--wallet`, `--chain`, `--did`, `--nonce`, `--signature`, `[--key_file]` | Submit challenge signature. Saves JWT to `credentials.json`. Auto-signs if `--key_file` provided. |
| `auth refresh` | None | Re-mints JWT using the currently valid JWT. Updates `credentials.json`. |
| `auth status` | None | Checks token health, expiry, and scopes. |

### `identity`

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `identity register` | `--did`, `--wallet`, `--public-key`, `[--display-name]` | Anchor a DID to the wallet. |
| `identity rotate` | `--did`, `--current-key`, `--new-key` | Rotate the Ed25519 signing key for the DID. |
| `identity get` | `<did>` | Look up state and current key for a registered DID. |

### `poi` (Proof of Intent)

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `poi create` | `[--chain]`, `[--network]`, `[--parent]`, `[--max-depth]`, `[--ttl]`, `[--metadata]` | Create a new Proof of Intent tracking record. |
| `poi get` | `<poi_id>` | Retrieve an existing PoI. |

### `intent`

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `intent send` | `--method`, `--params`, `[--poi-id]`, `[--confidence]`, `[--id]` | Submit JSON-RPC payload. Binds to PoI if `--poi-id` provided (via `X-Backpac-Poi-Id` header). |
| `intent status` | `<intent_id>` | Check state of a specific execution intent. |
| `intent verify` | `<intent_id>`, `[--receiver-did]`, `[--min-confidence]` | Receiver-side verification endpoint for a settled PoI. |
| `intent wait` | `<intent_id>`, `[--interval]`, `[--timeout]` | Client-side poll until status is `FINALIZED`, `ABORTED`, or `EXPIRED`. |
| `intent list` | `[--status]`, `[--since]`, `[--limit]` | `GET /v1/intents` | List and filter authenticated agent's intents. |

### `watch`

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `watch intent` | `<intent_id>` | Stream live SSE state transitions for a specific intent. |
| `watch agent` | None | Stream live SSE account-wide agent events and notifications. |

### `proof`

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `proof get` | `<intent_id>`, `[--include-telemetry]`, `[--include-children]`, `[--verify-signature]`, `[--raw]` | Fetches the structured cryptographic PoT bundle. |
| `proof verify` | `<intent_id>` | Fetches bundle, fetches JWKS from issuer, performs local Ed25519 cryptographic signature verification against the payload hash. |

### `receive`

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `receive` | `--poi-id`, `[--from]`, `[--recipient]`, `[--expect-value]`, `[--require-finalized]` | Combined fetch and verify step for receiver agents. Validates sender, recipient, and value contexts. |

### `config`

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `config set` | `<key>`, `<value>` | Edits values in `~/.backpac/config.json`. |
| `config get` | `[<key>]` | Fetches a specific value, or dumps the full configuration structure. |

### `balance`

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `balance` | `[--wallet]`, `[--chain]` | Using built-in OWS wrappers, queries native token balances securely and uniformly across supported chains without explicit RPC configuration. |

### `wallet`

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `wallet list` | None | List all configured OWS wallets stored natively within the CLI. |
| `wallet create` | `--name`, `[--length]`, `[--passphrase]` | Create a new named OWS wallet (mnemonic and encrypted storage). |
| `wallet delete` | `--name` | Delete an existing OWS wallet by name. |

### `policy`

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `policy evaluate` | `--intent-id` | Run local execution boundaries and guardrails against an expected intent. |
| `policy set` | `--rules` | Apply a local JSON-based policy config to gate transactions autonomously. |

## Exit Codes

Errors must be strictly mapped from HTTP responses to integer exit codes to allow programmatic behavior.

| Condition / Meaning | Exit Code | HTTP Mapping |
| :--- | :---: | :--- |
| Success | `0` | `200`, `201`, `204` |
| General Error (Serialization, IO) | `1` | `500` or OS-level errors |
| Value Mismatch | `2` | PoI context validation failure |
| Not Finalized | `3` | `require-finalized` set but state is pending |
| Forbidden / Unauthorized | `4` | `401 Unauthorized` / `403 Forbidden` |
| Not Found | `5` | `404 Not Found` |
| Conflict | `6` | `409 Conflict` |
| Intent Expired | `7` | `410 Gone` (if message contains "expired") |
| Intent Aborted | `8` | `410 Gone` |
| Chain Depth Exceeded | `9` | Payload constraint check |
| Operation Timeout | `10` | `intent wait` duration exceeded |
| Signature Verification Error | `11` | Cryptographic match failure |
| PoT Not Ready | `12` | Proof missing signatures/payloads |
| Network Error | `13` | Unreachable endpoint, connection reset |
| Token Expired | `14` | `401 Unauthorized` (if message contains "expired") |
| Challenge Expired | `15` | Spec constraint |
| Insufficient Funds / x402 | `16` | `402 Payment Required` |




## Development

```bash
cargo test
```
