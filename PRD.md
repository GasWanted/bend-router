# Bend Route — Parallel DEX Routing Optimizer

## Overview

Bend Route is an open-source DEX aggregation routing engine built on Bend/HVM2. It finds optimal swap routes across decentralized exchanges by leveraging HVM's automatic parallelism to explore the full route tree — multi-hop paths, split routes, and cross-DEX combinations — faster and more thoroughly than existing solutions that rely on heuristic pruning.

## Problem

When a user swaps tokens through a DEX aggregator, the router must solve a combinatorial optimization: find the route (or combination of routes) that maximizes output for the given input, accounting for:

- Price impact on each AMM's bonding curve (nonlinear)
- Gas costs per hop
- Thousands of pools across dozens of DEXs
- Multi-hop paths (A→B→C might beat A→C)
- Split routing (sending 40% through route 1, 60% through route 2)

Current solutions are limited:

- **1inch Pathfinder** — Fast but uses aggressive heuristic pruning. Provably suboptimal on complex routes. Closed source.
- **CowSwap/CoW Protocol** — Batch auction model, different approach. Good for MEV protection but not optimized for real-time single-swap routing.
- **Paraswap** — Open source router but single-threaded route evaluation. Misses multi-hop opportunities.
- **Uniswap Smart Order Router** — Only routes through Uniswap pools. No cross-DEX optimization.

The core issue: route exploration is a tree search that explodes combinatorially. With 5,000 pools, 3 max hops, and split routing, the search space has billions of candidates. Everyone prunes aggressively, which means everyone misses better routes. The difference between a good route and the optimal route on a $1M swap can be $1,000-$10,000.

## Vision

A routing engine that explores the full route space rather than pruning to a subset, finding provably better routes by throwing parallel compute at the problem instead of accepting heuristic shortcuts. Open source so any aggregator, wallet, or protocol can integrate it.

## Goals

1. **Better routes** — Find routes that yield 0.1-0.5% more output than 1inch on swaps >$10K by exploring the full search space instead of pruning
2. **Real-time latency** — Return optimal routes in <50ms on a 16-core machine, suitable for interactive UIs and MEV-speed applications
3. **Multi-chain support** — Ethereum, Arbitrum, Optimism, Base, Polygon, BSC at launch with a pluggable pool adapter interface
4. **DEX-agnostic** — Support any AMM model: constant product (Uni V2), concentrated liquidity (Uni V3), StableSwap (Curve), weighted pools (Balancer), custom curves
5. **Embeddable** — Ship as a Rust library with C FFI, usable from any language. Optional standalone API server.

## Non-Goals

- Building a frontend/aggregator UI
- Order execution or transaction submission
- MEV protection (complementary to, not replacing, private mempools)
- Limit orders, DCA, or other advanced order types in v1
- Becoming a liquidity source (we route to pools, we don't hold liquidity)

## Architecture

```
                    ┌────────────────────────────────┐
                    │          API / Library          │
                    │   REST server or Rust crate     │
                    └───────────────┬────────────────┘
                                    │
                    ┌───────────────▼────────────────┐
                    │        Pool State Cache         │
                    │  In-memory, updated per block   │
                    │  Reserves, ticks, fees, etc.    │
                    │           (Rust)                │
                    └───────────────┬────────────────┘
                                    │
                    ┌───────────────▼────────────────┐
                    │       Route Tree Builder        │
                    │  Enumerate candidate paths      │
                    │           (Rust)                │
                    └───────────────┬────────────────┘
                                    │
              ┌─────────────────────▼─────────────────────┐
              │           Parallel Route Evaluator         │
              │                  (Bend)                    │
              │                                            │
              │  ┌──────────┐ ┌──────────┐ ┌──────────┐   │
              │  │ Path A   │ │ Path B   │ │ Path N   │   │
              │  │ A→B→C    │ │ A→D→C    │ │ A→E→F→C  │   │
              │  │ eval AMM │ │ eval AMM │ │ eval AMM │   │
              │  └────┬─────┘ └────┬─────┘ └────┬─────┘   │
              │       └────────────┼────────────┘          │
              │                    ▼                        │
              │          Split Optimizer                    │
              │   Find optimal % allocation across         │
              │   top-K routes (convex optimization)       │
              └─────────────────────┬─────────────────────┘
                                    │
                    ┌───────────────▼────────────────┐
                    │       Result + Calldata         │
                    │  Best route + encoded tx data   │
                    │           (Rust)                │
                    └────────────────────────────────┘

  FFI boundary: Rust handles I/O, pool state, calldata encoding
  Bend handles: route evaluation, AMM math, split optimization
```

## Core Components

### 1. Pool State Cache (Rust)

Maintains current state of all relevant pools across supported chains:

- Subscribes to new blocks via WebSocket/IPC
- Decodes pool state changes (reserves, liquidity, ticks, fees)
- Stores in memory as a graph: nodes = tokens, edges = pools
- Update latency target: <100ms after new block

Supported pool types:

| Type | DEXs | State needed |
|------|------|-------------|
| Constant product (x*y=k) | Uniswap V2, Sushiswap, PancakeSwap | reserve0, reserve1, fee |
| Concentrated liquidity | Uniswap V3, Pancake V3 | sqrtPriceX96, liquidity, tick bitmap, ticks |
| StableSwap | Curve | balances[], A parameter, fee |
| Weighted pools | Balancer V2 | balances[], weights[], fee |
| Custom/pluggable | Any | Adapter interface: `fn quote(pool, amount_in) -> amount_out` |

### 2. Route Tree Builder (Rust)

Generates the candidate route tree:

```
Input: (token_in, token_out, amount, max_hops=3)

Output: Tree of candidate paths
  ETH → USDC
  ├── direct: [Pool_UniV3_0.05%, Pool_UniV3_0.3%, Pool_Curve_ETH_USDC, ...]
  ├── 2-hop via WBTC: [ETH→WBTC pools] × [WBTC→USDC pools]
  ├── 2-hop via DAI: [ETH→DAI pools] × [DAI→USDC pools]
  ├── 2-hop via USDT: ...
  └── 3-hop: [ETH→X pools] × [X→Y pools] × [Y→USDC pools]
```

Pruning (before sending to Bend):
- Skip pools with <$1K liquidity
- Skip intermediate tokens with <3 pools
- Cap at configurable max candidates (default: 10,000 paths)

This pruning is coarse and safe — it eliminates obviously bad paths, not potentially good ones. The fine-grained evaluation happens in Bend.

### 3. Parallel Route Evaluator (Bend)

The core engine. Evaluates all candidate paths simultaneously:

```
-- AMM math: pure functions, no I/O
eval_univ2 : Reserves -> Amount -> Amount
eval_univ2 (r0, r1, fee) amount_in =
  let amount_with_fee = amount_in * (10000 - fee)
  let numerator = amount_with_fee * r1
  let denominator = r0 * 10000 + amount_with_fee
  numerator / denominator

eval_univ3 : TickState -> Amount -> Amount
eval_univ3 state amount_in =
  -- Step through tick ranges, consuming input amount
  -- Each tick crossing is a pure state transition
  fold_ticks state amount_in

-- Route evaluation: chain AMM evaluations
eval_route : List Pool -> Amount -> Amount
eval_route [] remaining = remaining
eval_route (pool :: rest) amount =
  let out = eval_pool pool amount
  eval_route rest out

-- Top-level: evaluate ALL routes in parallel
-- HVM distributes these across all cores automatically
eval_all : List Route -> Amount -> List (Route, Amount)
eval_all routes input =
  map (λr. (r, eval_route r.pools input)) routes
```

Each route evaluation is completely independent — no shared state, no locks, no coordination. HVM's scheduler distributes evaluations across cores.

### 4. Split Optimizer (Bend)

After evaluating individual routes, find the optimal split across top routes:

The problem: given routes R1...Rk with evaluation functions f1...fk, find allocation percentages p1...pk (summing to 1) that maximize `f1(p1 * input) + f2(p2 * input) + ... + fk(pk * input)`.

This is a convex optimization (AMM curves have diminishing returns). Solved by:

1. Evaluate each top route at multiple input sizes (10%, 20%, ..., 100%) — all in parallel
2. Build piecewise-linear approximation of each route's output curve
3. Greedy allocation: repeatedly allocate the next marginal unit to whichever route has the best marginal return

The parallel evaluation of routes × input sizes creates a 2D grid of independent computations — maximum parallelism.

### 5. Calldata Encoder (Rust)

Encodes the optimal route as executable transaction calldata:
- Direct swap calldata for single-route results
- Multi-call/batch calldata for split routes
- Supports encoding for major aggregator contracts or direct DEX router calls
- Includes gas estimation

## Multi-Chain Support

Each chain runs as an independent instance with its own pool cache:

| Chain | Priority | DEXs at launch |
|-------|----------|----------------|
| Ethereum | P0 | Uniswap V2/V3, Curve, Balancer, Sushiswap |
| Arbitrum | P0 | Uniswap V3, Camelot, GMX, Curve |
| Base | P0 | Aerodrome, Uniswap V3, Curve |
| Optimism | P1 | Velodrome, Uniswap V3, Curve |
| Polygon | P1 | QuickSwap, Uniswap V3, Curve, Balancer |
| BSC | P2 | PancakeSwap V2/V3, Thena, BiSwap |

Adding a new chain requires:
1. Pool adapter for each DEX (implements `fn quote(pool, amount_in) -> amount_out`)
2. Pool state subscriber (block listener + state decoder)
3. Token list + pool registry

## Performance Targets

Benchmarked on a 16-core machine, Ethereum mainnet pools:

| Scenario | 1inch API (observed) | Bend Route (target) |
|----------|---------------------|---------------------|
| Simple swap (ETH→USDC, top pools only) | ~50ms | <10ms |
| Complex swap (rare token, 3-hop) | ~200ms | <30ms |
| Large swap ($1M+, split routing needed) | ~300ms | <50ms |
| Batch: 100 swaps | ~10-30s | <500ms |

Route quality (measured as output amount):

| Scenario | 1inch | Bend Route (target) |
|----------|-------|---------------------|
| Standard swap ($10K) | baseline | +0.05-0.1% |
| Large swap ($100K+) | baseline | +0.1-0.3% |
| Exotic pair (low liquidity) | baseline | +0.2-0.5% |

The quality improvement comes from exploring routes that 1inch's heuristics prune.

## Integration Modes

### As a library (primary)

```rust
use bend_route::{Router, ChainConfig};

let router = Router::new(ChainConfig::ethereum(rpc_url));
router.sync().await; // initial pool state sync

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
println!("Calldata: 0x{}", hex::encode(&route.calldata));
```

### As an API server

```
GET /v1/route?chain=ethereum&tokenIn=0xC02...&tokenOut=0xA0b...&amountIn=100000000000000000000&maxHops=3

{
  "amountOut": "196200000000",
  "route": [...],
  "splits": [{"route": 0, "percent": 40}, {"route": 1, "percent": 60}],
  "gas": 250000,
  "calldata": "0x..."
}
```

## Milestones

### M1: Single-chain MVP (months 1-2)

- Uniswap V2 + V3 pool support on Ethereum
- Route tree builder with configurable max hops
- Parallel route evaluator in Bend (no split optimization)
- CLI: `bend-route quote --in WETH --out USDC --amount 1.0`
- Benchmark against 1inch API on 100 historical swaps

### M2: Full routing engine (months 3-4)

- Split route optimization
- Curve + Balancer pool support
- Gas-aware route scoring
- Rust library crate with async API
- REST API server
- Benchmark suite with daily regression against 1inch

### M3: Multi-chain + ecosystem (months 5-6)

- Arbitrum, Base, Optimism support
- Pluggable pool adapter interface
- Calldata encoder for direct execution
- WebSocket pool state streaming (real-time updates, no polling)
- npm/Python bindings via C FFI

### M4: Optimization + advanced features (months 7-9)

- Polygon, BSC support
- Route caching (same-block duplicate queries)
- Predictive routing (pre-compute routes for popular pairs)
- Multi-input routing (swap ETH+DAI → USDC in one operation)
- MEV-aware routing (flag routes likely to be sandwiched)

## Success Metrics

- **Route quality:** Measurably better output amounts than 1inch on >60% of swaps >$10K
- **Latency:** p99 <100ms for any single-chain route request
- **Adoption:** 3+ aggregator integrations within 6 months of launch
- **Coverage:** Support pools covering >90% of DEX volume on each supported chain

## Open Source Strategy

- License: MIT
- Core engine, all pool adapters, and API server are fully open source
- Benchmark suite is public and runs daily against competing routers
- Community-contributed pool adapters for long-tail DEXs
- Integration guides for wallets, aggregators, and MEV searchers

## Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Pool state staleness (using stale reserves) | Bad routes, failed txs | Sub-block state updates via pending tx mempool monitoring |
| Bend numeric precision (AMM math needs exact uint256) | Wrong output amounts | Use Rust FFI for all uint256 arithmetic, Bend only for orchestration |
| 1inch / competitors improve to close the gap | Reduced differentiation | Speed AND quality. Being open source is itself a moat for integrations |
| Pool adapter maintenance burden (DEXs upgrade) | Stale adapters | Community contribution model, automated adapter testing against on-chain |
| HVM2 overhead for small route trees | Slower than native Rust for simple swaps | Fast path: small trees (<100 routes) evaluated in Rust, Bend only for large trees |

## Technical Unknowns

1. **Uni V3 tick math in Bend** — Concentrated liquidity evaluation requires iterating through tick ranges. Need to determine if this loop structure maps well to HVM reduction or requires Rust FFI.
2. **Optimal tree size threshold** — At what candidate count does Bend's parallelism outperform single-threaded Rust? Need empirical data to set the crossover point.
3. **Memory consumption** — 10,000 route evaluations in parallel, each carrying pool state. Need to measure HVM's memory footprint and potentially implement batched evaluation.
4. **State consistency** — If pool state updates mid-computation, routes may be invalid. Need to snapshot state at computation start.
