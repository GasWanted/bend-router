# Bend Route

Parallel DEX routing optimizer built on [Bend/HVM2](https://github.com/HigherOrderCO/Bend). Explores the full route space across decentralized exchanges — multi-hop paths, split routes, and cross-DEX combinations — to find provably better swap routes.

## Why

DEX aggregators prune the route search space aggressively to stay fast. This means they miss better routes. On a $100K+ swap, the difference between a good route and the optimal route can be $100-$3,000.

Bend Route throws parallel compute at the problem instead of accepting heuristic shortcuts. HVM2's automatic parallelism lets us evaluate thousands of candidate routes simultaneously across all CPU cores — no manual threading, no locks, no coordination.

## How it works

```
Pool State Cache (Rust)     →  Live pool reserves, ticks, fees from on-chain
Route Tree Builder (Rust)   →  Enumerate candidate multi-hop paths
Route Evaluator (Bend)      →  Evaluate ALL candidates in parallel via HVM2
Split Optimizer (Bend)      →  Find optimal % allocation across top routes
Calldata Encoder (Rust)     →  Encode result as executable transaction data
```

## Supported AMMs

| Type | DEXs |
|------|------|
| Constant product (x*y=k) | Uniswap V2, SushiSwap, PancakeSwap |
| Concentrated liquidity | Uniswap V3, PancakeSwap V3 |
| StableSwap | Curve |
| Weighted pools | Balancer V2 |
| Custom | Any AMM via pluggable adapter interface |

## Multi-chain

Ethereum, Arbitrum, Base, Optimism, Polygon, BSC — each chain runs as an independent instance with its own pool cache.

## Usage

### As a library

```rust
use bend_route::{Router, ChainConfig};

let router = Router::new(ChainConfig::ethereum(rpc_url));
router.sync().await;

let route = router.find_route(FindRouteParams {
    token_in: WETH,
    token_out: USDC,
    amount_in: parse_ether("100"),
    max_hops: 3,
    max_splits: 4,
    slippage_bps: 50,
}).await?;

println!("Output: {} USDC", route.amount_out);
println!("Route: {:?}", route.path);
```

### As an API server

```
GET /v1/route?chain=ethereum&tokenIn=0xC02...&tokenOut=0xA0b...&amountIn=100000000000000000000&maxHops=3
```

## Performance targets

| Scenario | Target |
|----------|--------|
| Simple swap (ETH→USDC) | <10ms |
| Complex swap (rare token, 3-hop) | <30ms |
| Large swap ($1M+, split routing) | <50ms |

Benchmarked on a 16-core machine against Ethereum mainnet pools.

## Building

```bash
# Prerequisites: Rust 1.75+, Bend/HVM2
cargo build --release
```

## License

MIT
