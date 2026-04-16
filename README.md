# x402-facilitator

Universal [x402 payment protocol](https://www.x402.org/) facilitator for humans and AI agents. Verify and settle crypto payments across multiple blockchains through a single HTTP server.

> Forked from [x402-rs](https://github.com/anthropics/x402-rs) (Oct 2025). Released under Apache 2.0.

## What is x402?

The x402 protocol enables machine-to-machine and human-to-machine payments using HTTP status code 402. A **facilitator** is the settlement layer — it verifies payment signatures off-chain and executes on-chain transfers when requested.

## Supported Networks

| Network | Chain ID | Type | Settlement Method |
|---------|----------|------|-------------------|
| Base | 8453 | Mainnet | ERC-3009 / EIP-2612 |
| Base Sepolia | 84532 | Testnet | ERC-3009 / EIP-2612 |
| Polygon | 137 | Mainnet | ERC-3009 / EIP-2612 |
| Polygon Amoy | 80002 | Testnet | ERC-3009 / EIP-2612 |
| Avalanche C-Chain | 43114 | Mainnet | ERC-3009 / EIP-2612 |
| Avalanche Fuji | 43113 | Testnet | ERC-3009 / EIP-2612 |
| Sei | 1329 | Mainnet | ERC-3009 / EIP-2612 |
| Sei Testnet | 1328 | Testnet | ERC-3009 / EIP-2612 |
| Shibarium | 109 | Mainnet | Permit2 |
| Shibarium Puppynet | 157 | Testnet | Permit2 |
| XDC | 50 | Mainnet | ERC-3009 / EIP-2612 |
| XRPL EVM | 1440000 | Mainnet | ERC-3009 / EIP-2612 |
| Solana | — | Mainnet | SPL Token Transfer |
| Solana Devnet | — | Devnet | SPL Token Transfer |

Networks are **dynamically enabled** based on which `RPC_URL_*` environment variables you provide.

## Supported Tokens

**USDC** is the primary supported stablecoin, with verified contract addresses on every network above.

Shibarium (eip155:109) additionally supports all Permit2-compatible ERC-20 tokens in the ecosystem:

**Stablecoins**

| Token | Address | Decimals |
|-------|---------|----------|
| USDC | `0xf010f12dcA0b96D2d6685bf4dB3dbB4Ad500B6Ad` | 6 |
| USDT | `0xaB082b8ad96c7f47ED70ED971Ce2116469954cFB` | 6 |
| DAI | `0x0726959d22361B79e4D50A5D157b044A83eC870d` | 18 |

## Settlement Methods

The facilitator uses the best available method for each network:

- **ERC-3009** (`transferWithAuthorization`) — single-transaction gasless transfers. Used on Base, Polygon, Avalanche, Sei, XDC, XRPL EVM. Supports EOA, EIP-1271 (smart contract wallets), and EIP-6492 (counterfactual wallets).
- **EIP-2612** (`permit` + `transferFrom`) — two-step fallback for tokens without ERC-3009.
- **Permit2** (Coinbase x402 spec, `assetTransferMethod = "permit2"`) — used on Shibarium, where bridged ERC-20 tokens lack EIP-3009. Settles through the canonical [`x402ExactPermit2Proxy`](https://github.com/coinbase/x402/blob/main/specs/schemes/exact/scheme_exact_evm.md) at `0x402085c248EeA27D92E8b30b2C58ed07f9E20001`, which calls Uniswap's Permit2 (`0x000000000022D473030F116dDEE9F6B43aC78BA3`) `permitWitnessTransferFrom` with a canonical `Witness(address to, uint256 validAfter)`. The witness is enforced on-chain by the proxy, preventing the facilitator from redirecting funds. When the payload includes an EIP-2612 `permit_2612`, the facilitator routes to `settleWithPermit` so Permit2 approval and transfer happen in a single transaction.
- **Native token** — verifies already-submitted on-chain transactions for native coin transfers (ETH, BONE, AVAX, etc.).
- **SPL Token Transfer** — Solana-native token transfer with compute budget management.

## Quick Start

### 1. Configure environment

Copy `.env.example` and fill in your values:

```bash
cp .env.example .env
```

```env
HOST=0.0.0.0
PORT=8080

# Enable the networks you want (add/remove as needed)
RPC_URL_BASE_SEPOLIA=https://sepolia.base.org
RPC_URL_BASE=https://mainnet.base.org
# RPC_URL_POLYGON=https://polygon-rpc.com
# RPC_URL_SHIBARIUM=https://www.shibrpc.com
# RPC_URL_SHIBARIUM_PUPPYNET=https://puppynet.shibrpc.com

# Signer
SIGNER_TYPE=private-key
EVM_PRIVATE_KEY=0x<your-private-key>
# SOLANA_PRIVATE_KEY=<base58-encoded-keypair>

RUST_LOG=info
```

> **Note:** Only networks with configured RPC URLs will be available. Shibarium mainnet is commented out by default — uncomment `RPC_URL_SHIBARIUM` when ready.

### 2. Run with Docker

```bash
docker build -t x402-facilitator .
docker run --env-file .env -p 8080:8080 x402-facilitator
```

### 3. Run from source

```bash
cargo build --release
cargo run --release
```

The server starts at `http://localhost:8080`.

### 4. Point your app to the facilitator

Configure any x402-compatible client or SDK to use your facilitator URL. See [compatible clients](#compatible-clients) below.

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/verify` | Verify a payment signature off-chain (no state changes) |
| `POST` | `/settle` | Execute on-chain settlement |
| `GET` | `/supported` | List supported payment schemes and networks |
| `GET` | `/health` | Health check (same as `/supported`) |

### Verify

Validates a payment payload by simulating the on-chain call via `eth_call` (EVM) or transaction simulation (Solana). Returns whether the signature and amounts are valid without executing any transfers.

### Settle

Submits the actual on-chain transaction. For ERC-3009, this is a single `transferWithAuthorization` call. For Permit2, it's a `permitTransferFrom`. For EIP-2612, it's a `permit` followed by `transferFrom`.

## Compatible Clients

Works with all x402-compatible clients and SDKs:

- [x402 Payment Link](https://www.x402.org/) — Stripe-like payment links (recommended)
- [Coinbase Python SDK](https://github.com/coinbase/x402-python)
- [Coinbase TypeScript SDK](https://github.com/coinbase/x402-typescript)
- Starter templates: [x402-starter-kit](https://github.com/coinbase/x402-starter-kit) | [create-x402](https://github.com/coinbase/create-x402)

## Configuration Reference

### Required

| Variable | Description |
|----------|-------------|
| `SIGNER_TYPE` | Signer type (`private-key`) |
| `EVM_PRIVATE_KEY` | Hex-encoded private key for EVM chains (supports comma-separated for multiple keys) |
| `SOLANA_PRIVATE_KEY` | Base58-encoded keypair for Solana (required only if using Solana) |

### Network RPC URLs

| Variable | Network |
|----------|---------|
| `RPC_URL_BASE_SEPOLIA` | Base Sepolia testnet |
| `RPC_URL_BASE` | Base mainnet |
| `RPC_URL_POLYGON` | Polygon mainnet |
| `RPC_URL_POLYGON_AMOY` | Polygon Amoy testnet |
| `RPC_URL_AVALANCHE` | Avalanche C-Chain mainnet |
| `RPC_URL_AVALANCHE_FUJI` | Avalanche Fuji testnet |
| `RPC_URL_SEI` | Sei mainnet |
| `RPC_URL_SEI_TESTNET` | Sei testnet |
| `RPC_URL_SHIBARIUM` | Shibarium mainnet |
| `RPC_URL_SHIBARIUM_PUPPYNET` | Shibarium Puppynet testnet |
| `RPC_URL_SOLANA` | Solana mainnet |
| `RPC_URL_SOLANA_DEVNET` | Solana devnet |

### Optional

| Variable | Description | Default |
|----------|-------------|---------|
| `HOST` | HTTP bind address | `0.0.0.0` |
| `PORT` | HTTP port | `8080` |
| `RUST_LOG` | Log level (`info`, `debug`, `trace`) | — |

### Solana Compute Budget (Optional)

| Variable | Description | Default |
|----------|-------------|---------|
| `X402_SOLANA_MAX_COMPUTE_UNIT_LIMIT_SOLANA` | Max compute units (mainnet) | `400000` |
| `X402_SOLANA_MAX_COMPUTE_UNIT_LIMIT_SOLANA_DEVNET` | Max compute units (devnet) | `200000` |
| `X402_SOLANA_MAX_COMPUTE_UNIT_PRICE_SOLANA` | Max price in microlamports (mainnet) | `1000000` |
| `X402_SOLANA_MAX_COMPUTE_UNIT_PRICE_SOLANA_DEVNET` | Max price in microlamports (devnet) | `100000` |

## Observability

The facilitator supports OpenTelemetry-compatible traces and metrics. To enable, set:

```env
OTEL_EXPORTER_OTLP_ENDPOINT=https://api.honeycomb.io:443
OTEL_EXPORTER_OTLP_HEADERS=x-honeycomb-team=your_api_key,x-honeycomb-dataset=x402
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

Works with Honeycomb, Prometheus, Grafana, Jaeger, and other OTLP-compatible backends.

## Development

**Prerequisites:** Rust 1.80+

```bash
# Build
cargo build

# Run
cargo run

# Run with debug logging
RUST_LOG=debug cargo run
```

## Architecture

```
Client → POST /verify or /settle
              ↓
         HTTP Handler (Axum)
              ↓
         Facilitator trait
              ↓
    ┌─────────┴──────────┐
    EvmProvider      SolanaProvider
    ├─ ERC-3009      ├─ SPL Token
    ├─ EIP-2612      └─ Compute Budget
    ├─ Permit2
    └─ Native Token
```

- **Verify** simulates the transfer off-chain (`eth_call` / tx simulation) — no gas spent
- **Settle** submits the real transaction on-chain
- Providers are lazily initialized based on configured RPC URLs
- Multiple EVM private keys supported with round-robin selection

## Related Resources

- [x402 Protocol Documentation](https://www.x402.org/)
- [x402 Overview by Coinbase](https://docs.cdp.coinbase.com/x402/docs/welcome)
- [Facilitator Documentation](https://docs.cdp.coinbase.com/x402/docs/facilitator)

## License

Apache-2.0
