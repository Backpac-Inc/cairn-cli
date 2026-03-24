# Cairn CLI

Cairn (`cairn`) is Backpac's agent-native command-line tool designed for autonomous settlement workflows.

Built entirely in Rust for maximum speed, memory safety, and cross-platform compatibility, Cairn operates completely headless (without interactive prompts) by default to favor machine readability and agent instrumentation.

## Features

- **Agent Identity Management**: DID-anchored authentication via EIP-4361 (Sign-in with Ethereum).
- **PoI Engine**: Create, link, and trace structured Proofs of Intent across disparate networks.
- **RPC Gateway**: Broadcast zero-knowledge transactions securely over Backpac's endpoint matrix.
- **SSE Real-Time Monitoring**: Stream live state transitions for intents and globally emitted agent events.
- **Proof Verification**: Client-side cryptography ensures institutional trust through JWKS signed bundles.
- **x402 Automation**: Native 402 detection and EIP-712 wallet integration for automatic payment resolution.

## Installation

### Prerequisites

- Rust 1.94.0 or higher
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

## Quick Start

### 1. Configure the environment

Set your preferred execution chain and network:

```bash
cairn config set chain ethereum
cairn config set network mainnet
```

### 2. Authenticate

Get a challenge payload, sign it with your wallet private key, and authenticate to receive a JWT session.

If you have your private key saved locally (e.g. at `~/.backpac/mykey.pem`), you can automate the flow:

```bash
# Request challenge
CHALLENGE=$(cairn auth challenge --wallet 0xYOUR_WALLET --chain eip155:1 --did did:key:z6MkYOURDID)
NONCE=$(echo $CHALLENGE | jq -r .nonce)

# Connect and sign
cairn auth connect \
  --wallet 0xYOUR_WALLET \
  --chain eip155:1 \
  --did did:key:z6MkYOURDID \
  --nonce $NONCE \
  --signature "" \
  --key-file ~/.backpac/mykey.pem
```

*Your successful login JWT is securely stored at `~/.backpac/credentials.json`.*

### 3. Dispatch an RPC Intent

Initialize a Proof of Intent tracker, and dispatch a JSON-RPC call linked to it:

```bash
# Initialize a PoI
POI_ID=$(cairn poi create --ttl 600 | jq -r .id)

# Dispatch intent with PoI binding
cairn intent send \
  --method eth_sendRawTransaction \
  --params '["0xSignedTxData"]' \
  --poi-id $POI_ID \
  --confidence 0.95
```

### 4. Wait, Watch, and Verify

You can block execution until the transaction lands on-chain, stream live updates, and verify the cryptographic signature of the receipt.

```bash
# Wait up to 120 seconds, polling every 2 seconds
cairn intent wait <INTENT_ID> --timeout 120 --interval 2

# Stream live status updates via SSE
cairn watch intent <INTENT_ID>

# Stream all global agent notifications and events via SSE
cairn watch agent

# Verify Backpac's JWKS signature on the resulting Proof of Transport bundle
cairn proof verify <INTENT_ID>
```

## Global Configuration & Flags

Every command supports global modifiers for easy execution overriding:

| Flag | Env Var | Description |
|:---|:---|:---|
| `--output <FORMAT>` | `CAIRN_OUTPUT` | Switch output format (`json` or `text`). |
| `--quiet`, `-q` | | Suppresses stdout, returning only execution exit codes. |
| `--api-url <URL>` | `BACKPAC_API_URL` | Temporarily overrides API location. |
| `--jwt <TOKEN>` | `BACKPAC_JWT` | Submits the provided token instead of local state. |

## Exit Codes

Cairn translates all system boundaries into deterministic exit codes, allowing agent environments to securely evaluate failure models.

| Code | Meaning |
|:---:|:---|
| 0 | Success |
| 1 | General error (IO/Serialization) |
| 2 | Invalid user input |
| 3 | Authentication Error |
| 4 | Missing resource (404) |
| 6 | Intent Expired |
| 7 | Intent Aborted |
| 9 | CLI Timeout |
| 15 | Insufficient Funds / x402 Trigger |

See `CLI_DESIGN.md` for the full 16-code mapping.

## Development

Run tests:

```bash
cargo test
```
