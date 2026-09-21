# x402-facilitator for SHIBARIUM

Universal [x402 payment protocol](https://www.x402.org/) facilitator for humans and AI agents. Verify and settle crypto payments across multiple blockchains through a single HTTP server.

This fork adds **Shibarium (chain 109)** and **Puppynet (chain 157)** support, settling through Permit2 because Shibarium's bridged ERC-20s do not implement EIP-3009.

> Forked from [x402-rs](https://github.com/x402-rs/x402-rs). Released under Apache 2.0.

## What is x402?

The x402 protocol enables machine-to-machine and human-to-machine payments using HTTP status code 402. A **facilitator** is the settlement layer — it verifies payment signatures off-chain and executes on-chain transfers when requested.

x402 is permissionless: anyone can run a facilitator, and any app can point at any facilitator it trusts. Nothing here needs approval from Coinbase, the Shib team, or anyone else.

## Run your own facilitator

You need Docker, an RPC endpoint, and a funded wallet. Budget about five minutes.

### 1. Clone and configure

```bash
git clone https://github.com/Testingtester2/x402-facilitator
cd x402-facilitator
cp .env.example .env
```

Edit `.env`. The minimum for Shibarium mainnet:

```env
HOST=0.0.0.0
PORT=8080
SERVER_PORT=8080

# Shibarium mainnet (chain 109)
RPC_URL_SHIBARIUM=https://rpc.shibarium.shib.io

# Wallet that pays gas and submits settlements
SIGNER_TYPE=private-key
EVM_PRIVATE_KEY=0x<your-private-key>

RUST_LOG=info
```

To test first, point at Puppynet instead:

```env
RPC_URL_SHIBARIUM_PUPPYNET=https://rpc.puppynet.shib.io
```

Networks are enabled by which `RPC_URL_*` variables you set — no other configuration. Set several and the same server handles all of them.

### 2. Fund the signer wallet

The address behind `EVM_PRIVATE_KEY` submits every settlement transaction, so it needs **BONE** for gas on Shibarium. It never needs to hold the tokens being paid — those move directly from payer to recipient.

### 3. Start it

```bash
docker compose up -d --build
```

Or from source (Rust 1.85+, the crate is edition 2024):

```bash
cargo run --release
```

### 4. Check it is alive

```bash
curl -s localhost:8080/supported
```

```json
{"kinds":[{"x402Version":1,"scheme":"exact","network":"shibarium"},
          {"x402Version":1,"scheme":"native","network":"shibarium"}]}
```

If `shibarium` appears in that list, your facilitator is live and ready to settle.

### 5. Point an app at it

Any x402 client or SDK takes a facilitator URL. Give it yours instead of the default one — see [Compatible clients](#compatible-clients).

## Shibarium deployment

The contracts the Permit2 settlement path depends on are live on Shibarium mainnet at their canonical cross-chain addresses, so a client that already speaks x402 needs no Shibarium-specific code:

| Contract | Address |
|----------|---------|
| Permit2 (Uniswap) | `0x000000000022D473030F116dDEE9F6B43aC78BA3` |
| x402ExactPermit2Proxy | `0x402085c248EeA27D92E8b30b2C58ed07f9E20001` |
| Signature validator | `0xdAcD51A54883eb67D95FAEb2BBfdC4a9a6BD2a3B` |

**One thing payers must do first.** Permit2 needs a single on-chain `approve` from each payer, once per token, before their first payment:

```
USDC.approve(0x000000000022D473030F116dDEE9F6B43aC78BA3, <amount or max uint256>)
```

After that one transaction, every subsequent payment is a signature only — no gas, no further approvals. This is the same first-run step Permit2 requires on Ethereum, Base and every other chain it runs on.

## Supported networks

| Network | Chain ID | Type | Settlement method |
|---------|----------|------|-------------------|
| Shibarium | 109 | Mainnet | Permit2 |
| Shibarium Puppynet | 157 | Testnet | Permit2 |
| Base | 8453 | Mainnet | ERC-3009 / EIP-2612 |
| Base Sepolia | 84532 | Testnet | ERC-3009 / EIP-2612 |
| Polygon | 137 | Mainnet | ERC-3009 / EIP-2612 |
| Polygon Amoy | 80002 | Testnet | ERC-3009 / EIP-2612 |
| Avalanche C-Chain | 43114 | Mainnet | ERC-3009 / EIP-2612 |
| Avalanche Fuji | 43113 | Testnet | ERC-3009 / EIP-2612 |
| Sei | 1329 | Mainnet | ERC-3009 / EIP-2612 |
| Sei Testnet | 1328 | Testnet | ERC-3009 / EIP-2612 |
| XDC | 50 | Mainnet | ERC-3009 / EIP-2612 |
| XRPL EVM | 1440000 | Mainnet | ERC-3009 / EIP-2612 |
| Solana | — | Mainnet | SPL Token Transfer |
| Solana Devnet | — | Devnet | SPL Token Transfer |

## Supported tokens

**USDC** is the primary supported stablecoin, with verified contract addresses on every network above.

On Shibarium (eip155:109), Permit2 works with any standard ERC-20, so the ecosystem's bridged stablecoins are all payable:

| Token | Address | Decimals |
|-------|---------|----------|
| USDC | `0xf010f12dcA0b96D2d6685bf4dB3dbB4Ad500B6Ad` | 6 |
| USDT | `0xaB082b8ad96c7f47ED70ED971Ce2116469954cFB` | 6 |
| DAI | `0x0726959d22361B79e4D50A5D157b044A83eC870d` | 18 |

## Settlement methods

The facilitator picks the best available method for each network:

- **ERC-3009** (`transferWithAuthorization`) — single-transaction gasless transfers. Used on Base, Polygon, Avalanche, Sei, XDC, XRPL EVM. Supports EOA, EIP-1271 (smart contract wallets) and EIP-6492 (counterfactual wallets).
- **EIP-2612** (`permit` + `transferFrom`) — two-step fallback for tokens without ERC-3009.
- **Permit2** (Coinbase x402 spec, `assetTransferMethod = "permit2"`) — used on Shibarium, where bridged ERC-20s lack EIP-3009. Settles through the canonical [`x402ExactPermit2Proxy`](https://github.com/coinbase/x402/blob/main/specs/schemes/exact/scheme_exact_evm.md), which calls Permit2's `permitWitnessTransferFrom` with a canonical `Witness(address to, uint256 validAfter)`. The witness is enforced on-chain by the proxy, so **the facilitator cannot redirect your funds** — it can only execute the transfer you signed, to the recipient you signed for. When the payload includes an EIP-2612 `permit_2612`, the facilitator routes to `settleWithPermit` so the Permit2 approval and the transfer happen in one transaction.
- **Native token** — verifies already-submitted on-chain transactions for native coin transfers (ETH, BONE, AVAX, etc.).
- **SPL Token Transfer** — Solana-native token transfer with compute budget management.

## API endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/verify` | Verify a payment signature off-chain (no state changes) |
| `POST` | `/settle` | Execute on-chain settlement |
| `GET` | `/supported` | List supported payment schemes and networks |
| `GET` | `/health` | Health check (same as `/supported`) |

**Verify** validates a payment payload by simulating the on-chain call via `eth_call` (EVM) or transaction simulation (Solana). It returns whether the signature and amounts are valid without executing any transfer and without spending gas.

**Settle** submits the real transaction. For Permit2 that is `permitWitnessTransferFrom` through the proxy; for ERC-3009 a single `transferWithAuthorization`; for EIP-2612 a `permit` followed by `transferFrom`.

## Compatible clients

Works with all x402-compatible clients and SDKs:

- [x402 Payment Link](https://www.x402.org/) — Stripe-like payment links (recommended)
- [Coinbase Python SDK](https://github.com/coinbase/x402-python)
- [Coinbase TypeScript SDK](https://github.com/coinbase/x402-typescript)
- Starter templates: [x402-starter-kit](https://github.com/coinbase/x402-starter-kit) | [create-x402](https://github.com/coinbase/create-x402)

## Configuration reference

### Required

| Variable | Description |
|----------|-------------|
| `SIGNER_TYPE` | Signer type (`private-key`) |
| `EVM_PRIVATE_KEY` | Hex-encoded private key for EVM chains (comma-separated for multiple keys, used round-robin) |
| `SOLANA_PRIVATE_KEY` | Base58-encoded keypair for Solana (only if using Solana) |

At least one `RPC_URL_*` variable is also required — a facilitator with no networks enabled has nothing to settle.

### Network RPC URLs

| Variable | Network | Public endpoint |
|----------|---------|-----------------|
| `RPC_URL_SHIBARIUM` | Shibarium mainnet | `https://rpc.shibarium.shib.io` |
| `RPC_URL_SHIBARIUM_PUPPYNET` | Shibarium Puppynet testnet | `https://rpc.puppynet.shib.io` |
| `RPC_URL_BASE` | Base mainnet | `https://mainnet.base.org` |
| `RPC_URL_BASE_SEPOLIA` | Base Sepolia testnet | `https://sepolia.base.org` |
| `RPC_URL_POLYGON` | Polygon mainnet | |
| `RPC_URL_POLYGON_AMOY` | Polygon Amoy testnet | |
| `RPC_URL_AVALANCHE` | Avalanche C-Chain mainnet | |
| `RPC_URL_AVALANCHE_FUJI` | Avalanche Fuji testnet | |
| `RPC_URL_SEI` | Sei mainnet | |
| `RPC_URL_SEI_TESTNET` | Sei testnet | |
| `RPC_URL_XDC` | XDC mainnet | |
| `RPC_URL_XRPL_EVM` | XRPL EVM mainnet | |
| `RPC_URL_SOLANA` | Solana mainnet | |
| `RPC_URL_SOLANA_DEVNET` | Solana devnet | |

### Optional

| Variable | Description | Default |
|----------|-------------|---------|
| `HOST` | HTTP bind address | `0.0.0.0` |
| `PORT` | HTTP port inside the process | `8080` |
| `SERVER_PORT` | Host port published by `docker compose` | — |
| `RUST_LOG` | Log level (`info`, `debug`, `trace`) | — |

### Solana compute budget (optional)

| Variable | Description | Default |
|----------|-------------|---------|
| `X402_SOLANA_MAX_COMPUTE_UNIT_LIMIT_SOLANA` | Max compute units (mainnet) | `400000` |
| `X402_SOLANA_MAX_COMPUTE_UNIT_LIMIT_SOLANA_DEVNET` | Max compute units (devnet) | `200000` |
| `X402_SOLANA_MAX_COMPUTE_UNIT_PRICE_SOLANA` | Max price in microlamports (mainnet) | `1000000` |
| `X402_SOLANA_MAX_COMPUTE_UNIT_PRICE_SOLANA_DEVNET` | Max price in microlamports (devnet) | `100000` |

## Observability

The facilitator emits OpenTelemetry-compatible traces and metrics. To enable:

```env
OTEL_EXPORTER_OTLP_ENDPOINT=https://api.honeycomb.io:443
OTEL_EXPORTER_OTLP_HEADERS=x-honeycomb-team=your_api_key,x-honeycomb-dataset=x402
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

Works with Honeycomb, Prometheus, Grafana, Jaeger and other OTLP-compatible backends.

## Development

**Prerequisites:** Rust 1.85+ (edition 2024)

```bash
cargo build               # build
cargo run                 # run
RUST_LOG=debug cargo run  # run with debug logging
cargo fmt && cargo clippy # format and lint
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
- Providers are lazily initialized from the configured RPC URLs
- Multiple EVM private keys are supported with round-robin selection

## Troubleshooting

| Symptom | Cause |
|---------|-------|
| `shibarium` missing from `/supported` | `RPC_URL_SHIBARIUM` not set, or the RPC is unreachable |
| `env SIGNER_TYPE not set` at startup | `.env` was not created, or Docker was not given `--env-file .env` |
| Settlement reverts on allowance | The payer has not yet approved Permit2 on that token (see above) |
| Settlement fails to broadcast | The signer wallet is out of BONE |
| `docker compose up` publishes no port | `SERVER_PORT` is unset in `.env` |

## Related resources

- [x402 Protocol Documentation](https://www.x402.org/)
- [x402 Overview by Coinbase](https://docs.cdp.coinbase.com/x402/docs/welcome)
- [Facilitator Documentation](https://docs.cdp.coinbase.com/x402/docs/facilitator)
- [Shibarium docs](https://docs.shib.io/)

## License

Apache-2.0
