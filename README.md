# x402-facilitator

> This project is forked from [x402-rs](https://github.com/x402-rs/x402-rs) in Oct 2025. The source code is released under the same Apache 2.0 license.

The **x402 facilitator project** aims to create universal x402 payment infrastructure for both humans and machines (AI agents). It will support the x402 payment protocol across

* all blockchains.
* all fungible tokens and coins, including all ERC-20 compatible tokens.
* traditional credit card payment networks.
* traditional bank transfers.

The initial focus is to support USDC and USDT stablecoins across blockchains.

### Shibarium support

This fork adds **Shibarium** (chain ID 109, `eip155:109`) and **Puppynet testnet** (chain ID 157, `eip155:157`) as supported networks.

Shibarium's stablecoins are bridged ERC-20 tokens that do **not** implement EIP-3009 (`transferWithAuthorization`). Instead, settlement uses [Uniswap's Permit2](https://github.com/Uniswap/permit2) contract — already deployed on Shibarium at the canonical address `0x000000000022D473030F116dDEE9F6B43aC78BA3` — which provides signature-based token transfers via `permitTransferFrom` with a witness pattern. This preserves x402's trust-minimization guarantee by locking funds to the intended recipient in the signed message.

**Shibarium token addresses:**

| Token | Address |
|-------|---------|
| USDC | `0xf010f12dcA0b96D2d6685bf4dB3dbB4Ad500B6Ad` |
| USDT | `0xaB082b8ad96c7f47ED70ED971Ce2116469954cFB` |
| DAI | `0x0726959d22361B79e4D50A5D157b044A83eC870d` |

**To enable Shibarium**, uncomment the RPC URL in your `.env` file:

```dotenv
# Shibarium mainnet (currently commented out — uncomment to enable)
RPC_URL_SHIBARIUM=https://www.shibrpc.com

# Shibarium Puppynet testnet
RPC_URL_SHIBARIUM_PUPPYNET=https://puppynet.shibrpc.com
```

> **Note:** The Shibarium livenet RPC URL is commented out by default in `.env.example`. Uncomment `RPC_URL_SHIBARIUM` when you are ready to accept payments on Shibarium mainnet.

## Current software release

* Rust crate for [x402-facilitator](https://crates.io/crates/x402-facilitator)
* [Documentation](https://docs.rs/x402-facilitator/0.11.0/x402_facilitator/)
* Demo: [payment link](https://pay.x402labs.dev/demo) | [screencast](https://youtube.com/shorts/5l6WhjIHk1A)

## Supported clients

The **x402-facilitator** works with all x402-compatible clients, SDKs, and middleware. Just configure them to use your own facilitator server (see below).

* RECOMMENDED: [x402 payment link](https://github.com/second-state/x402-payment-link) similiar to Stripe payment links
* The Coinbase SDKs for [Python](https://github.com/coinbase/x402/tree/main/examples/python) and [Typescript](https://github.com/coinbase/x402/tree/main/examples/typescript)
* Starter project templates: [x402-starter-kit](https://github.com/dabit3/x402-starter-kit) | [create-x402](https://github.com/dabit3/create-x402)

## Usage

### 1. Provide environment variables

Create a `.env` file or set environment variables directly. Example `.env`:

```dotenv
HOST=0.0.0.0
PORT=8080
RPC_URL_BASE_SEPOLIA=https://sepolia.base.org
RPC_URL_BASE=https://mainnet.base.org
SIGNER_TYPE=private-key
EVM_PRIVATE_KEY=0xdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef
SOLANA_PRIVATE_KEY=6ASf5EcmmEHTgDJ4X4ZT5vT6iHVJBXPg5AN5YoTCpGWt
RUST_LOG=info
```

**Important:**
The supported networks are determined by which RPC URLs you provide:
- If you set only `RPC_URL_BASE_SEPOLIA`, then only Base Sepolia network is supported.
- If you set both `RPC_URL_BASE_SEPOLIA` and `RPC_URL_BASE`, then both Base Sepolia and Base Mainnet are supported.
- If an RPC URL for a network is missing, that network will not be available for settlement or verification.

### 2. Build and Run with Docker

Build a Docker image locally:
```shell
docker build -t x402-facilitator .
docker run --env-file .env -p 8080:8080 x402-facilitator
```

The container:
* Exposes port `8080` (or a port you configure with `PORT` environment variable).
* Starts on `http://localhost:8080` by default.
* Requires minimal runtime dependencies (based on `debian:bullseye-slim`).

### 3. Point your application to your Facilitator

If you are building an x402-powered application, update the Facilitator URL to point to your self-hosted instance. An example is the [x402 payment link](https://github.com/second-state/x402-payment-link) project.

> ℹ️ **Tip:** For production deployments, ensure your Facilitator is reachable via HTTPS and protect it against public abuse.


## Configuration

The service reads configuration via `.env` file or directly through environment variables.

Available variables:

* `RUST_LOG`: Logging level (e.g., `info`, `debug`, `trace`),
* `HOST`: HTTP host to bind to (default: `0.0.0.0`),
* `PORT`: HTTP server port (default: `8080`),
* `SIGNER_TYPE` (required): Type of signer to use. Only `private-key` is supported now,
* `EVM_PRIVATE_KEY` (required): Private key in hex for EVM networks, like `0xdeadbeef...`,
* `SOLANA_PRIVATE_KEY` (required): Private key in hex for Solana networks, like `0xdeadbeef...`,
* `RPC_URL_BASE_SEPOLIA`: Ethereum RPC endpoint for Base Sepolia testnet,
* `RPC_URL_BASE`: Ethereum RPC endpoint for Base mainnet,
* `RPC_URL_AVALANCHE_FUJI`: Ethereum RPC endpoint for Avalanche Fuji testnet,
* `RPC_URL_AVALANCHE`: Ethereum RPC endpoint for Avalanche C-Chain mainnet.
* `RPC_URL_SOLANA`: RPC endpoint for Solana mainnet.
* `RPC_URL_SOLANA_DEVNET`: RPC endpoint for Solana devnet.
* `RPC_URL_POLYGON`: RPC endpoint for Polygon mainnet.
* `RPC_URL_POLYGON_AMOY`: RPC endpoint for Polygon Amoy testnet.
* `RPC_URL_SEI`: RPC endpoint for Sei mainnet.
* `RPC_URL_SEI_TESTNET`: RPC endpoint for Sei testnet.
* `RPC_URL_SHIBARIUM`: RPC endpoint for Shibarium mainnet (uses Permit2 for settlement).
* `RPC_URL_SHIBARIUM_PUPPYNET`: RPC endpoint for Shibarium Puppynet testnet.


## Observability

The facilitator emits [OpenTelemetry](https://opentelemetry.io)-compatible traces and metrics to standard endpoints,
making it easy to integrate with tools like Honeycomb, Prometheus, Grafana, and others.
Tracing spans are annotated with HTTP method, status code, URI, latency, other request and process metadata.

To enable tracing and metrics export, set the appropriate `OTEL_` environment variables:

```dotenv
# For Honeycomb, for example:
# Endpoint URL for sending OpenTelemetry traces and metrics
OTEL_EXPORTER_OTLP_ENDPOINT=https://api.honeycomb.io:443
# Comma-separated list of key=value pairs to add as headers
OTEL_EXPORTER_OTLP_HEADERS=x-honeycomb-team=your_api_key,x-honeycomb-dataset=x402-rs
# Export protocol to use for telemetry. Supported values: `http/protobuf` (default), `grpc`
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

The service automatically detects and initializes exporters if `OTEL_EXPORTER_OTLP_*` variables are provided.

## Development

Prerequisites:
- Rust 1.80+
- `cargo` and a working toolchain

Build locally:
```shell
cargo build
```
Run:
```shell
cargo run
```

## Related Resources

* [x402 Protocol Documentation](https://x402.org)
* [x402 Overview by Coinbase](https://docs.cdp.coinbase.com/x402/docs/overview)
* [Facilitator Documentation by Coinbase](https://docs.cdp.coinbase.com/x402/docs/facilitator)


## License

[Apache-2.0](LICENSE)
