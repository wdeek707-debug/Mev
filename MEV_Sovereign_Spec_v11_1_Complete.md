# MEV Sovereign Unified Specification v11.1

*Complete Hardened Specification and Reference Implementation - Arbitrum One*

## 1. Status and scope

This is the single authoritative document. It replaces v10.5 (22 pages), v10.6 and the earlier v11 draft. It contains: threat model, architecture, economics and models, normative requirements, verification requirements, the complete reference implementation (every file the v10.5 blueprint listed, corrected, plus the Rust math, feed decoder and a Solidity math library), the list of corrections made to v10.5, the unverified-claims register, release gates and open decisions.

> **Note:** Honesty notes. (1) No code in this document was compiled, run, fuzzed or audited; it is a reference to build and test, not a certified artifact. (2) The v10.5 statement "Zero ungrounded claims" and the label "Certora/Spearbit-Hardened" are withdrawn; see section 9. (3) Numeric thresholds are proposed defaults to be tuned against measurements. (4) v10.5 never specified how an on-chain event becomes an ExecutionSignal; v11.1 adds a reference strategy layer in section 12 (spokes, sizing, risk). It makes the system complete as a design; it does not prove profitability.

Requirement keywords MUST / SHOULD are used in the RFC sense. A requirement with no passing verification is not met.

## 2. Threat model

| Item | Definition |
|---|---|
| Assets | (a) Executor token balance (should be ~0 at rest); (b) operator hot-key ETH for gas; (c) profit recipient funds; (d) admin authority; (e) availability of the trading loop. |
| Actors | External attacker (fake pools, spoofed flash-loan callbacks, malicious tokens/routers); compromised operator key; compromised bot host; malicious or faulty data feed; operator/dev mistakes; competitors. |
| Trust boundaries | Contract trusts only: immutable constants (Balancer, Aave, routers), whitelisted pools/tokens set by admin, and the armed transient state. Bot trusts nothing from the feed until validated and simulated on forked state. |
| Worst-case targets | Operator key leaked => loss bounded by maxLoan x failed-tx gas, never principal theft (recipient and admin are not operator-controlled). Bot logic bug => circuit breakers stop it within the loss limit. Contract bug => pause, sweep, redeploy. |
| Out of scope (accepted risks) | Competition and price moves, chain/sequencer outages, upstream protocol exploits (Aave, Balancer, Uniswap, Camelot). These are bounded by caps, not eliminated. |

## 3. Architecture and spoke topology

Contracts: one MEVExecutor (section 7.16) with three flash tiers (Uniswap V3 flash swap, Balancer V2, Aave V3), a shared atomic strategy (optional Aave liquidation, up to MAX_HOPS swaps, delta-isolated profit guard) and a role model: admin (cold multisig/timelock), guardian (pause only), profitRecipient (cold, immutable), operators (hot keys, execute only). Off-chain (Rust): feed ingest -> Governor/spokes -> age gate -> flash selection -> route validation -> encode -> forked-state simulation -> guard -> bid -> lease wallet -> sign/send -> reconcile. Math has one reference model (Solidity MEVMath) mirrored in Rust.

| Spoke | Component | Role | Anchor in code |
|---|---|---|---|
| 0 | Uniswap V2 / Sushi | Constant-product pools (x*y=k) | protocolId 0, IUniV2Router |
| 1 | Uniswap V3 | Concentrated liquidity, Tier-0 flash swap | protocolId 1, uniswapV3SwapCallback |
| 2 | Camelot V3 (Algebra) | Dynamic-fee concentrated liquidity | protocolId 3, ICamelotAlgebraRouter |
| 3 | Aave V3 | Liquidation engine and Tier-2 flash loan | liquidationCall, executeOperation |
| 4 | Balancer V2 Vault | Tier-1 flash loan (fee parameter must be read on-chain) | executeViaBalancer, receiveFlashLoan |
| 5 | Sequencer feed | Real-time ingest | feed.rs, ARBITRUM_FEED_WS |
| 6 | Oracle monitoring (Chainlink / Atlas SVR) | Deviation monitoring (50 bps threshold is a parameter) | OnChainEvent::OracleUpdate |
| 7 | Backrun / closed-path engine | Partial-exit optimization | partial.rs |

## 4. Market data, economics and models

### 4.1 Claims from v10.5 and their status

| Claim | Status |
|---|---|
| Arbitrum One adopted Priority Gas Auctions with 125 ms rounds (K = 2) | [UNVERIFIED] unsourced; read primary sources. Bidding disabled until confirmed. |
| Entropy research: 87% of atomic arbitrage and 98% of liquidations bypass Timeboost; ~97% of express-lane wins by three entities | [UNVERIFIED] check the cited governance material. |
| Private sequencer mempool (no front-running) | [UNVERIFIED] check docs; note it conflicts with a competitor-fee-driven bidder. |
| Aave liquidation bonus 4.5%-10% | [UNVERIFIED] read per-asset parameters on-chain. |
| Block time 250 ms | [UNVERIFIED] confirm. |
| Atlas searcher bond 0.1 ETH | [UNVERIFIED] check onboarding guide. |

### 4.2 Unit-economics model (replaces the v10.5 tables)

All profit terms are in one unit. Per successful event: net = out_final - loan - flash_fee - (pool fees, embedded in the simulated output) - L2_gas*(base+priority) - L1_data_fee - slippage_buffer. Per failed event the cost is gas actually consumed up to the failure point at the same prices plus the L1 data fee. A ProfitGuard revert happens after all hops ran, so a guard failure costs close to a full execution, not the "~45,000 gas" claimed in v10.5 (main.rs itself assumed 280,000). Expected value per attempt = p_success * net_success - (1 - p_success) * cost_failure. Monthly break-even events = monthly_fixed_cost / (p_success * net_success - (1 - p_success) * cost_failure), valid only if that denominator is positive. Flash fees: Balancer fee and Aave premium MUST be read on-chain; a Uniswap V3 flash swap pays the pool fee tier; none is assumed zero. The v10.5 cost and revenue tables (infrastructure $0-$294/month, $180-$1,200 per event, 3-30 events/day, 1 event/month break-even) were the author's estimates without sources; they are removed and MUST be replaced by Stage-1 measurements.

### 4.3 Latency decay and opportunity age

edge_retained(dt) = exp(-lambda * dt), lambda = ln(2)/125 = 0.005545177444479562 per ms (half-life 125 ms). This is a heuristic; lambda is a tunable parameter until the ordering policy is verified. Age classes (decision D-1): Fresh 0-40 ms; Warm 41-110 ms; Aging 111-250 ms (liquidations only); otherwise Expired and dropped before simulation.

### 4.4 Bidding

max_total_bribe = gross_wei * max_bribe_bps / 10,000. max_priority_per_gas = max_total_bribe / gas_units. bid_priority = min(competitor_priority + 1,000,000 wei, max_priority_per_gas) when bidding_enabled, else 0. total_cost = (base_fee + bid_priority) * gas_units + L1_data_fee. net = gross - total_cost; execute only if net >= min_net_profit. gross MUST be converted to wei from the loan token; competitor_priority MUST be measured or 0. Bidding is OFF by default (OC-09).

### 4.5 Closed partial exit

closed_net[k] = simulate(Steps[0..k] + ClosingSwap(Token[k] -> LoanToken)) - loan_amount - cost(k + 1), with cost in loan-token units including L2 gas, L1 data fee and flash fee (v10.5 added wei to token amounts, which is only valid when the loan token is WETH and ignored the flash fee). The candidate route must respect MAX_HOPS and route continuity.

### 4.6 Optimal input for a two-pool cycle

With gamma = (10,000 - fee)/10,000 on both pools, pool A reserves (aX, aY) and pool B reserves (bY, bX): the profit-maximizing input is x* = (gamma*s - aX*bY) / (gamma*(bY + gamma*aY)), s = sqrt(aX*bX*aY*bY). In integer form x* = den*(num*s - den*aX*bY) / (num*(den*bY + num*aY)) with num = 10,000 - fee, den = 10,000 (derivation checked by hand). Integer rounding means the result can be off by a unit or two, so the neighbor search of MP-05 is mandatory. Assumes equal fees on both pools.

## 5. Normative requirements

### 5.1 Smart contract (SC)

| ID | Requirement | Verified by |
|---|---|---|
| SC-01 | Flash callbacks (V3, Balancer, Aave) MUST run only when the executor armed the expected caller and repay token in transient storage in the same tx and msg.sender equals it. All transient slots MUST be zero after every entry point. Balancer callback additionally requires single-token arrays; Aave callback requires initiator == address(this). | Unit + spoofing mocks; Halmos |
| SC-02 | V3 callback MUST require strictly opposite-sign deltas before any unsigned cast, infer the repay token from the deltas and require it equals the armed token, and the entry point MUST revert unless the callback ran exactly once. | Unit, malicious-pool mocks |
| SC-03 | After any successful execution: executor balance of repayToken == preBalance; profitRecipient received exactly finalBal - preBalance - requiredRepay, and that is >= minNetProfit. Profit MUST be computed from balances, never from router return values. | Invariant, fork, Halmos |
| SC-04 | Role separation: admin, guardian, profitRecipient are immutable and non-zero; operators change only via admin; an operator MUST NOT be able to alter any config or move funds elsewhere than profitRecipient; admin can move funds only via sweep to profitRecipient; guardian can only pause. | Access-control test over every selector; Halmos |
| SC-05 | Limits (all MUST): per-token maxLoan (0 = disabled); V3 pool whitelist; hop-token whitelist; MAX_HOPS (proposed 4); deadline horizon (proposed <= 30 s ahead); per-token minProfitFloor with minNetProfit >= floor and > 0; pause blocks every entry point. | Unit + fuzz on payload fields |
| SC-06 | Route: first tokenIn == startToken (or collateral after liquidation); each tokenIn == previous tokenOut; last tokenOut == repayToken. Liquidation only on Balancer/Aave tiers with debtAsset == startToken == repayToken; never on the V3 tier. | Unit + fuzz of random routes |
| SC-07 | Allowances: after liquidationCall the Aave allowance == 0; after each exact-input hop the router allowance == 0; approve(0)-retry path MUST work for USDT-style tokens without corrupting calldata. | USDT-style mock; invariant |
| SC-08 | Token behaviors MUST be handled or explicitly rejected: no-return-value tokens, tokens returning false, fee-on-transfer, rebasing, ERC777-style hooks (reentrancy), blacklisting tokens (revert), returndata bombs. Anything not supported MUST fail closed. | Adversarial token mocks (section 8.3) |
| SC-09 | Every config change and every execution MUST emit an event (operator/pool/token/floor/cap/pause changes; Executed with repayToken and profit). | Event assertions |
| SC-10 | No delegatecall, selfdestruct, upgradeability, or payable fallback/receive. Hard-coded protocol addresses MUST be verified against the chain by a deploy script (code exists, expected interface responds). | Static analysis; deploy-script test |
| SC-11 | Pin exact solc version (no caret), evm_version cancun, fixed optimizer settings; reproducible build; verified source on the explorer; ArbOS must support EIP-1153 (check before deploy). | Build script; CI |
| SC-12 | Admin SHOULD be a multisig behind a timelock for risk-increasing changes (raising maxLoan, adding pools/tokens, lowering floors); risk-reducing changes and pause SHOULD be immediate. | Deployment checklist |

### 5.2 Math parity (MP)

| ID | Requirement | Verified by |
|---|---|---|
| MP-01 | MEVMath (Solidity) is the single reference. Rust, and Huff/Yul if shipped, MUST match it. Mapping: Solidity revert == Rust None == Huff/Yul revert, for every case: overflow, fee > 10000 (getAmountOut), fee >= 10000 (getAmountIn), amountOut >= reserveOut, any factor > 2^128 in sqrtProduct4. Zero inputs return 0 in all languages. A pool with reserveIn == 0 MUST return 0 everywhere. | Differential fuzz |
| MP-02 | Rust generates a JSON vector file (inputs, expected outputs/None); Foundry consumes it with vm.readFile and vm.parseJson. ffi MUST be false in CI. | CI job |
| MP-03 | Vector set MUST include: 0, 1, 2, 10^k, 2^64-1, 2^128-1, 2^128, 2^128+1, 2^255, 2^256-1, all pairs/quads of these for multi-arg functions, fees {0,1,30,300,9999,10000,10001}, plus >= 1,000,000 random vectors per function in the nightly run. | Vector generator tests |
| MP-04 | sqrt: r*r <= x < (r+1)^2 checked with 512-bit arithmetic for the whole domain including x = 2^256-1. sqrtProduct/sqrtProduct4 equal an independent 512-bit reference. | Fuzz |
| MP-05 | Optimal-input: the chosen input MUST yield profit >= profit at +/-1, +/-10 and >= a coarse grid search. The closed form is never trusted without the neighbor/grid check. Output MUST be re-simulated through the exact swap functions before use. | Property test |
| MP-06 | Huff/Yul: either pass MP-01..MP-04 or are removed from the release scope. The Yul object MUST have a runtime sub-object to be deployable. | Differential fuzz or removal |

### 5.3 Off-chain (OC)

| ID | Requirement | Verified by |
|---|---|---|
| OC-01 | No code path may sign a transaction without a successful simulation verdict. Enforce in types: a VerifiedTx newtype that only the guard can construct; the signer accepts only VerifiedTx. main.rs MUST NOT construct SimulationReport. | Compile-time + unit |
| OC-02 | Simulation MUST run on forked state at the target block (not an empty DB) with the same calldata, and MUST check recipient balance delta, executor balance == pre-existing, gas ceiling and success. Simulated vs realized gas MUST be tracked, including the L1 component; revm is not assumed to reproduce Arbitrum gas exactly. | Fork replay tests |
| OC-03 | Money arithmetic MUST be checked or explicitly saturating. Release profile: overflow-checks = true. Deny clippy::arithmetic_side_effects, cast_possible_truncation, unwrap_used, expect_used, indexing_slicing in non-test code. | CI lints |
| OC-04 | Feed decoding MUST never panic and MUST handle: single tx messages, batches, nested batches, truncated data, length-prefix wraparound (use checked_add), type-byte stripping before EIP-2718 decode, unknown kinds (skip, count, alert). | Unit vectors + cargo-fuzz |
| OC-05 | Nonces: peek, commit only after the node accepts the tx, resync from the chain pending nonce (may move backwards), one in-flight tx per wallet; no gaps under rejected simulations or send failures. | Concurrency tests |
| OC-06 | Governor: panic = "unwind" in release; per-spoke time budget on the hot path; DEGRADED returns to HEALTHY on success; TRIPPED gets automatic probing after a cool-down; every transition is logged and metered. | Unit + chaos test |
| OC-07 | Keys: no raw private key held in a plain String or logged; use a keystore/KMS/hardware signer; zeroize secrets; each operator wallet holds only gas money (cap enforced by monitoring). | Review + secret-scan in CI |
| OC-08 | Startup self-check MUST fail closed unless: chain id is 42161; executor.profitRecipient() equals config; every signer is an operator on-chain; maxLoan > 0 for each traded token; pools used are whitelisted; contract code hash equals the deployed build. | Integration test |
| OC-09 | Bidding is disabled until the ordering policy is verified (section 9.3). Competitor fee MUST be measured from the feed or default to zero. Total bid plus all costs (flash fee, pool fees, L2 gas, L1 fee, slippage) MUST stay below expected gross; no hard-coded constants for competitor fee or L1 fee in production paths. | Unit + shadow data |
| OC-10 | Opportunity age uses a monotonic clock; deadlines use block time. Clock drift alarm. Age thresholds are one table shared by code and spec (decision D-1). | Unit |

### 5.4 Operations and risk (OP)

| ID | Requirement | Verified by |
|---|---|---|
| OP-01 | Automatic circuit breakers: per-spoke loss limit, global daily loss limit (in ETH), consecutive-failure limit, failed-tx ratio limit. A trip MUST pause new sends and SHOULD call pause() on-chain through the guardian key. | Chaos/drill |
| OP-02 | Reconciliation after every executed tx: realized profit and gas vs simulation. Proposed: 3 deviations beyond 20% trip the spoke and alert. | Shadow + live metrics |
| OP-03 | Alerts: executor balance != 0 at rest; operator gas low/high; recipient delta anomalies; paused state; feed lag; sim/real drift; nonce gaps; RPC divergence. | Alert tests |
| OP-04 | Runbook and drill before live funds: pause, revoke operators, sweep, rotate keys, redeploy. Time-to-pause target is measured in the drill. | Drill report |
| OP-05 | Change control: any change to contract or critical Rust re-runs all gates; no hotfix to live without green tests. | CI policy |

## 6. Formal and property-level requirements

Targets for Halmos or Certora (or documented reasons if not feasible). Pseudocode is illustrative and not compiled.

```text
// P1 (SC-03): inventory untouched
check_balance_unchanged(token, amount, payload) {
    pre = token.balanceOf(executor);
    vm.prank(operator); executor.executeViaBalancer(token, amount, payload);   // success path only
    assert token.balanceOf(executor) == pre;
}
// P2 (SC-03): only the recipient gains, and by at least minNetProfit
check_only_recipient_gains(...) {
    r0 = token.balanceOf(recipient);  ... execute ...
    assert token.balanceOf(recipient) - r0 >= payload.minNetProfit;  assert payload.minNetProfit > 0;
}
// P3 (SC-04): operators cannot touch config or redirect funds
check_operator_cannot_admin(bytes4 sel, bytes args) {
    vm.prank(operator); (ok,) = address(executor).call(abi.encodePacked(sel, args));
    if (isAdminOrGuardianSelector(sel)) assert !ok;
}
// P4 (SC-01): callbacks revert when not armed
check_callbacks_require_arming(...) { assert callback(...) reverts when transient slots are zero; }
```

### 5.5 Invariants (referenced by tests and gates)

| # | Invariant (each must be an executable test) |
|---|---|
| I-1 | A callback runs only if the executor armed EXPECTED_CALLBACK and EXPECTED_REPAY_TOKEN in the same tx; msg.sender equals the armed address; all transient slots are zero after every entry point. |
| I-2 | V3 callback deltas have strictly opposite signs before any unsigned cast; the repay token inferred from the deltas equals the armed token. |
| I-3 | The V3 swap must invoke the callback exactly once (CALLBACK_DONE), otherwise revert. |
| I-4 | Contract balance of repayToken after any successful execution == preBalance (nothing taken from, nothing added to, pre-existing inventory). |
| I-5 | profitRecipient balance delta >= minNetProfit and equals finalBal - preBalance - requiredRepay; minNetProfit > 0. |
| I-6 | Route is continuous and ends in repayToken; liquidation only on Balancer/Aave paths and only with debtAsset == startToken == repayToken. |
| I-7 | Loan amount <= maxLoan[token]; V3 pool is whitelisted; paused blocks all entry points. |
| I-8 | Operator keys cannot change admin, guardian, profitRecipient, operators, whitelists or caps. Only admin sweeps, and only to profitRecipient. |
| I-9 | After liquidationCall, allowance of debtAsset to Aave == 0. After every hop, allowance to the router == 0 for exact-input routers. |
| I-10 | Rust and Solidity math agree: same output for same input, and "None" in Rust == revert in Solidity (overflow, fee >= 10000 where applicable, amountOut >= reserveOut, factor > 2^128). |
| I-11 | Rust never signs a tx unless a revm run on forked state succeeded and the recipient delta met minNetProfit. |
| I-12 | Per-wallet nonces never skip: a nonce is consumed only after a successful broadcast. |

## 7. Reference implementation

> **Note:** Not compiled. Likely first compile issues: alloy 0.8 / revm 19 API paths and feature names (check before pinning), ruint shift/operator forms, private constants inside inline assembly, and stack depth in the Solidity callbacks. Module tree: constants, hub, partial, bidder, physics, age, flash, signer, simroom, tx, math, feed, main.

### 7.1 Cargo.toml

```text
[package]
name = "mev-sovereign"
version = "11.0.0"
edition = "2021"

[dependencies]
# Pin EXACT versions once the APIs below are confirmed to compile; feature names must be checked.
alloy = { version = "0.8", features = ["full", "node-bindings", "rpc-types-eth", "signer-local", "sol-types"] }
revm = { version = "19", features = ["std", "serde", "alloydb"] }   # alloydb = forked-state backend (verify)
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
eyre = "0.6"
tracing = "0.1"
tracing-subscriber = "0.3"
zeroize = "1"

[dev-dependencies]
proptest = "1"

[profile.release]
panic = "unwind"          # REQUIRED for catch_unwind in the Governor (OC-06)
overflow-checks = true    # OC-03
lto = true
codegen-units = 1

[lints.rust]
unsafe_code = "forbid"

[lints.clippy]            # OC-03
arithmetic_side_effects = "deny"
cast_possible_truncation = "deny"
cast_sign_loss = "deny"
unwrap_used = "deny"
expect_used = "deny"
indexing_slicing = "deny"
```

### 7.2 src/constants.rs

```text
// src/constants.rs  -- every address MUST be re-verified against official sources (OC-08).
use alloy::primitives::{address, Address};

pub const ARBITRUM_CHAIN_ID: u64 = 42161;
pub const ARBITRUM_FEED_WS: &str = "wss://arb1.arbitrum.io/feed";          // verify

pub const BALANCER_V2_VAULT: Address = address!("BA12222222228d8Ba445958a75a0704d566BF2C8");
pub const AAVE_V3_POOL:      Address = address!("794a61358D6845594F94dc1DB02A252b5b4814aD");
pub const UNISWAP_V3_ROUTER: Address = address!("E592427A0AEce92De3Edee1F18E0157C05861564");
pub const CAMELOT_V3_ROUTER: Address = address!("1F721E2E82F6676FCE4eA07A5958cF098D339e18");
pub const UNISWAP_V2_ROUTER: Address = address!("4752ba5DBc23f44D87826276BF6Fd6b1C372aD24");

// Arbitrum system contracts (verify which one is correct for each use before relying on it)
pub const ARB_GAS_INFO_ADDR:   Address = address!("000000000000000000000000000000000000006C");
pub const NODE_INTERFACE_ADDR: Address = address!("00000000000000000000000000000000000000C8");

pub const WETH: Address = address!("82aF49447D8a07e3bd95BD0d56f35241523fBab1");
pub const USDC: Address = address!("af88d065e77c8cC2239327C5EDb3A432268e5831");
pub const USDT: Address = address!("Fd086bc7CD5C481DCC9C85ebE478A1C0b69FCbb9");
pub const ARB:  Address = address!("912CE59144191C1204E64559FE8253a0e49E6548");
pub const WBTC: Address = address!("2f2a2543B76A4166549F7aaB2e75Bef0aefC5B0f");

pub const MAX_HOPS: usize = 4;                 // must equal MEVExecutor.MAX_HOPS
pub const MAX_DEADLINE_HORIZON_SECS: u64 = 30; // must equal MEVExecutor.MAX_DEADLINE_HORIZON
```

### 7.3 src/hub.rs

```text
// src/hub.rs  -- Governor with panic isolation, time budget, recovery and cool-down probing (OC-06)
use std::panic::{catch_unwind, AssertUnwindSafe};
use std::sync::atomic::{AtomicU32, AtomicU64, AtomicU8, Ordering};
use std::sync::Arc;
use std::time::Instant;
use alloy::primitives::{Address, U256};

pub const STATUS_HEALTHY: u8 = 0;
pub const STATUS_DEGRADED: u8 = 1;
pub const STATUS_TRIPPED: u8 = 2;

#[derive(Debug, Clone)]
pub enum OnChainEvent {
    SequencerSwap { pool: Address, token_in: Address, token_out: Address, amount_in: U256, timestamp_ms: u64 },
    OracleUpdate { feed: Address, asset: Address, new_price: U256, timestamp_ms: u64 },
    HealthFactorDrop { borrower: Address, collateral: Address, debt: Address, hf_wad: U256, timestamp_ms: u64 },
}

#[derive(Debug, Clone)]
pub struct SwapStep {
    pub protocol_id: u8,      // 0 = UniV2, 1 = UniV3, 3 = Camelot
    pub pool: Address,
    pub token_in: Address,
    pub token_out: Address,
    pub fee_or_tier: u32,     // UniV3 fee in hundredths of a bip (500, 3000, 10000); must fit 24 bits
    pub min_out: U256,
}

#[derive(Debug, Clone)]
pub struct LiquidationTarget {
    pub borrower: Address, pub collateral_asset: Address, pub debt_asset: Address, pub debt_to_cover: U256,
}

#[derive(Debug, Clone)]
pub struct ExecutionSignal {
    pub spoke_id: u8,
    pub loan_token: Address,
    pub loan_amount: U256,
    pub steps: Vec<SwapStep>,
    pub liquidation: Option<LiquidationTarget>,
    pub expected_profit_loan_token: U256,   // v11: profit in LOAN-TOKEN units (spokes must fill)
    pub expected_gross_profit_wei: U256,    // same profit converted to wei (spokes must convert)
    pub event_timestamp_ms: u64,
}

pub trait SpokeNode: Send + Sync {
    fn id(&self) -> u8;
    fn name(&self) -> &'static str;
    fn on_event(&self, event: &Arc<OnChainEvent>) -> Option<ExecutionSignal>;
}

pub struct SpokeWrapper {
    pub node: Box<dyn SpokeNode>,
    pub failures: AtomicU32,
    pub max_failures: u32,
    pub status: AtomicU8,
    pub tripped_at_ms: AtomicU64,
}

pub struct Governor {
    spokes: Vec<SpokeWrapper>,
    cooldown_ms: u64,      // TRIPPED spokes are probed again after this
    budget_us: u128,       // a spoke slower than this is treated as failed and its output dropped (stale)
}

impl Governor {
    pub fn new(cooldown_ms: u64, budget_us: u128) -> Self { Self { spokes: Vec::new(), cooldown_ms, budget_us } }

    pub fn register(&mut self, node: Box<dyn SpokeNode>, max_failures: u32) {
        self.spokes.push(SpokeWrapper {
            node, failures: AtomicU32::new(0), max_failures,
            status: AtomicU8::new(STATUS_HEALTHY), tripped_at_ms: AtomicU64::new(0),
        });
    }

    pub fn revive(&self, spoke_id: u8) {
        for w in &self.spokes {
            if w.node.id() == spoke_id {
                w.failures.store(0, Ordering::SeqCst);
                w.status.store(STATUS_HEALTHY, Ordering::SeqCst);
            }
        }
    }

    fn record_failure(&self, w: &SpokeWrapper, now_ms: u64) {
        let c = w.failures.fetch_add(1, Ordering::SeqCst).saturating_add(1);
        if c >= w.max_failures {
            w.tripped_at_ms.store(now_ms, Ordering::Release);
            w.status.store(STATUS_TRIPPED, Ordering::Release);
            tracing::error!(spoke = w.node.name(), "spoke TRIPPED");
        } else {
            w.status.store(STATUS_DEGRADED, Ordering::Release);
        }
    }

    pub fn broadcast(&self, event: Arc<OnChainEvent>, now_ms: u64) -> Vec<ExecutionSignal> {
        let mut signals = Vec::new();
        for w in &self.spokes {
            if w.status.load(Ordering::Acquire) == STATUS_TRIPPED {
                let since = now_ms.saturating_sub(w.tripped_at_ms.load(Ordering::Acquire));
                if since < self.cooldown_ms { continue; }
                w.status.store(STATUS_DEGRADED, Ordering::Release);       // probe after cool-down
            }
            let started = Instant::now();
            let res = catch_unwind(AssertUnwindSafe(|| w.node.on_event(&event)));
            let slow = started.elapsed().as_micros() > self.budget_us;
            match res {
                Ok(out) if !slow => {
                    w.failures.store(0, Ordering::Relaxed);
                    w.status.store(STATUS_HEALTHY, Ordering::Release);    // recovery on success
                    if let Some(sig) = out { signals.push(sig); }
                }
                _ => self.record_failure(w, now_ms),                       // panic or over budget
            }
        }
        signals
    }
}
```

### 7.4 src/partial.rs

```text
// src/partial.rs  -- closed-path / partial-exit evaluation. All amounts in LOAN-TOKEN units (C-units fix).
use alloy::primitives::{Address, U256};
use crate::hub::SwapStep;

pub struct ClosedPathCandidate { pub steps: Vec<SwapStep>, pub net_profit: U256, pub is_partial_exit: bool }

pub trait PoolQuoteEngine {
    fn simulate_swap(&self, step: &SwapStep, amount_in: U256) -> Option<U256>;
    fn find_direct_closing_step(&self, token_in: Address, loan_token: Address) -> Option<SwapStep>;
    /// Total cost of an n-step route in LOAN-TOKEN units: L2 gas + L1 data fee + flash fee
    /// (Balancer/Aave premium/V3 pool fee). The implementation converts wei -> loan token.
    fn estimate_cost_in_loan_token(&self, num_steps: usize, loan_amount: U256) -> Option<U256>;
}

pub fn evaluate_optimal_closed_path(
    loan_token: Address, loan_amount: U256, full_steps: &[SwapStep],
    engine: &dyn PoolQuoteEngine, max_steps: usize,
) -> Option<ClosedPathCandidate> {
    if full_steps.is_empty() { return None; }
    let mut best: Option<ClosedPathCandidate> = None;
    let mut cur_amt = loan_amount;
    let mut cur_token = loan_token;
    let mut prefix: Vec<SwapStep> = Vec::new();
    let total = full_steps.len();

    for (idx, step) in full_steps.iter().enumerate() {
        if prefix.len() >= max_steps { break; }
        if step.token_in != cur_token { break; }                       // broken route: stop, keep best so far
        let out = match engine.simulate_swap(step, cur_amt) { Some(v) => v, None => break };
        prefix.push(step.clone());
        cur_amt = out;
        cur_token = step.token_out;

        let is_last = idx.saturating_add(1) == total;
        let candidate: Option<(Vec<SwapStep>, U256, bool)> = if is_last {
            if cur_token == loan_token { Some((prefix.clone(), cur_amt, false)) } else { None }
        } else if idx >= 1 && prefix.len() < max_steps {
            engine.find_direct_closing_step(cur_token, loan_token).and_then(|close| {
                if close.token_in != cur_token || close.token_out != loan_token { return None; }
                let fin = engine.simulate_swap(&close, cur_amt)?;
                let mut closed = prefix.clone();
                closed.push(close);
                Some((closed, fin, true))
            })
        } else { None };

        if let Some((steps, final_out, partial)) = candidate {
            let cost = engine.estimate_cost_in_loan_token(steps.len(), loan_amount);
            let net = cost
                .and_then(|c| loan_amount.checked_add(c))
                .and_then(|required| final_out.checked_sub(required));
            if let Some(net) = net {
                if !net.is_zero() && best.as_ref().map_or(true, |b| net > b.net_profit) {
                    best = Some(ClosedPathCandidate { steps, net_profit: net, is_partial_exit: partial });
                }
            }
        }
    }
    best
}
```

### 7.5 src/bidder.rs

```text
// src/bidder.rs  -- saturating/checked bidding. Disabled by default until the ordering policy is verified (OC-09).
#[derive(Debug, Clone)]
pub struct PgaBidDecision {
    pub should_execute: bool,
    pub max_fee_per_gas_wei: u128,
    pub priority_fee_per_gas_wei: u128,
    pub expected_net_profit_wei: u128,
}
impl PgaBidDecision {
    fn reject() -> Self {
        Self { should_execute: false, max_fee_per_gas_wei: 0, priority_fee_per_gas_wei: 0, expected_net_profit_wei: 0 }
    }
}

pub struct DynamicBidder {
    pub max_bribe_bps: u128,        // cap on bribe as a share of gross, e.g. 4000 = 40%
    pub min_net_profit_wei: u128,
    pub bidding_enabled: bool,      // false => priority fee 0
}

impl DynamicBidder {
    /// gross_profit_wei MUST already be converted to wei; competitor fee MUST be measured or 0.
    pub fn evaluate_bid(&self, gross_profit_wei: u128, l2_gas_units: u128, l2_base_fee_wei: u128,
                        l1_data_fee_wei: u128, competitor_priority_fee_wei: u128) -> PgaBidDecision {
        if l2_gas_units == 0 || gross_profit_wei <= l1_data_fee_wei { return PgaBidDecision::reject(); }
        let max_total_bribe = gross_profit_wei
            .saturating_mul(self.max_bribe_bps).checked_div(10_000).unwrap_or(0).min(gross_profit_wei);
        let max_prio = max_total_bribe.checked_div(l2_gas_units).unwrap_or(0);
        let final_prio = if self.bidding_enabled {
            competitor_priority_fee_wei.saturating_add(1_000_000).min(max_prio)
        } else { 0 };
        let l2_cost = l2_base_fee_wei.saturating_add(final_prio).saturating_mul(l2_gas_units);
        let total_cost = l2_cost.saturating_add(l1_data_fee_wei);
        let net = match gross_profit_wei.checked_sub(total_cost) {
            Some(n) if n > 0 => n,
            _ => return PgaBidDecision::reject(),
        };
        let buffer = l2_base_fee_wei.saturating_mul(12).checked_div(10).unwrap_or(l2_base_fee_wei);
        PgaBidDecision {
            should_execute: net >= self.min_net_profit_wei,
            max_fee_per_gas_wei: buffer.saturating_add(final_prio),
            priority_fee_per_gas_wei: final_prio,
            expected_net_profit_wei: net,
        }
    }
}
```

### 7.6 src/physics.rs and src/age.rs

```text
// src/physics.rs  -- heuristic decay; lambda is a TUNABLE parameter until the ordering policy is verified
pub const PGA_LAMBDA_PER_MS: f64 = 0.005545177444479562;      // ln(2)/125, half-life 125 ms

#[inline]
pub fn edge_retained(delta_ms: u64) -> f64 {
    #[allow(clippy::cast_precision_loss)]
    let t = delta_ms as f64;
    (-PGA_LAMBDA_PER_MS * t).exp()
}

#[inline]
pub fn decay_adjusted_profit_wei(gross_profit_wei: u128, delta_ms: u64) -> u128 {
    #[allow(clippy::cast_precision_loss, clippy::cast_possible_truncation, clippy::cast_sign_loss)]
    {
        let v = (gross_profit_wei as f64) * edge_retained(delta_ms);   // factor <= 1, so v <= input
        if v.is_finite() && v >= 0.0 { (v as u128).min(gross_profit_wei) } else { 0 }
    }
}

// src/age.rs  -- ONE threshold table shared by code and spec (decision D-1)
pub const FRESH_MAX_MS: u64 = 40;
pub const WARM_MAX_MS: u64 = 110;
pub const LIQ_AGING_MAX_MS: u64 = 250;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum OpportunityAge { Fresh, Warm, Aging, Expired }

#[inline]
pub fn classify_opportunity_age(delta_ms: u64, is_liquidation: bool) -> OpportunityAge {
    if delta_ms <= FRESH_MAX_MS { OpportunityAge::Fresh }
    else if delta_ms <= WARM_MAX_MS { OpportunityAge::Warm }
    else if is_liquidation && delta_ms <= LIQ_AGING_MAX_MS { OpportunityAge::Aging }
    else { OpportunityAge::Expired }
}
```

### 7.7 src/flash.rs

```text
// src/flash.rs  -- flash source selection. Returns None when the route is not executable as specified.
use alloy::primitives::{Address, U256};
use crate::hub::SwapStep;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum FlashLoanProvider {
    UniswapV3ZeroHopCallback { pool: Address, zero_for_one: bool },
    BalancerV2Vault,
    AaveV3Pool,
}

#[derive(Debug, Clone)]
pub struct FlashExecutionPlan {
    pub provider: FlashLoanProvider,
    /// Tier-0: hop 0 is executed by v3Pool.swap() itself, so this is steps[1..]. Tier-1/2: all steps.
    pub remaining_hops: Vec<SwapStep>,
    /// Token actually held when the executor strategy starts (Tier-0: steps[0].token_out).
    pub start_token: Address,
}

pub fn select_optimal_flash_source(
    loan_token: Address, loan_amount: U256, steps: &[SwapStep], has_liquidation: bool,
    balancer_liquidity: U256,                    // REAL on-chain Vault balance of loan_token
    tier0_enabled: bool, v3_pool_allowed: &dyn Fn(Address) -> bool,
) -> Option<FlashExecutionPlan> {
    if steps.is_empty() { return None; }
    let closes = steps.last().map_or(false, |s| s.token_out == loan_token);

    if tier0_enabled && !has_liquidation && steps.len() >= 2 && closes {
        if let Some(first) = steps.first() {
            if first.protocol_id == 1 && first.token_in == loan_token && first.token_in != first.token_out
                && v3_pool_allowed(first.pool)
            {
                let zero_for_one = first.token_in < first.token_out;   // V3 orders token0 < token1 by address
                return Some(FlashExecutionPlan {
                    provider: FlashLoanProvider::UniswapV3ZeroHopCallback { pool: first.pool, zero_for_one },
                    remaining_hops: steps.get(1..)?.to_vec(),
                    start_token: first.token_out,
                });
            }
        }
    }
    let provider = if balancer_liquidity >= loan_amount { FlashLoanProvider::BalancerV2Vault }
                   else { FlashLoanProvider::AaveV3Pool };
    Some(FlashExecutionPlan { provider, remaining_hops: steps.to_vec(), start_token: loan_token })
}
```

### 7.8 src/signer.rs

```text
// src/signer.rs  -- single-flight wallets, lease-based nonces, no raw keys in memory structs (OC-05, OC-07)
use std::sync::atomic::{AtomicBool, AtomicU64, AtomicUsize, Ordering};
use std::sync::Arc;
use alloy::primitives::{Address, Bytes};
use crate::simroom::VerifiedTx;

/// Implemented by a keystore / KMS / hardware adapter. Never exposes the key.
pub trait SignerHandle: Send + Sync {
    fn sign(&self, tx: &VerifiedTx, nonce: u64, max_fee_per_gas: u128, priority_fee: u128) -> Option<Bytes>;
}

pub struct ManagedWallet {
    pub address: Address,
    pub signer: Arc<dyn SignerHandle>,
    pub nonce: AtomicU64,
    pub in_flight: AtomicBool,
}

pub struct SignerPool { wallets: Vec<ManagedWallet>, cursor: AtomicUsize }

/// Holding a Lease means exclusive use of the wallet. Dropping it releases the wallet WITHOUT
/// consuming the nonce (a failed or rejected send leaves no gap).
pub struct Lease<'a> { pub wallet: &'a ManagedWallet, pub nonce: u64 }
impl Drop for Lease<'_> { fn drop(&mut self) { self.wallet.in_flight.store(false, Ordering::Release); } }

impl SignerPool {
    pub fn new(wallets: Vec<ManagedWallet>) -> Self { Self { wallets, cursor: AtomicUsize::new(0) } }

    pub fn acquire(&self) -> Option<Lease<'_>> {
        let n = self.wallets.len();
        let start = self.cursor.fetch_add(1, Ordering::Relaxed);
        for off in 0..n {
            let idx = start.wrapping_add(off).checked_rem(n)?;
            let w = self.wallets.get(idx)?;
            if w.in_flight.compare_exchange(false, true, Ordering::AcqRel, Ordering::Acquire).is_ok() {
                return Some(Lease { wallet: w, nonce: w.nonce.load(Ordering::SeqCst) });
            }
        }
        None
    }

    /// Call ONLY after the node accepted the transaction.
    pub fn commit(&self, lease: Lease<'_>) {
        let used = lease.nonce;
        let _ = lease.wallet.nonce.compare_exchange(used, used.saturating_add(1), Ordering::SeqCst, Ordering::SeqCst);
        // `lease` drops here and releases the wallet.
    }

    /// Authoritative resync from the chain's PENDING nonce; may move backwards.
    pub fn resync(&self, addr: Address, chain_pending: u64) {
        for w in &self.wallets { if w.address == addr { w.nonce.store(chain_pending, Ordering::SeqCst); } }
    }

    pub fn addresses(&self) -> Vec<Address> { self.wallets.iter().map(|w| w.address).collect() }
}
```

### 7.9 src/simroom.rs

```text
// src/simroom.rs  -- simulation contract + guard + VerifiedTx seal (OC-01, OC-02)
use alloy::primitives::{Address, Bytes, U256};

pub struct UnsignedTx { pub to: Address, pub data: Bytes, pub gas_limit: u64 }

/// Produced ONLY by `Simulator` implementations, never hand-built in main.rs.
pub struct SimulationReport {
    pub success: bool,
    pub gas_used: u64,
    pub recipient: Address,                  // observed profit recipient (from state/logs)
    pub recipient_balance_before: U256,      // loan token
    pub recipient_balance_after: U256,
    pub executor_balance_after: U256,        // loan token left on the executor
}

/// Implementation requirements: revm over FORKED state at the target block (AlloyDB or equivalent),
/// non-committing transact(), same calldata and sender as production, balances read before/after.
pub trait Simulator {
    fn run(&self, block_number: u64, tx: &UnsignedTx, recipient: Address, loan_token: Address) -> Option<SimulationReport>;
}

/// Sealed: the only constructor is RuntimeInvariantGuard::verify, so the signer cannot be reached
/// without a passing simulation.
pub struct VerifiedTx { tx: UnsignedTx, _seal: () }
impl VerifiedTx {
    pub fn tx(&self) -> &UnsignedTx { &self.tx }
    pub fn calldata(&self) -> &Bytes { &self.tx.data }
}

pub struct RuntimeInvariantGuard { pub profit_recipient: Address, pub max_gas_ceiling: u64 }

impl RuntimeInvariantGuard {
    pub fn verify(&self, r: &SimulationReport, pre_existing: U256, min_net_profit: U256, tx: UnsignedTx)
        -> Option<VerifiedTx>
    {
        if !r.success || r.gas_used > self.max_gas_ceiling { return None; }
        if r.recipient != self.profit_recipient { return None; }
        if r.executor_balance_after != pre_existing { return None; }              // I-4
        if min_net_profit.is_zero() { return None; }                              // I-5
        let delta = r.recipient_balance_after.checked_sub(r.recipient_balance_before)?;
        if delta < min_net_profit { return None; }                                // I-5
        Some(VerifiedTx { tx, _seal: () })
    }
}
```

### 7.10 src/tx.rs

```text
// src/tx.rs  -- calldata encoder. Route is validated before encoding; fee tiers are rejected, never truncated.
use alloy::primitives::{Address, Bytes, Uint, I256, U160, U256};
use alloy::sol;
use alloy::sol_types::{SolCall, SolValue};
use crate::constants::MAX_HOPS;
use crate::flash::{FlashExecutionPlan, FlashLoanProvider};
use crate::hub::{ExecutionSignal, SwapStep};

pub const MIN_SQRT_RATIO_PLUS_ONE: u64 = 4_295_128_740;                      // TickMath.MIN_SQRT_RATIO + 1
// TickMath.MAX_SQRT_RATIO - 1 = 1461446703485210103287273052203988822378723970341
pub const MAX_SQRT_RATIO_MINUS_ONE_LIMBS: [u64; 3] = [0x5d951d5263988d25, 0xefd1fc6a50648849, 0x00000000fffd8963];

sol! {
    struct RouteHop { uint8 protocolId; address tokenIn; address tokenOut; uint24 feeTier; uint256 minAmountOut; }
    struct ExecutionPayload {
        bool hasLiquidation; address borrower; address collateralAsset; address debtAsset;
        uint256 debtToCover; uint256 minNetProfit; uint256 deadline; RouteHop[] hops;
    }
    interface IMEVExecutor {
        function executeViaUniV3Callback(address v3Pool, address repayToken, bool zeroForOne,
            int256 amountSpecified, uint160 sqrtPriceLimitX96, bytes calldata payloadBytes) external;
        function executeViaBalancer(address loanToken, uint256 loanAmount, bytes calldata payloadBytes) external;
        function executeViaAave(address loanToken, uint256 loanAmount, bytes calldata payloadBytes) external;
    }
}

/// Mirrors SC-06. start_token: Tier-0 = steps[0].token_out (plan.start_token), Tier-1/2 = loan token.
pub fn validate_route(loan_token: Address, start_token: Address, hops: &[SwapStep]) -> bool {
    if hops.len() > MAX_HOPS { return false; }
    let mut cur = start_token;
    for h in hops { if h.token_in != cur { return false; } cur = h.token_out; }
    cur == loan_token
}

pub fn encode_executor_calldata(
    signal: &ExecutionSignal, plan: &FlashExecutionPlan, min_net_profit: U256, deadline_unix: U256,
) -> Option<Bytes> {
    if !validate_route(signal.loan_token, plan.start_token, &plan.remaining_hops) { return None; }
    if min_net_profit.is_zero() { return None; }

    let mut hops: Vec<RouteHop> = Vec::with_capacity(plan.remaining_hops.len());
    for s in &plan.remaining_hops {
        if s.fee_or_tier > 0x00FF_FFFF { return None; }                          // reject, never truncate
        hops.push(RouteHop {
            protocolId: s.protocol_id, tokenIn: s.token_in, tokenOut: s.token_out,
            feeTier: Uint::<24, 1>::try_from(u64::from(s.fee_or_tier)).ok()?,
            minAmountOut: s.min_out,
        });
    }
    let (has_liq, borrower, col, debt, cover) = match &signal.liquidation {
        Some(l) => (true, l.borrower, l.collateral_asset, l.debt_asset, l.debt_to_cover),
        None => (false, Address::ZERO, Address::ZERO, Address::ZERO, U256::ZERO),
    };
    let payload = ExecutionPayload { hasLiquidation: has_liq, borrower, collateralAsset: col, debtAsset: debt,
        debtToCover: cover, minNetProfit: min_net_profit, deadline: deadline_unix, hops };
    let payload_bytes = Bytes::from(payload.abi_encode());

    let calldata = match plan.provider {
        FlashLoanProvider::UniswapV3ZeroHopCallback { pool, zero_for_one } => {
            if has_liq { return None; }
            let limit = if zero_for_one { U160::from(MIN_SQRT_RATIO_PLUS_ONE) }
                        else { U160::from_limbs(MAX_SQRT_RATIO_MINUS_ONE_LIMBS) };
            let amt = I256::try_from(signal.loan_amount).ok()?;
            if !amt.is_positive() { return None; }
            IMEVExecutor::executeViaUniV3CallbackCall { v3Pool: pool, repayToken: signal.loan_token,
                zeroForOne: zero_for_one, amountSpecified: amt, sqrtPriceLimitX96: limit,
                payloadBytes: payload_bytes }.abi_encode()
        }
        FlashLoanProvider::BalancerV2Vault => IMEVExecutor::executeViaBalancerCall {
            loanToken: signal.loan_token, loanAmount: signal.loan_amount, payloadBytes: payload_bytes }.abi_encode(),
        FlashLoanProvider::AaveV3Pool => IMEVExecutor::executeViaAaveCall {
            loanToken: signal.loan_token, loanAmount: signal.loan_amount, payloadBytes: payload_bytes }.abi_encode(),
    };
    Some(Bytes::from(calldata))
}

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn max_ratio_limbs_match_tickmath() {
        let v = U160::from_limbs(MAX_SQRT_RATIO_MINUS_ONE_LIMBS);
        assert_eq!(v.to_string(), "1461446703485210103287273052203988822378723970341");
    }
}
```

### 7.11 src/math.rs

```text
// src/math.rs  -- Rust mirror of Solidity MEVMath. Rule (MP-01): Solidity revert == Rust None.
use alloy::primitives::{U256, U512};

pub const BPS: u64 = 10_000;
fn bps() -> U256 { U256::from(BPS) }
fn one() -> U256 { U256::from(1u64) }

pub fn sqrt(x: U256) -> U256 {
    if x.is_zero() { return U256::ZERO; }
    let mut z = (x >> 1usize).wrapping_add(x & one());            // ceil(x/2): cannot overflow
    let mut y = x;
    while z < y { y = z; z = (x / z).wrapping_add(z) >> 1usize; } // x/z + z < 2^256 for the ceil seed
    y
}

fn mul512(a: U256, b: U256) -> U512 { U512::from(a).wrapping_mul(U512::from(b)) }  // never wraps for two U256

/// Exact floor(sqrt(v)) for v < 2^512, digit by digit (reference-grade; replace by a tested
/// integer-Newton only after differential tests prove equality).
fn sqrt512(v: U512) -> U256 {
    let mut r = U256::ZERO;
    let mut bit = one() << 255usize;
    while !bit.is_zero() {
        let cand = r | bit;
        if mul512(cand, cand) <= v { r = cand; }
        bit >>= 1usize;
    }
    r
}

pub fn sqrt_product(a: U256, b: U256) -> U256 {
    if a.is_zero() || b.is_zero() { return U256::ZERO; }
    sqrt512(mul512(a, b))
}

/// None == FactorTooLarge revert. Exact for factors <= 2^128 - 1.
pub fn sqrt_product4(a: U256, b: U256, c: U256, d: U256) -> Option<U256> {
    if a.is_zero() || b.is_zero() || c.is_zero() || d.is_zero() { return Some(U256::ZERO); }
    let max128 = (one() << 128usize).wrapping_sub(one());
    if a > max128 || b > max128 || c > max128 || d > max128 { return None; }
    Some(sqrt512(mul512(a.wrapping_mul(b), c.wrapping_mul(d))))   // each product < 2^256
}

pub fn get_amount_out(amount_in: U256, reserve_in: U256, reserve_out: U256, fee_bps: U256) -> Option<U256> {
    if fee_bps > bps() { return None; }
    if amount_in.is_zero() || reserve_in.is_zero() || reserve_out.is_zero() { return Some(U256::ZERO); }
    let aif = amount_in.checked_mul(bps().wrapping_sub(fee_bps))?;
    let num = aif.checked_mul(reserve_out)?;
    let den = reserve_in.checked_mul(bps())?.checked_add(aif)?;
    num.checked_div(den)
}

pub fn get_amount_in(amount_out: U256, reserve_in: U256, reserve_out: U256, fee_bps: U256) -> Option<U256> {
    if fee_bps >= bps() { return None; }
    if amount_out.is_zero() || reserve_in.is_zero() || reserve_out.is_zero() { return Some(U256::ZERO); }
    if amount_out >= reserve_out { return None; }
    let num = reserve_in.checked_mul(amount_out)?.checked_mul(bps())?;
    let den = reserve_out.wrapping_sub(amount_out).checked_mul(bps().wrapping_sub(fee_bps))?;
    num.checked_div(den)?.checked_add(one())
}

pub fn price_impact_bps(amount_in: U256, amount_out: U256, reserve_in: U256, reserve_out: U256) -> Option<U256> {
    if amount_in.is_zero() || reserve_in.is_zero() || reserve_out.is_zero() { return Some(U256::ZERO); }
    let num = amount_out.checked_mul(reserve_in)?;
    let den = amount_in.checked_mul(reserve_out)?;
    if num >= den { return Some(U256::ZERO); }
    den.wrapping_sub(num).checked_mul(bps())?.checked_div(den)
}

/// Closed-form optimal token0 input for a two-pool cycle (same fee on both pools).
/// x* = den*(num*s - den*aX*bY) / (num*(den*bY + num*aY)),  s = sqrt(aX*bX*aY*bY).
pub fn optimal_input_0_for_1(a_x: U256, a_y: U256, b_y: U256, b_x: U256, fee_bps: U256) -> Option<U256> {
    let den = bps();
    if a_x.is_zero() || a_y.is_zero() || b_y.is_zero() || b_x.is_zero() || fee_bps >= den { return Some(U256::ZERO); }
    let num = den.wrapping_sub(fee_bps);
    let s = sqrt_product4(a_x, b_x, a_y, b_y)?;
    let scaled = num.checked_mul(s)?;
    let base = den.checked_mul(a_x)?.checked_mul(b_y)?;
    if scaled <= base { return Some(U256::ZERO); }
    let top = den.checked_mul(scaled.wrapping_sub(base))?;
    let bottom = num.checked_mul(den.checked_mul(b_y)?.checked_add(num.checked_mul(a_y)?)?)?;
    top.checked_div(bottom)
}

pub fn estimate_profit(x: U256, a_x: U256, a_y: U256, b_y: U256, b_x: U256, fee_bps: U256) -> Option<U256> {
    let out1 = get_amount_out(x, a_x, a_y, fee_bps)?;
    let out2 = get_amount_out(out1, b_y, b_x, fee_bps)?;
    Some(if out2 > x { out2.wrapping_sub(x) } else { U256::ZERO })
}

/// MP-05: never trust the closed form alone; check neighbors and keep the best.
pub fn optimal_input_refined(a_x: U256, a_y: U256, b_y: U256, b_x: U256, fee_bps: U256) -> Option<(U256, U256)> {
    let x0 = optimal_input_0_for_1(a_x, a_y, b_y, b_x, fee_bps)?;
    if x0.is_zero() { return Some((U256::ZERO, U256::ZERO)); }
    let mut best = (x0, estimate_profit(x0, a_x, a_y, b_y, b_x, fee_bps)?);
    for d in [1u64, 2, 10] {
        let d = U256::from(d);
        for cand in [x0.checked_add(d), x0.checked_sub(d)].into_iter().flatten() {
            if cand.is_zero() { continue; }
            if let Some(p) = estimate_profit(cand, a_x, a_y, b_y, b_x, fee_bps) {
                if p > best.1 { best = (cand, p); }
            }
        }
    }
    Some(best)
}
```

### 7.12 src/feed.rs

```text
// src/feed.rs  -- Arbitrum feed decoding; must never panic (OC-04). Import paths: verify against alloy 0.8.
use alloy::consensus::TxEnvelope;
use alloy::eips::eip2718::Decodable2718;
use alloy::rlp::Decodable;

pub const KIND_BATCH: u8 = 0x03;       // L2MessageKind: batch of length-prefixed sub-messages (8-byte BE length)
pub const KIND_SIGNED_TX: u8 = 0x04;   // L2MessageKind: signed transaction (typed or legacy RLP follows)
pub const KIND_HEARTBEAT: u8 = 0x06;
pub const MAX_DEPTH: u8 = 4;

#[derive(Debug)]
pub enum FeedError { Truncated, TooDeep, UnsupportedKind(u8) }

/// Collects the raw signed-tx bytes (type byte 0x04 already stripped) from a single or batched message.
pub fn extract_signed_txs<'a>(msg: &'a [u8], depth: u8, out: &mut Vec<&'a [u8]>) -> Result<(), FeedError> {
    if depth > MAX_DEPTH { return Err(FeedError::TooDeep); }
    let (kind, rest) = msg.split_first().ok_or(FeedError::Truncated)?;
    match *kind {
        KIND_SIGNED_TX => { out.push(rest); Ok(()) }          // the most common case: a single tx
        KIND_BATCH => {
            let mut pos = rest;
            while !pos.is_empty() {
                if pos.len() < 8 { return Err(FeedError::Truncated); }
                let (len_bytes, tail) = pos.split_at(8);
                let raw: [u8; 8] = len_bytes.try_into().map_err(|_| FeedError::Truncated)?;
                let len = usize::try_from(u64::from_be_bytes(raw)).map_err(|_| FeedError::Truncated)?;
                if tail.len() < len { return Err(FeedError::Truncated); }   // no wrap: compared, not added
                let (item, next) = tail.split_at(len);
                extract_signed_txs(item, depth.saturating_add(1), out)?;    // nested batches supported
                pos = next;
            }
            Ok(())
        }
        KIND_HEARTBEAT => Ok(()),
        other => Err(FeedError::UnsupportedKind(other)),     // caller counts and alerts; never aborts the stream
    }
}

pub fn decode_tx_bytes(raw: &[u8]) -> Option<TxEnvelope> {
    let first = raw.first().copied()?;
    let mut buf = raw;
    if first < 0x80 { TxEnvelope::decode_2718(&mut buf).ok() }     // EIP-2718 typed
    else { TxEnvelope::decode(&mut buf).ok() }                      // legacy RLP
}
```

### 7.13 src/main.rs

```text
// src/main.rs  -- reference pipeline. No step may be skipped; nothing here fabricates a SimulationReport.
use std::sync::Arc;
use std::time::{SystemTime, UNIX_EPOCH};
use alloy::primitives::{Address, Bytes, U256};
use crate::age::{classify_opportunity_age, OpportunityAge};
use crate::bidder::DynamicBidder;
use crate::flash::select_optimal_flash_source;
use crate::hub::{ExecutionSignal, Governor, OnChainEvent};
use crate::physics::decay_adjusted_profit_wei;
use crate::signer::SignerPool;
use crate::simroom::{RuntimeInvariantGuard, Simulator, UnsignedTx};
use crate::tx::encode_executor_calldata;

pub struct EngineConfig {
    pub shadow_mode: bool,
    pub executor: Address,
    pub profit_recipient: Address,       // cold wallet; MUST equal executor.profitRecipient() (self-check)
    pub max_gas_ceiling: u64,
    pub tier0_enabled: bool,
    pub deadline_secs: u64,              // <= MAX_DEADLINE_HORIZON_SECS
    pub min_profit_bps_of_expected: u128,// minNetProfit = max(floor, expected * bps / 10_000)
}

/// Everything the pipeline needs from the chain; implementations must use real data, not constants.
pub trait ChainView {
    fn block_number(&self) -> u64;
    fn balancer_liquidity(&self, token: Address) -> U256;
    fn l2_base_fee_wei(&self) -> u128;
    fn l1_data_fee_wei(&self, calldata: &Bytes) -> Option<u128>;          // NodeInterface-based, verified
    fn competitor_priority_fee_wei(&self) -> u128;                        // measured from the feed, else 0
    fn value_in_wei(&self, token: Address, amount: U256) -> Option<u128>; // price conversion
    fn min_profit_floor(&self, token: Address) -> U256;                   // read from the contract
    fn v3_pool_allowed(&self, pool: Address) -> bool;                     // read from the contract
}
pub trait Sender { fn send_raw(&self, raw: Bytes) -> Result<(), String>; fn pending_nonce(&self, a: Address) -> Option<u64>; }
pub trait ExecutorReader {
    fn chain_id(&self) -> u64;
    fn profit_recipient(&self) -> Address;
    fn is_operator(&self, who: Address) -> bool;
    fn max_loan(&self, token: Address) -> U256;
    fn code_hash(&self) -> [u8; 32];
}

/// OC-08: fail closed.
pub fn startup_self_check(cfg: &EngineConfig, r: &dyn ExecutorReader, signers: &SignerPool,
                          traded: &[Address], expected_code_hash: [u8; 32]) -> eyre::Result<()> {
    eyre::ensure!(r.chain_id() == crate::constants::ARBITRUM_CHAIN_ID, "wrong chain id");
    eyre::ensure!(r.profit_recipient() == cfg.profit_recipient, "profit recipient mismatch");
    eyre::ensure!(r.code_hash() == expected_code_hash, "executor code hash mismatch");
    for a in signers.addresses() { eyre::ensure!(r.is_operator(a), "signer {a} is not an operator"); }
    for t in traded { eyre::ensure!(!r.max_loan(*t).is_zero(), "token {t} has maxLoan == 0"); }
    eyre::ensure!(cfg.deadline_secs <= crate::constants::MAX_DEADLINE_HORIZON_SECS, "deadline horizon too large");
    Ok(())
}

pub enum Outcome { Dropped(&'static str), ShadowOk, Sent }

pub fn handle_signal(cfg: &EngineConfig, sig: &ExecutionSignal, now_ms: u64, chain: &dyn ChainView,
    bidder: &DynamicBidder, guard: &RuntimeInvariantGuard, sim: &dyn Simulator,
    signers: &SignerPool, sender: &dyn Sender) -> Outcome
{
    let delta_ms = now_ms.saturating_sub(sig.event_timestamp_ms);
    let is_liq = sig.liquidation.is_some();
    if classify_opportunity_age(delta_ms, is_liq) == OpportunityAge::Expired { return Outcome::Dropped("expired"); }

    let plan = match select_optimal_flash_source(sig.loan_token, sig.loan_amount, &sig.steps, is_liq,
        chain.balancer_liquidity(sig.loan_token), cfg.tier0_enabled, &|p| chain.v3_pool_allowed(p)) {
        Some(p) => p, None => return Outcome::Dropped("no plan") };

    let floor = chain.min_profit_floor(sig.loan_token);
    let frac = sig.expected_profit_loan_token
        .checked_mul(U256::from(cfg.min_profit_bps_of_expected)).and_then(|v| v.checked_div(U256::from(10_000u64)))
        .unwrap_or(U256::ZERO);
    let min_net = if frac > floor { frac } else { floor };

    let deadline = U256::from((now_ms / 1000).saturating_add(cfg.deadline_secs));
    let calldata = match encode_executor_calldata(sig, &plan, min_net, deadline) {
        Some(c) => c, None => return Outcome::Dropped("encode/route invalid") };
    let unsigned = UnsignedTx { to: cfg.executor, data: calldata.clone(), gas_limit: cfg.max_gas_ceiling };

    let report = match sim.run(chain.block_number(), &unsigned, cfg.profit_recipient, sig.loan_token) {
        Some(r) => r, None => return Outcome::Dropped("simulation failed") };
    let gas_used = report.gas_used;
    let realized_profit = report.recipient_balance_after.saturating_sub(report.recipient_balance_before);
    let verified = match guard.verify(&report, U256::ZERO /* pre-existing: executor holds none at rest */, min_net, unsigned) {
        Some(v) => v, None => return Outcome::Dropped("guard rejected") };

    let gross_wei = match chain.value_in_wei(sig.loan_token, realized_profit) {
        Some(v) => v, None => return Outcome::Dropped("no price") };
    let decayed = decay_adjusted_profit_wei(gross_wei, delta_ms);
    let l1 = match chain.l1_data_fee_wei(&calldata) { Some(v) => v, None => return Outcome::Dropped("no l1 fee") };
    let bid = bidder.evaluate_bid(decayed, u128::from(gas_used), chain.l2_base_fee_wei(), l1,
                                  chain.competitor_priority_fee_wei());
    if !bid.should_execute { return Outcome::Dropped("not profitable after costs"); }
    if cfg.shadow_mode { return Outcome::ShadowOk; }

    let lease = match signers.acquire() { Some(l) => l, None => return Outcome::Dropped("no wallet free") };
    let raw = match lease.wallet.signer.sign(&verified, lease.nonce, bid.max_fee_per_gas_wei, bid.priority_fee_per_gas_wei) {
        Some(r) => r, None => return Outcome::Dropped("sign failed") };
    match sender.send_raw(raw) {
        Ok(()) => { signers.commit(lease); Outcome::Sent }
        Err(_) => { let addr = lease.wallet.address; drop(lease);
                    if let Some(n) = sender.pending_nonce(addr) { signers.resync(addr, n); }
                    Outcome::Dropped("send failed") }
    }
}

// main(): load config -> build adapters -> startup_self_check -> connect feed (feed.rs) -> for each
// message: extract_signed_txs -> decode -> build OnChainEvent -> governor.broadcast(now_ms) ->
// handle_signal for each signal -> reconcile executed txs (OP-02) and update circuit breakers (OP-01).
// NOTE: how an OnChainEvent becomes an ExecutionSignal (the spokes' detection logic) is NOT specified
// by v10.5 and is out of scope here (decision D-6).
```

### 7.14 Notes on the lab models

ConstantProductAMM and ArbitrageRouter are simulation models, not production contracts. If kept: remove the dead check amountOut >= reserve, the unused error Overflow and the unused "using MEVMath for uint256"; give addLiquidity no pretense of safety (it has no token transfers); and make ArbitrageRouter use the same checked semantics as math.rs. Huff and Yul: see MP-06; the Yul object needs a runtime sub-object to be deployable, and both must reproduce getAmountOut exactly (0 for a zero reserve, revert for fee > 10,000 and on overflow) or be dropped.

### 7.15 contracts/src/libraries/MEVMath.sol

```text
// SPDX-License-Identifier: MIT
pragma solidity 0.8.34;       // pin exactly; must match the version used for Annex build (SC-11)

library MEVMath {
    uint256 internal constant BPS = 10_000;
    error FeeTooHigh(uint256 feeBps);
    error FactorTooLarge();
    error InsufficientLiquidity();

    function max(uint256 a, uint256 b) internal pure returns (uint256) { return a >= b ? a : b; }
    function min(uint256 a, uint256 b) internal pure returns (uint256) { return a <= b ? a : b; }

    function sqrt(uint256 x) internal pure returns (uint256 z) {
        if (x == 0) return 0;
        z = (x >> 1) + (x & 1);
        uint256 y = x;
        while (z < y) { y = z; z = (x / z + z) / 2; }
        return y;
    }

    function sqrtProduct(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0 || b == 0) return 0;
        unchecked { if (a <= type(uint256).max / b) return sqrt(a * b); }
        (uint256 hi, uint256 lo) = _mul512(a, b);
        return _sqrt512(hi, lo);
    }

    /// floor(sqrt(a*b*c*d)); exact for factors <= 2^128-1, reverts otherwise (never approximates).
    function sqrtProduct4(uint256 a, uint256 b, uint256 c, uint256 d) internal pure returns (uint256) {
        if (a == 0 || b == 0 || c == 0 || d == 0) return 0;
        if (a > type(uint128).max || b > type(uint128).max || c > type(uint128).max || d > type(uint128).max)
            revert FactorTooLarge();
        unchecked { (uint256 hi, uint256 lo) = _mul512(a * b, c * d); return _sqrt512(hi, lo); }
    }

    function _mul512(uint256 a, uint256 b) private pure returns (uint256 hi, uint256 lo) {
        assembly ("memory-safe") {
            let mm := mulmod(a, b, not(0))
            lo := mul(a, b)
            hi := sub(sub(mm, lo), lt(mm, lo))
        }
    }

    function _sqrt512(uint256 hi, uint256 lo) private pure returns (uint256 r) {
        uint256 bit = 1 << 255;
        while (bit > 0) {
            uint256 candidate = r | bit;
            (uint256 ch, uint256 cl) = _mul512(candidate, candidate);
            if (ch < hi || (ch == hi && cl <= lo)) r = candidate;
            bit >>= 1;
        }
    }

    function getAmountOut(uint256 amountIn, uint256 reserveIn, uint256 reserveOut, uint256 feeBps)
        internal pure returns (uint256)
    {
        if (feeBps > BPS) revert FeeTooHigh(feeBps);
        if (amountIn == 0 || reserveIn == 0 || reserveOut == 0) return 0;
        uint256 aif = amountIn * (BPS - feeBps);
        return (aif * reserveOut) / (reserveIn * BPS + aif);
    }

    function getAmountIn(uint256 amountOut, uint256 reserveIn, uint256 reserveOut, uint256 feeBps)
        internal pure returns (uint256)
    {
        if (feeBps >= BPS) revert FeeTooHigh(feeBps);
        if (amountOut == 0 || reserveIn == 0 || reserveOut == 0) return 0;
        if (amountOut >= reserveOut) revert InsufficientLiquidity();
        return (reserveIn * amountOut * BPS) / ((reserveOut - amountOut) * (BPS - feeBps)) + 1;
    }

    function priceImpactBps(uint256 amountIn, uint256 amountOut, uint256 reserveIn, uint256 reserveOut)
        internal pure returns (uint256)
    {
        if (amountIn == 0 || reserveIn == 0 || reserveOut == 0) return 0;
        uint256 num = amountOut * reserveIn;
        uint256 den = amountIn * reserveOut;
        if (num >= den) return 0;
        return ((den - num) * BPS) / den;
    }
}
```

### 7.16 contracts/src/MEVExecutor.sol (merged final)

> **Note:** Role model, limits (maxLoan, pool and hop-token whitelists, MAX_HOPS, deadline horizon, per-token minProfitFloor), route validation, callback-executed flag, events, pause and the corrected _safeApprove. Admin setters for hop tokens and floors are included. Note: allowedHopToken must contain every hop tokenOut including the repay token.

```text
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;   // build with evm_version = "cancun" (EIP-1153)

interface IERC20 { function balanceOf(address a) external view returns (uint256); }
interface IBalancerVault {
    function flashLoan(address recipient, address[] memory tokens,
                       uint256[] memory amounts, bytes memory userData) external;
}
interface IAaveV3Pool {
    function flashLoanSimple(address receiver, address asset, uint256 amount,
                             bytes calldata params, uint16 referralCode) external;
    function liquidationCall(address collateral, address debt, address user,
                             uint256 debtToCover, bool receiveAToken) external;
}
interface IUniV3Pool {
    function token0() external view returns (address);
    function token1() external view returns (address);
    function swap(address recipient, bool zeroForOne, int256 amountSpecified,
                  uint160 sqrtPriceLimitX96, bytes calldata data)
        external returns (int256 amount0, int256 amount1);
}
interface IUniV3Router {
    struct ExactInputSingleParams {
        address tokenIn; address tokenOut; uint24 fee; address recipient;
        uint256 deadline; uint256 amountIn; uint256 amountOutMinimum; uint160 sqrtPriceLimitX96;
    }
    function exactInputSingle(ExactInputSingleParams calldata p)
        external payable returns (uint256 amountOut);
}
interface ICamelotAlgebraRouter {
    struct ExactInputSingleParams {
        address tokenIn; address tokenOut; address recipient; uint256 deadline;
        uint256 amountIn; uint256 amountOutMinimum; uint160 limitSqrtPrice;
    }
    function exactInputSingle(ExactInputSingleParams calldata p)
        external payable returns (uint256 amountOut);
}
interface IUniV2Router {
    function swapExactTokensForTokens(uint256 amountIn, uint256 amountOutMin,
        address[] calldata path, address to, uint256 deadline)
        external returns (uint256[] memory amounts);
}

contract MEVExecutor {
    // ---- roles (v10.6: executor key is NOT the profit sink and NOT the admin) ----
    address public immutable admin;            // cold key / multisig / timelock
    address public immutable guardian;         // may pause only
    address public immutable profitRecipient;  // cold address, receives all profit
    mapping(address => bool)    public isOperator;    // hot execution keys (SignerPool)
    mapping(address => bool)    public allowedV3Pool; // whitelist for Tier-0
    mapping(address => uint256) public maxLoan;       // per repay token; 0 = disabled
    bool public paused;
    mapping(address => uint256) public minProfitFloor;     // per repay token
    mapping(address => bool)    public allowedHopToken;    // every hop tokenOut (incl. the repay token)
    uint256 public constant MAX_HOPS = 4;
    uint256 public constant MAX_DEADLINE_HORIZON = 30;     // seconds

    event Executed(address indexed repayToken, uint256 profit);
    event OperatorSet(address indexed who, bool ok);
    event PoolSet(address indexed pool, bool ok);
    event HopTokenSet(address indexed token, bool ok);
    event MaxLoanSet(address indexed token, uint256 cap);
    event MinProfitFloorSet(address indexed token, uint256 floor);
    event PausedSet(bool paused, address indexed by);

    address public constant BALANCER_VAULT = 0xBA12222222228d8Ba445958a75a0704d566BF2C8;
    address public constant AAVE_V3_POOL   = 0x794a61358D6845594F94dc1DB02A252b5b4814aD;
    address public constant UNIV3_ROUTER   = 0xE592427A0AEce92De3Edee1F18E0157C05861564;
    address public constant CAMELOT_ROUTER = 0x1F721E2E82F6676FCE4eA07A5958cF098D339e18;
    address public constant UNIV2_ROUTER   = 0x4752ba5DBc23f44D87826276BF6Fd6b1C372aD24;

    // EIP-1153 transient slots
    uint256 private constant REENTRANCY_SLOT           = 0x929eee149b4bd21268;
    uint256 private constant EXPECTED_CALLBACK_SLOT    = 0x82f49447d8a07e3bd9;
    uint256 private constant EXPECTED_REPAY_TOKEN_SLOT = 0x559fe8253a0e49e654;
    uint256 private constant PRE_BALANCE_SLOT          = 0x4166549f7aab2e75be;
    uint256 private constant CALLBACK_DONE_SLOT        = 0x1c7e1a6b2d93f05a44;

    struct RouteHop { uint8 protocolId; address tokenIn; address tokenOut;
                      uint24 feeTier; uint256 minAmountOut; }
    struct ExecutionPayload {
        bool hasLiquidation; address borrower; address collateralAsset; address debtAsset;
        uint256 debtToCover; uint256 minNetProfit; uint256 deadline; RouteHop[] hops;
    }
    struct Ctx {                 // avoids stack-too-deep and carries BOTH tokens
        address repayToken;      // token we owe back to the lender
        address startToken;      // token we actually received first
        uint256 startingAmount;  // amount of startToken received
        uint256 requiredRepay;   // amount of repayToken owed
        uint256 preBalance;      // contract balance of repayToken before the loan
        bool    allowLiquidation;
    }

    error Unauthorized(); error Paused(); error ReentrancyDetected();
    error InvalidCallbackDelta(); error RepayTokenMismatch();
    error UnsupportedProtocol(uint8 protocolId);
    error SafeTransferFailed(); error SafeApproveFailed();
    error DeadlineExpired(uint256 deadline, uint256 currentTimestamp);
    error InsufficientProfit(uint256 finalBal, uint256 requiredBal);
    error PoolNotAllowed(); error LoanCapExceeded(); error BadRoute();
    error TooManyHops(); error DeadlineTooFar(); error MinProfitBelowFloor(); error TokenNotAllowed();
    error CallbackNotExecuted(); error LiquidationNotAllowed();
    error ZeroAddress(); error ZeroAmount();

    modifier onlyAdmin()    { if (msg.sender != admin) revert Unauthorized(); _; }
    modifier onlyOperator() { if (!isOperator[msg.sender]) revert Unauthorized(); _; }
    modifier whenNotPaused(){ if (paused) revert Paused(); _; }
    modifier nonReentrantCallback() {
        uint256 locked;
        assembly { locked := tload(REENTRANCY_SLOT) if iszero(locked) { tstore(REENTRANCY_SLOT, 1) } }
        if (locked != 0) revert ReentrancyDetected();
        _;
        assembly { tstore(REENTRANCY_SLOT, 0) }
    }

    constructor(address admin_, address guardian_, address profitRecipient_) {
        if (admin_ == address(0) || guardian_ == address(0) || profitRecipient_ == address(0))
            revert ZeroAddress();
        admin = admin_; guardian = guardian_; profitRecipient = profitRecipient_;
    }

    // ---------------- admin surface (no fund-moving power except sweep to recipient) ----
    function setOperator(address who, bool ok) external onlyAdmin { isOperator[who] = ok; emit OperatorSet(who, ok); }
    function setV3Pool(address pool, bool ok)  external onlyAdmin { allowedV3Pool[pool] = ok; emit PoolSet(pool, ok); }
    function setHopToken(address t, bool ok) external onlyAdmin { allowedHopToken[t] = ok; emit HopTokenSet(t, ok); }
    function setMinProfitFloor(address t, uint256 f) external onlyAdmin { minProfitFloor[t] = f; emit MinProfitFloorSet(t, f); }
    function setMaxLoan(address token, uint256 cap) external onlyAdmin { maxLoan[token] = cap; emit MaxLoanSet(token, cap); }
    function pause() external {
        if (msg.sender != admin && msg.sender != guardian) revert Unauthorized();
        paused = true; emit PausedSet(true, msg.sender);
    }
    function unpause() external onlyAdmin { paused = false; emit PausedSet(false, msg.sender); }
    function emergencySweep(address token) external onlyAdmin {
        uint256 bal = IERC20(token).balanceOf(address(this));
        if (bal > 0) _safeTransfer(token, profitRecipient, bal);
    }

    // ---------------- low-level token helpers ----------------
    function _safeTransfer(address token, address to, uint256 value) internal {
        bool success;
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, 0xa9059cbb00000000000000000000000000000000000000000000000000000000)
            mstore(add(ptr, 0x04), and(to, 0xffffffffffffffffffffffffffffffffffffffff))
            mstore(add(ptr, 0x24), value)
            success := call(gas(), token, 0, ptr, 0x44, ptr, 0x20)
            if success {
                switch returndatasize()
                case 0 { success := gt(extcodesize(token), 0) }
                default { success := and(gt(returndatasize(), 0x1f), eq(mload(ptr), 1)) }
            }
        }
        if (!success) revert SafeTransferFailed();
    }

    // v11 fix C6: return data goes to rptr, so the calldata at ptr is never overwritten
    // (v10.5 wrote the return value over the selector, corrupting the approve(0) retry).
    function _safeApprove(address token, address spender, uint256 value) internal {
        bool success;
        assembly {
            let ptr := mload(0x40)
            let rptr := add(ptr, 0x80)
            mstore(ptr, 0x095ea7b300000000000000000000000000000000000000000000000000000000)
            mstore(add(ptr, 0x04), and(spender, 0xffffffffffffffffffffffffffffffffffffffff))
            mstore(add(ptr, 0x24), value)
            success := call(gas(), token, 0, ptr, 0x44, rptr, 0x20)
            if success {
                switch returndatasize()
                case 0 { success := gt(extcodesize(token), 0) }
                default { success := and(gt(returndatasize(), 0x1f), eq(mload(rptr), 1)) }
            }
            if and(iszero(success), gt(value, 0)) {
                mstore(add(ptr, 0x24), 0)
                let resetOk := call(gas(), token, 0, ptr, 0x44, rptr, 0x20)
                if resetOk {
                    mstore(add(ptr, 0x24), value)
                    success := call(gas(), token, 0, ptr, 0x44, rptr, 0x20)
                    if success {
                        switch returndatasize()
                        case 0 { success := gt(extcodesize(token), 0) }
                        default { success := and(gt(returndatasize(), 0x1f), eq(mload(rptr), 1)) }
                    }
                }
            }
        }
        if (!success) revert SafeApproveFailed();
    }

    function _arm(address expectedCaller, address repayToken, uint256 preBal) internal {
        assembly {
            tstore(EXPECTED_CALLBACK_SLOT, expectedCaller)
            tstore(EXPECTED_REPAY_TOKEN_SLOT, repayToken)
            tstore(PRE_BALANCE_SLOT, preBal)
            tstore(CALLBACK_DONE_SLOT, 0)
        }
    }
    function _disarm() internal {
        assembly {
            tstore(EXPECTED_CALLBACK_SLOT, 0) tstore(EXPECTED_REPAY_TOKEN_SLOT, 0)
            tstore(PRE_BALANCE_SLOT, 0)       tstore(CALLBACK_DONE_SLOT, 0)
        }
    }
    function _checkCap(address token, uint256 amount) internal view {
        if (amount == 0) revert ZeroAmount();
        if (amount > maxLoan[token]) revert LoanCapExceeded();   // maxLoan==0 disables token
    }

    // ================= Tier-0: Uniswap V3 flash swap (no liquidation on this path) =======
    function executeViaUniV3Callback(
        address v3Pool, address repayToken, bool zeroForOne,
        int256 amountSpecified, uint160 sqrtPriceLimitX96, bytes calldata payloadBytes
    ) external onlyOperator whenNotPaused {
        if (!allowedV3Pool[v3Pool]) revert PoolNotAllowed();
        if (amountSpecified <= 0) revert ZeroAmount();            // exact-input only
        _checkCap(repayToken, uint256(amountSpecified));
        _arm(v3Pool, repayToken, IERC20(repayToken).balanceOf(address(this)));
        IUniV3Pool(v3Pool).swap(address(this), zeroForOne, amountSpecified,
                                sqrtPriceLimitX96, payloadBytes);
        uint256 done;
        assembly { done := tload(CALLBACK_DONE_SLOT) }
        if (done != 1) revert CallbackNotExecuted();
        _disarm();
    }

    function uniswapV3SwapCallback(int256 amount0Delta, int256 amount1Delta, bytes calldata data)
        external nonReentrantCallback
    {
        address expectedPool; address expectedRepayToken; uint256 preBal;
        assembly {
            expectedPool := tload(EXPECTED_CALLBACK_SLOT)
            expectedRepayToken := tload(EXPECTED_REPAY_TOKEN_SLOT)
            preBal := tload(PRE_BALANCE_SLOT)
        }
        if (msg.sender != expectedPool || expectedPool == address(0)) revert Unauthorized();
        if (!((amount0Delta > 0 && amount1Delta < 0) || (amount1Delta > 0 && amount0Delta < 0)))
            revert InvalidCallbackDelta();

        address t0 = IUniV3Pool(msg.sender).token0();
        address t1 = IUniV3Pool(msg.sender).token1();
        bool owe0 = amount0Delta > 0;
        address repayToken = owe0 ? t0 : t1;
        if (repayToken != expectedRepayToken) revert RepayTokenMismatch();
        assembly { tstore(CALLBACK_DONE_SLOT, 1) }

        Ctx memory c = Ctx({
            repayToken: repayToken,
            startToken: owe0 ? t1 : t0,                                   // v11 fix C2
            startingAmount: owe0 ? uint256(-amount1Delta) : uint256(-amount0Delta),
            requiredRepay: owe0 ? uint256(amount0Delta) : uint256(amount1Delta),
            preBalance: preBal,
            allowLiquidation: false                                       // v11 fix C2
        });
        _runUnifiedStrategy(c, data);
        _safeTransfer(repayToken, msg.sender, c.requiredRepay);
    }

    // ================= Tier-1: Balancer V2 =================
    function executeViaBalancer(address loanToken, uint256 loanAmount, bytes calldata payloadBytes)
        external onlyOperator whenNotPaused
    {
        _checkCap(loanToken, loanAmount);
        _arm(BALANCER_VAULT, loanToken, IERC20(loanToken).balanceOf(address(this)));
        address[] memory tokens = new address[](1); tokens[0] = loanToken;
        uint256[] memory amounts = new uint256[](1); amounts[0] = loanAmount;
        IBalancerVault(BALANCER_VAULT).flashLoan(address(this), tokens, amounts, payloadBytes);
        _disarm();
    }

    function receiveFlashLoan(address[] memory tokens, uint256[] memory amounts,
        uint256[] memory feeAmounts, bytes memory userData) external nonReentrantCallback
    {
        address expectedCaller; address expectedRepayToken; uint256 preBal;
        assembly {
            expectedCaller := tload(EXPECTED_CALLBACK_SLOT)
            expectedRepayToken := tload(EXPECTED_REPAY_TOKEN_SLOT)
            preBal := tload(PRE_BALANCE_SLOT)
        }
        if (msg.sender != BALANCER_VAULT || expectedCaller != BALANCER_VAULT) revert Unauthorized();
        if (tokens.length != 1 || amounts.length != 1 || feeAmounts.length != 1
            || tokens[0] != expectedRepayToken) revert RepayTokenMismatch();
        Ctx memory c = Ctx({ repayToken: tokens[0], startToken: tokens[0],
            startingAmount: amounts[0], requiredRepay: amounts[0] + feeAmounts[0],
            preBalance: preBal, allowLiquidation: true });
        _runUnifiedStrategy(c, userData);
        _safeTransfer(tokens[0], BALANCER_VAULT, c.requiredRepay);
    }

    // ================= Tier-2: Aave V3 =================
    function executeViaAave(address loanToken, uint256 loanAmount, bytes calldata payloadBytes)
        external onlyOperator whenNotPaused
    {
        _checkCap(loanToken, loanAmount);
        _arm(AAVE_V3_POOL, loanToken, IERC20(loanToken).balanceOf(address(this)));
        IAaveV3Pool(AAVE_V3_POOL).flashLoanSimple(address(this), loanToken, loanAmount, payloadBytes, 0);
        _disarm();
    }

    function executeOperation(address asset, uint256 amount, uint256 premium,
        address initiator, bytes calldata params) external nonReentrantCallback returns (bool)
    {
        address expectedCaller; address expectedRepayToken; uint256 preBal;
        assembly {
            expectedCaller := tload(EXPECTED_CALLBACK_SLOT)
            expectedRepayToken := tload(EXPECTED_REPAY_TOKEN_SLOT)
            preBal := tload(PRE_BALANCE_SLOT)
        }
        if (msg.sender != AAVE_V3_POOL || expectedCaller != AAVE_V3_POOL || initiator != address(this))
            revert Unauthorized();
        if (asset != expectedRepayToken) revert RepayTokenMismatch();
        Ctx memory c = Ctx({ repayToken: asset, startToken: asset, startingAmount: amount,
            requiredRepay: amount + premium, preBalance: preBal, allowLiquidation: true });
        _runUnifiedStrategy(c, params);
        _safeApprove(asset, AAVE_V3_POOL, c.requiredRepay);
        return true;
    }

    // ================= Atomic core =================
    function _runUnifiedStrategy(Ctx memory c, bytes memory raw) internal {
        ExecutionPayload memory p = abi.decode(raw, (ExecutionPayload));
        if (block.timestamp > p.deadline) revert DeadlineExpired(p.deadline, block.timestamp);
        if (p.hops.length > MAX_HOPS) revert TooManyHops();
        if (p.deadline > block.timestamp + MAX_DEADLINE_HORIZON) revert DeadlineTooFar();
        if (p.minNetProfit == 0 || p.minNetProfit < minProfitFloor[c.repayToken]) revert MinProfitBelowFloor();

        address curToken = c.startToken;
        uint256 curAmount = c.startingAmount;

        if (p.hasLiquidation) {
            if (!c.allowLiquidation) revert LiquidationNotAllowed();
            if (p.debtAsset != c.startToken || p.debtAsset != c.repayToken
                || p.debtToCover == 0 || p.debtToCover > c.startingAmount) revert RepayTokenMismatch();
            uint256 colPre = IERC20(p.collateralAsset).balanceOf(address(this));
            _safeApprove(p.debtAsset, AAVE_V3_POOL, p.debtToCover);
            IAaveV3Pool(AAVE_V3_POOL).liquidationCall(
                p.collateralAsset, p.debtAsset, p.borrower, p.debtToCover, false);
            _safeApprove(p.debtAsset, AAVE_V3_POOL, 0);
            curAmount = IERC20(p.collateralAsset).balanceOf(address(this)) - colPre;
            curToken = p.collateralAsset;
            if (curAmount == 0) revert ZeroAmount();
        }

        uint256 n = p.hops.length;
        for (uint256 i = 0; i < n; ++i) {
            RouteHop memory h = p.hops[i];
            if (h.tokenIn != curToken) revert BadRoute();             // v11 fix C3: continuity
            if (!allowedHopToken[h.tokenOut]) revert TokenNotAllowed();
            curAmount = _executeHop(h, curAmount, p.deadline);
            curToken = h.tokenOut;
        }
        if (curToken != c.repayToken) revert BadRoute();              // v11 fix C3: ends in repay token

        // Delta-isolated guard (unchanged semantics)
        uint256 finalBal = IERC20(c.repayToken).balanceOf(address(this));
        uint256 needed = c.preBalance + c.requiredRepay + p.minNetProfit;
        if (finalBal < needed) revert InsufficientProfit(finalBal, needed);

        // v11 fix C1: sweep ONLY the net profit; pre-existing balance stays in the contract
        uint256 profit = finalBal - c.preBalance - c.requiredRepay;
        if (profit > 0) _safeTransfer(c.repayToken, profitRecipient, profit);   // v11 fix C4
        emit Executed(c.repayToken, profit);
    }

    function _executeHop(RouteHop memory h, uint256 amountIn, uint256 deadline)
        internal returns (uint256 out)
    {
        if (h.protocolId == 1) {
            _safeApprove(h.tokenIn, UNIV3_ROUTER, amountIn);
            out = IUniV3Router(UNIV3_ROUTER).exactInputSingle(IUniV3Router.ExactInputSingleParams({
                tokenIn: h.tokenIn, tokenOut: h.tokenOut, fee: h.feeTier, recipient: address(this),
                deadline: deadline, amountIn: amountIn, amountOutMinimum: h.minAmountOut,
                sqrtPriceLimitX96: 0 }));
        } else if (h.protocolId == 3) {
            _safeApprove(h.tokenIn, CAMELOT_ROUTER, amountIn);
            out = ICamelotAlgebraRouter(CAMELOT_ROUTER).exactInputSingle(
                ICamelotAlgebraRouter.ExactInputSingleParams({
                tokenIn: h.tokenIn, tokenOut: h.tokenOut, recipient: address(this),
                deadline: deadline, amountIn: amountIn, amountOutMinimum: h.minAmountOut,
                limitSqrtPrice: 0 }));
        } else if (h.protocolId == 0) {
            _safeApprove(h.tokenIn, UNIV2_ROUTER, amountIn);
            address[] memory path = new address[](2);
            path[0] = h.tokenIn; path[1] = h.tokenOut;
            uint256[] memory outs = IUniV2Router(UNIV2_ROUTER).swapExactTokensForTokens(
                amountIn, h.minAmountOut, path, address(this), deadline);
            out = outs[outs.length - 1];
        } else {
            revert UnsupportedProtocol(h.protocolId);
        }
    }
}
```

## 8. Verification requirements

### 8.1 Quality gates

| Check | Requirement (proposed defaults) |
|---|---|
| Coverage | 100% line and branch on MEVExecutor and MEVMath; Rust coverage on math, guard, signer, decoder, governor. |
| Mutation testing | Gambit (or equivalent) on the contract and MEVMath; cargo-mutants on Rust math/guard/signer. Score >= 90%; every survivor reviewed and either killed by a new test or justified in writing. |
| Fuzz | Foundry default >= 10,000 runs in CI; >= 1,000,000 for MEVMath nightly. Invariants: >= 512 runs, depth >= 200; fail_on_revert = true, or per-handler success ratio >= 70% logged and enforced. |
| Static analysis | Slither with no unresolved high/medium findings; Rust: clippy with the denied lints of OC-03, cargo-audit, cargo-deny. |
| Red-first evidence | For each corrected issue the test MUST be shown failing on the old code for the right reason (-vvv output archived), then passing on the fix. |

### 8.2 Fork tests (Arbitrum One)

End-to-end per tier and per protocol (V2-style router, V3 router, Camelot, Balancer, Aave flash, Aave liquidation) at several pinned blocks, with real USDT, USDC, WETH, ARB, WBTC. Assert: SC-03 balances, router allowances zero, events, gas within the ceiling, and agreement with the Rust simulation.

### 8.3 Adversarial mocks (each MUST have a test)

| # | Mock | Expected result |
|---|---|---|
| AM-1 | V3 pool that returns from swap() without calling the callback | CallbackNotExecuted |
| AM-2 | V3 pool that calls the callback twice | Revert (reentrancy or not-executed) |
| AM-3 | Pool with wrong token0/token1 or non-opposite deltas | RepayTokenMismatch / InvalidCallbackDelta |
| AM-4 | Attacker calls Balancer vault flashLoan(executor, ...) with crafted userData | Unauthorized |
| AM-5 | Attacker calls receiveFlashLoan / executeOperation / uniswapV3SwapCallback directly | Unauthorized |
| AM-6 | USDT-style token (no return value, requires approve(0) first) | Succeeds, allowance ends at 0 |
| AM-7 | Token returning false; fee-on-transfer; rebasing | Fails closed or profit measured by balance |
| AM-8 | ERC777-style hook reentering executeVia* / callbacks | Reverts |
| AM-9 | Router returning inflated amountOut, or consuming less than approved | Profit from balances; allowance reset |
| AM-10 | Token that reverts on transfer to recipient (blacklist) and returndata-bomb token | Whole tx reverts; no stuck state |
| AM-11 | Malformed payloads: empty hops, gap in route, wrong end token, hops > MAX_HOPS, far deadline, minNetProfit below floor, unknown protocolId | Specific custom errors |

### 8.4 Test matrix (red on v10.5, green on v11)

Each test MUST first FAIL on the v10.5 behavior for the stated reason (archive the -vvv output), then pass on the fix.

| Test | Covers | On v10.5 |
|---|---|---|
| test_Sweep_KeepsPreExistingBalance (preBal > 0) | C1, I-4 | FAIL: balance drained to owner |
| test_V3Path_Liquidation_Rejected / UsesCorrectTokens | C2 | FAIL: wrong-token compare or wrong approval |
| test_BrokenHopChain_Reverts (gap, wrong start, wrong end) | C3 | FAIL: opaque revert / wasted gas, no BadRoute |
| test_OperatorCannotChangeRecipientOrLists; test_OnlyAdminSweepsToRecipient | C4, I-8 | Not expressible on v10.5 (no roles) |
| test_V3Pool_NotWhitelisted_Reverts; test_LoanCap; test_Paused_Blocks_All | C5, I-7 | FAIL: no such checks |
| test_Approve_USDT_RetryPath (token that requires approve(0) first) | C6 | FAIL: retry sends corrupted selector |
| test_V3_SwapWithoutCallback_Reverts (malicious pool mock) | C7, I-3 | FAIL: silently succeeds |
| test_Callback_RevertsIfNotArmed (all 3) ; test_TransientSlotsZeroAfter | I-1 | PASS expected (regression) |
| test_InvalidCallbackDelta_Reverts; test_RepayTokenMismatch_Reverts | I-2 | PASS expected (regression) |
| testFuzz_SqrtProduct4_ExactOrReverts (factors around 2^128) | I-10 | FAIL: silent underestimate or panic 0x11 |
| testFuzz_GetAmountIn_BoundsRevertExplicitly (amountOut >= reserveOut, fee == BPS) | I-10 | FAIL: panic 0x11 / 0x12 instead of custom error |
| prop_rust_matches_solidity_vectors; prop_no_silent_overflow | I-10 | FAIL: wrap/None mismatch |
| test_Guard_RejectsHandBuiltReport; test_Guard_RecipientDelta | C8-C10, I-11 | FAIL: guard always true |
| test_Nonce_NoGapOnRejectedSim; test_Resync_MovesBackward | C11, I-12 | FAIL: nonce gap, no backward resync |
| testFork_Balancer / Aave / V3 / Camelot / Liquidation end-to-end on Arbitrum fork | all | n/a (needs fork) |
| invariant_ExecutorBalanceEqualsPre; invariant_OnlyRecipientGains | I-4, I-5 | FAIL on v10.5 |

## 9. Corrections to v10.5 and unverified claims

### 9.1 Why this document differs from v10.5

v10.5 contained logic errors, a decorative runtime guard, unit mistakes, unsourced market claims and the label "Gold Master / Hardened" without evidence of audit. The table lists every correction; the code in section 7 already incorporates them.

### 9.2 Correction table

| ID | v10.5 location | Problem | v10.6 fix |
|---|---|---|---|
| C1 | Changelog #3, 5.12 ProfitGuard sweep | Guard protects preExistingBalance, then sweepableProfit = finalBalance - requiredRepay sends that same balance to owner. Contradicts "delta-isolated". | Sweep only finalBal - preBalance - requiredRepay to profitRecipient. Pre-existing balance stays. |
| C2 | 5.12 V3 callback | _runUnifiedStrategy receives repayToken (token owed) but the received amount is in the OTHER token. Liquidation check compares debtAsset to the wrong token and debtToCover to the wrong amount. | Ctx carries startToken and repayToken. Liquidation is rejected on the V3 path (matches flash.rs, which already excludes it). Liquidation requires debtAsset == startToken == repayToken. |
| C3 | 5.12 hops | No route validation. Bad payloads waste gas and fail with opaque errors. | hops[0].tokenIn == startToken (or collateral after liquidation); each tokenIn == previous tokenOut; last tokenOut == repayToken, else BadRoute(). |
| C4 | 5.12 roles, 5.9, 6.1 | owner = msg.sender (deployer) and every entry point is onlyOwner, yet the spec runs 2-3 hot SignerPool wallets. Any non-deployer wallet reverts. Owner is also the profit sink: one hot key = everything. | admin (cold), guardian (pause only), immutable profitRecipient (cold), operator set for hot keys. Executor keys cannot change recipient or lists. |
| C5 | 5.12 limits | No pool whitelist, no loan cap, no pause. Tier-0 trusts any v3Pool the caller names. | allowedV3Pool whitelist, maxLoan per token (0 = disabled), pause/unpause, MinProfitZero check. |
| C6 | 5.12 _safeApprove | Return data is written to ptr, overwriting the approve selector. The approve(0)-then-approve retry (the USDT case it exists for) then sends a corrupted call. | Return data goes to rptr = ptr + 0x80. |
| C7 | 5.12 V3 entry | If swap() returns without invoking the callback nothing detects it. | CALLBACK_DONE transient flag; revert CallbackNotExecuted() if not set. |
| C8 | 5.10 + 5.13 | main.rs builds SimulationReport itself (success: true, balance computed from the expected profit, recipient = treasury). verify_simulation can never fail. The guard is decorative. | Report must come from a real revm run on forked state (section 4.2). main.rs may not construct it. |
| C9 | 5.13 governor_treasury | Set to constants::WETH, a token contract address, not a wallet. | Config value = the profitRecipient cold address, loaded from config and checked equal to the contract getter at startup. |
| C10 | 5.10 balance check | final_loan_token_balance >= pre + repay + profit is measured on the contract; after the sweep and repayment the contract holds roughly pre, so a correct trade fails. | Measure recipient balance delta and require executor balance == pre-existing balance after execution. |
| C11 | 5.9 SignerPool | fetch_add reserves a nonce at selection time, before simulation or send. A rejected or failed tx leaves a nonce gap and later txs hang. resync uses fetch_max so it can never move back. | Peek nonce, commit only after successful broadcast, resync with store() from chain pending nonce. One in-flight tx per wallet. |
| C12 | 1.2 revert cost | "~45,000 gas, $0.005-$0.02". ProfitGuard fails after all hops run, so a guard revert costs about the full execution (main.rs itself assumes 280,000 gas). | Replace with measured numbers from Stage 1. Budget and break-even in 1.2 must be recomputed. [UNVERIFIED] |
| C13 | 1.2 "zero-fee flash loans" | Aave V3 charges a premium; a Tier-0 flash swap pays the pool fee tier. Only Balancer is described as 0%. | Fees are inputs to requiredRepay and to profit estimates. Verify current Balancer and Aave fee parameters on-chain. [UNVERIFIED] |
| C14 | 5.11 tx.rs | fee_or_tier & 0xFFFFFF silently truncates. No route validation before encoding. | Reject fee > 0xFFFFFF; run validate_route() mirroring C3 before encoding. |
| C15 | 4.1 vs 5.7 | Spec text: Aging 111-220 ms, Expired > 250 ms. Code: anything over 220 is Expired and liquidations get no extra window. | Decision needed. v10.6 uses Aging 111..=250 for liquidations only; arbitrage expires after 110 ms. |
| C16 | 5.3 Governor | catch_unwind does nothing under panic = "abort". Spokes run sequentially so a slow one blocks the hot path. DEGRADED never returns to HEALTHY; TRIPPED only via manual revive(). | Cargo profile panic = "unwind"; run spokes with a per-spoke deadline; set HEALTHY on a successful call; auto-probe a TRIPPED spoke after a cool-down. |
| C17 | 5 "100% Complete Files" | simroom.rs has no revm. main.rs never connects to the feed, signs or sends. Governor has no registered spokes. partial.rs is never called. balancer_liquidity is faked with loan_amount. | Replace the claim with the status table in 4.5. |
| C18 | 1.1 / 4.2 / 5.5 ordering policy | The PGA / 125 ms rounds claim and competitor_priority_fee input are unsourced (see section 5). main.rs passes a constant 10,000,000 for the competitor fee. | Bidder is optional until the ordering policy is verified. Competitor fee must be measured from the feed or default to 0. [UNVERIFIED] |
| C19 | 5.2 / 5.10 simulation | CacheDB<EmptyDB> has no state, so simulations pass or fail meaninglessly. revm does not reproduce Arbitrum gas (L1 component, precompiles) exactly. | Back the DB with forked state at the target block; compare simulated vs actual gas in Stage 1. |
| C20 | 4.2 / 5.11 L1 fee | getCurrentTxL1GasFees() reports the fee of the current transaction, so it is not a pre-trade estimator off-chain. [UNVERIFIED] | Use the NodeInterface gas-estimate-components path for off-chain estimation and verify the exact call against Arbitrum docs. |

### 9.3 Unverified claims register (resolve before live funds)

| Claim in v10.5 | Required action |
|---|---|
| Arbitrum ordering moved to Priority Gas Auctions with 125 ms rounds; Entropy figures (87% / 98% / 97%); zero front-running claim | Read primary sources (governance proposal, Arbitrum docs). Until confirmed: bidder disabled (OC-09), decay constant treated as a tunable parameter. |
| Revert cost ~45k gas, $0.005-$0.02; revenue per event and events/day | Replace with measured Stage-1 data; recompute break-even. |
| Flash fees (Balancer 0%, Aave premium) | Read on-chain parameters at deploy time and at runtime. |
| getCurrentTxL1GasFees as an estimator; alloy/revm API details | Verify against Arbitrum docs; compile and test. |
| Router/token addresses; Atlas bond (0.1 ETH) | Confirm from official sources; encode in OC-08 self-check. |
| "Certora/Spearbit-Hardened" | Remove the label until an actual audit report exists. |

## 10. Release gates and capital ramp

| Gate | Exit criteria |
|---|---|
| G0 Build | Pinned compile, lints and static analysis clean, coverage targets met. |
| G1 Tests | All SC/MP/OC tests green, red-first evidence archived, fuzz/invariant/mutation thresholds met. |
| G2 Fork | Section 8.2 complete at multiple blocks with adversarial mocks passing. |
| G3 Formal | P1-P4 proven, or each gap documented with compensating tests. |
| G4 Audit | Independent audit done; findings fixed and re-tested. |
| G5 Shadow | >= 7 days and >= 500 opportunities on forked-state simulation; zero invariant violations; sim vs real drift within OP-02 tolerance; section 9.3 items resolved. |
| G6 Live ramp | Start with tiny maxLoan and a daily loss limit; raise (proposed: at most 2x per week) only while realized profit tracks simulation and no breaker tripped. Re-audit before large caps. |

## 11. Open decisions

- D-1 Age thresholds: spec text says Aging 111-220 ms and Expired > 250 ms; code differs. Proposed: arbitrage expires after 110 ms, liquidations Aging up to 250 ms.

- D-2 Enable Tier-0 (V3 flash swap) at launch or start with Balancer/Aave only (smaller attack surface).

- D-3 Liquidation scope: which collateral/debt pairs, close-factor handling, leftover debt token policy.

- D-4 Hop-token and pool whitelist contents; who may add and with what delay.

- D-5 Keep or drop Huff/Yul (MP-06).

- D-6 Spoke logic: RESOLVED as a reference design in section 12 (profitability still unproven; decided only by shadow data).

## 12. Strategy layer (v11.1): spokes, sizing, risk, measurement

### 12.1 Scope and honesty
This section supplies what v10.5 left out: how events become executable opportunities, how sizes are chosen for any pool type, how liquidations are evaluated, and how losses are limited and measured. It is a reference design. It does not contain an edge that is known to win against other searchers; whether it pays is decided only by the shadow-mode measurements in 12.5.

### 12.2 State and quoting model
- The sequencer feed carries transactions, not state. It is an early signal that a pool is about to change, not a source of reserves.
- Authoritative state: V2-style reserves from Sync logs (or multicall getReserves) each block, stored in PoolRegistry with the block number. A quote from reserves older than max_staleness_blocks MUST NOT be used (ST-08).
- Speculative overlay: apply_virtual_swap updates local V2 reserves with a pending swap so cycles are evaluated on the post-swap state. It is overwritten by the next authoritative update; the forked-state simulation remains the only gate.
- V3 / Camelot quoting: V2 is quoted locally and exactly. Concentrated-liquidity pools MUST be quoted by one of: (a) a local port of the swap loop with a tick-bitmap cache, validated against the on-chain quoter (ST-02); (b) QuoterV2 via eth_call on a node you operate; (c) revm on forked state. Option (a) is fastest and the most work.
- Sizing: maximize() (integer ternary search plus a final window scan) works for any quoter, with the cap equal to the on-chain maxLoan. For two constant-product pools the closed form of 4.6 is used as a cross-check (ST-03).
- Trigger rule: only cycles that contain the pool whose price just changed are evaluated; the cycle list is precomputed per base token.
- Units: every signal carries profit in loan-token units and in wei; the Valuer converts using a documented price source (ST-10). f64 is used only to prune candidates, never for money.

### 12.3 Strategy requirements (ST)
| ID | Requirement | Verified by |
|---|---|---|
| ST-01 | No signal is emitted with non-positive simulated net profit; every signal passes validate_route and uses only pools and hop tokens that are whitelisted on-chain. | Unit + property tests |
| ST-02 | The V3/Camelot quoter equals the on-chain quoter output (difference 0, or a documented rounding of at most 1 wei) on at least 100,000 random swaps across tick boundaries on a fork. | Fork differential test |
| ST-03 | maximize() returns profit >= the best of a grid of at least 1,000 points and >= the closed form for constant-product pairs, on random pools including tick-crossing cases. | Property tests |
| ST-04 | Cycle enumeration never reuses a pool, never exceeds MAX_HOPS legs, covers both directions, and the touching map contains every pool of every cycle. | Unit tests |
| ST-05 | Liquidation math: expected_collateral is monotone in cover, matches Aave's own calculation on a fork for sample positions; close-factor and bonus encodings are verified against live reserve configuration; positions with insufficient collateral are skipped. | Fork tests |
| ST-06 | Cooldown prevents duplicate signals for one cycle within cooldown_ms; poisoned locks fail closed. | Unit tests |
| ST-07 | Circuit breaker trips exactly at each configured limit, never auto-resets, and the trip results in executor.pause() through the guardian key; drilled before live funds. | Unit tests + drill |
| ST-08 | No quote from stale reserves; any sequence of virtual swaps and Sync updates never yields negative or overflowing reserves. | Fuzz |
| ST-09 | Malformed or random events never panic a spoke; p99 per-event latency stays within the Governor budget under replayed load. | cargo-fuzz + replay |
| ST-10 | The price source for valuation is documented, its staleness is bounded, and conversions are checked. | Review + tests |
| ST-11 | Synthetic scenario: pools with a known arbitrage produce the expected signal, size and profit (within the refinement window), and the fork simulation agrees. | Scenario tests |
| ST-12 | Signals referencing a non-whitelisted pool or hop token are never emitted; adding a pool follows the admin (timelock) procedure. | Unit tests |

### 12.4 Reference code (not compiled)
Module tree addition: strategy/{pools, search, cycle, liq, risk}.rs. The code uses the types from section 7 (SwapStep, ExecutionSignal, SpokeNode, OnChainEvent) and the checked math from math.rs.

#### 12.4.1 src/strategy/pools.rs

```text
// src/strategy/pools.rs  -- pool registry, local exact V2 state, speculative pending-swap overlay
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use alloy::primitives::{Address, U256};

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum PoolKind { V2 { fee_bps: u16 }, V3 { fee_tier: u32 }, Camelot }

#[derive(Clone, Debug)]
pub struct PoolInfo { pub address: Address, pub kind: PoolKind, pub token0: Address, pub token1: Address }
impl PoolInfo {
    pub fn protocol_id(&self) -> u8 { match self.kind { PoolKind::V2 { .. } => 0, PoolKind::V3 { .. } => 1, PoolKind::Camelot => 3 } }
    pub fn fee_or_tier(&self) -> u32 { match self.kind { PoolKind::V3 { fee_tier } => fee_tier, _ => 0 } }
    pub fn other(&self, t: Address) -> Option<Address> {
        if t == self.token0 { Some(self.token1) } else if t == self.token1 { Some(self.token0) } else { None }
    }
}

#[derive(Clone, Copy, Debug, Default)]
pub struct V2Reserves { pub r0: U256, pub r1: U256, pub block: u64 }

pub fn sorted_pair(a: Address, b: Address) -> (Address, Address) { if a < b { (a, b) } else { (b, a) } }

pub struct PoolRegistry {
    pools: Vec<PoolInfo>,
    by_pair: HashMap<(Address, Address), Vec<usize>>,
    by_token: HashMap<Address, Vec<usize>>,
    by_addr: HashMap<Address, usize>,
    v2: RwLock<HashMap<Address, V2Reserves>>,
}

impl PoolRegistry {
    pub fn new(pools: Vec<PoolInfo>) -> Self {
        let (mut by_pair, mut by_token, mut by_addr) = (HashMap::new(), HashMap::new(), HashMap::new());
        for (i, p) in pools.iter().enumerate() {
            by_pair.entry(sorted_pair(p.token0, p.token1)).or_insert_with(Vec::new).push(i);
            by_token.entry(p.token0).or_insert_with(Vec::new).push(i);
            by_token.entry(p.token1).or_insert_with(Vec::new).push(i);
            by_addr.insert(p.address, i);
        }
        Self { pools, by_pair, by_token, by_addr, v2: RwLock::new(HashMap::new()) }
    }
    pub fn pool(&self, a: Address) -> Option<&PoolInfo> { self.by_addr.get(&a).and_then(|i| self.pools.get(*i)) }
    pub fn pools_with_token(&self, t: Address) -> Vec<&PoolInfo> {
        self.by_token.get(&t).map(|v| v.iter().filter_map(|i| self.pools.get(*i)).collect()).unwrap_or_default()
    }
    pub fn pools_for_pair(&self, a: Address, b: Address) -> Vec<&PoolInfo> {
        self.by_pair.get(&sorted_pair(a, b)).map(|v| v.iter().filter_map(|i| self.pools.get(*i)).collect()).unwrap_or_default()
    }
    /// Authoritative update from Sync logs / multicall getReserves (the feed carries txs, not state).
    pub fn update_v2(&self, pool: Address, r: V2Reserves) { if let Ok(mut g) = self.v2.write() { g.insert(pool, r); } }
    pub fn v2_reserves(&self, pool: Address) -> Option<V2Reserves> { self.v2.read().ok()?.get(&pool).copied() }

    /// SPECULATIVE: apply a pending feed swap to local reserves so cycles are evaluated on the
    /// post-swap state. The next authoritative Sync overwrites it; the forked-state simulation
    /// remains the only gate that matters.
    pub fn apply_virtual_swap(&self, pool: Address, token_in: Address, amount_in: U256, block: u64) -> Option<()> {
        let info = self.pool(pool)?;
        let PoolKind::V2 { fee_bps } = info.kind else { return None };
        let mut g = self.v2.write().ok()?;
        let r = g.get_mut(&pool)?;
        let zero_in = token_in == info.token0;
        if !zero_in && token_in != info.token1 { return None; }
        let (rin, rout) = if zero_in { (r.r0, r.r1) } else { (r.r1, r.r0) };
        let out = crate::math::get_amount_out(amount_in, rin, rout, U256::from(fee_bps))?;
        let (new_in, new_out) = (rin.checked_add(amount_in)?, rout.checked_sub(out)?);
        if zero_in { r.r0 = new_in; r.r1 = new_out } else { r.r1 = new_in; r.r0 = new_out }
        r.block = block;
        Some(())
    }
}

/// Exact output for swapping `amount_in` through `pool`. None == not quotable (stale, unknown, overflow).
pub trait Quoter: Send + Sync {
    fn quote(&self, pool: &PoolInfo, token_in: Address, amount_in: U256, now_block: u64) -> Option<U256>;
}

pub struct V2LocalQuoter { pub reg: Arc<PoolRegistry>, pub max_staleness_blocks: u64 }
impl Quoter for V2LocalQuoter {
    fn quote(&self, pool: &PoolInfo, token_in: Address, amount_in: U256, now_block: u64) -> Option<U256> {
        let PoolKind::V2 { fee_bps } = pool.kind else { return None };
        let r = self.reg.v2_reserves(pool.address)?;
        if now_block.saturating_sub(r.block) > self.max_staleness_blocks { return None; }   // ST-10
        let (rin, rout) = if token_in == pool.token0 { (r.r0, r.r1) }
                          else if token_in == pool.token1 { (r.r1, r.r0) } else { return None };
        crate::math::get_amount_out(amount_in, rin, rout, U256::from(fee_bps))
    }
}

/// V2 locally and exactly (no RPC); everything else through `onchain`, which MUST be one of:
/// (a) a local port of the Uniswap V3 / Algebra swap loop validated against the on-chain Quoter,
/// (b) QuoterV2 eth_call on a node you run, (c) revm on forked state. See requirement ST-02.
pub struct RoutingQuoter { pub v2: V2LocalQuoter, pub onchain: Arc<dyn Quoter> }
impl Quoter for RoutingQuoter {
    fn quote(&self, pool: &PoolInfo, token_in: Address, amount_in: U256, now_block: u64) -> Option<U256> {
        match pool.kind {
            PoolKind::V2 { .. } => self.v2.quote(pool, token_in, amount_in, now_block),
            _ => self.onchain.quote(pool, token_in, amount_in, now_block),
        }
    }
}
```

#### 12.4.2 src/strategy/search.rs

```text
// src/strategy/search.rs  -- sizing that works for ANY pool type (does not need a closed form)
use alloy::primitives::{I256, U256};

pub fn signed_profit(out: U256, x: U256) -> Option<I256> {
    I256::try_from(out).ok()?.checked_sub(I256::try_from(x).ok()?)
}

/// Maximize f over integers in [lo, hi]. f MUST be unimodal on the interval (true for chains of
/// constant-product swaps and for concentrated liquidity inside one tick range; across tick
/// boundaries it is only approximately true, hence the final window scan and requirement ST-03).
/// Returns (x, profit) only when the best profit is strictly positive.
pub fn maximize<F: Fn(U256) -> Option<I256>>(f: F, lo: U256, hi: U256) -> Option<(U256, I256)> {
    if lo.is_zero() || lo > hi { return None; }
    let (mut l, mut h) = (lo, hi);
    let three = U256::from(3u64);
    let window = U256::from(16u64);
    while h.wrapping_sub(l) > window {
        let third = h.wrapping_sub(l) / three;
        let (m1, m2) = (l.checked_add(third)?, h.checked_sub(third)?);
        match (f(m1), f(m2)) {
            (Some(a), Some(b)) => { if a < b { l = m1 } else { h = m2 } }
            (None, Some(_)) => l = m1,
            (Some(_), None) => h = m2,
            (None, None) => return None,
        }
    }
    let mut best: Option<(U256, I256)> = None;
    let mut x = l;
    while x <= h {
        if let Some(p) = f(x) { if best.map_or(true, |(_, bp)| p > bp) { best = Some((x, p)); } }
        x = x.checked_add(U256::from(1u64))?;
    }
    best.filter(|(_, p)| p.is_positive())
}
```

#### 12.4.3 src/strategy/cycle.rs (2-pool and multi-leg cycles)

```text
// src/strategy/cycle.rs  -- one spoke for 2-pool arbitrage AND triangular / longer cycles
use std::collections::HashMap;
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::{Arc, Mutex};
use alloy::primitives::{Address, I256, U256};
use crate::hub::{ExecutionSignal, OnChainEvent, SpokeNode, SwapStep};
use super::pools::{PoolInfo, PoolRegistry, Quoter};
use super::search::{maximize, signed_profit};

#[derive(Clone, Debug)]
pub struct Leg { pub pool: PoolInfo, pub token_in: Address, pub token_out: Address }

pub trait Valuer: Send + Sync { fn to_wei(&self, token: Address, amount: U256) -> Option<U256>; }

/// All simple paths from `from` to `to` with at most `max_len` legs, no pool used twice.
/// Cycles are paths with from == to (>= 2 legs because a pool cannot pair a token with itself).
pub fn enumerate_paths(reg: &PoolRegistry, from: Address, to: Address, max_len: usize) -> Vec<Vec<Leg>> {
    fn dfs(reg: &PoolRegistry, cur: Address, to: Address, max_len: usize, path: &mut Vec<Leg>, out: &mut Vec<Vec<Leg>>) {
        if path.len() >= max_len { return; }
        for p in reg.pools_with_token(cur) {
            if path.iter().any(|l| l.pool.address == p.address) { continue; }
            let Some(next) = p.other(cur) else { continue };
            path.push(Leg { pool: p.clone(), token_in: cur, token_out: next });
            if next == to { out.push(path.clone()); } else { dfs(reg, next, to, max_len, path, out); }
            path.pop();
        }
    }
    let mut out = Vec::new();
    dfs(reg, from, to, max_len, &mut Vec::new(), &mut out);
    out
}

pub fn run_legs(q: &dyn Quoter, legs: &[Leg], x: U256, block: u64) -> Option<Vec<U256>> {
    let mut amounts = Vec::with_capacity(legs.len());
    let mut cur = x;
    for l in legs { cur = q.quote(&l.pool, l.token_in, cur, block)?; amounts.push(cur); }
    Some(amounts)
}

/// Cheap prune. f64 is for pruning ONLY, never for money.
pub fn spot_gain(q: &dyn Quoter, legs: &[Leg], probe: U256, block: u64) -> Option<f64> {
    let out = *run_legs(q, legs, probe, block)?.last()?;
    let (o, p) = (out.to_string().parse::<f64>().ok()?, probe.to_string().parse::<f64>().ok()?);
    if p > 0.0 { Some(o / p) } else { None }
}

pub fn slip(amount: U256, bps: u64) -> U256 {
    amount.checked_mul(U256::from(10_000u64.saturating_sub(bps))).map_or(amount, |v| v / U256::from(10_000u64))
}

pub struct CycleConfig {
    pub base_tokens: Vec<Address>,
    pub max_loan: HashMap<Address, U256>,   // mirrors on-chain maxLoan (OC-08 checks equality)
    pub min_profit_wei: U256,
    pub slippage_bps: u64,
    pub cooldown_ms: u64,
    pub max_legs: usize,                    // <= constants::MAX_HOPS
}

pub struct CycleSpoke {
    pub id: u8,
    reg: Arc<PoolRegistry>, quoter: Arc<dyn Quoter>, valuer: Arc<dyn Valuer>,
    cfg: CycleConfig, block: Arc<AtomicU64>,
    cycles: Vec<Vec<Leg>>,                       // precomputed
    touching: HashMap<Address, Vec<usize>>,      // pool -> cycle indexes
    recent: Mutex<HashMap<usize, u64>>,
}

impl CycleSpoke {
    pub fn new(id: u8, reg: Arc<PoolRegistry>, quoter: Arc<dyn Quoter>, valuer: Arc<dyn Valuer>,
               cfg: CycleConfig, block: Arc<AtomicU64>) -> Self {
        let mut cycles = Vec::new();
        for b in &cfg.base_tokens { cycles.extend(enumerate_paths(&reg, *b, *b, cfg.max_legs)); }
        let mut touching: HashMap<Address, Vec<usize>> = HashMap::new();
        for (i, c) in cycles.iter().enumerate() { for l in c { touching.entry(l.pool.address).or_default().push(i); } }
        Self { id, reg, quoter, valuer, cfg, block, cycles, touching, recent: Mutex::new(HashMap::new()) }
    }

    fn cooling(&self, idx: usize, ts: u64) -> bool {
        let Ok(g) = self.recent.lock() else { return true };                       // fail closed
        g.get(&idx).map_or(false, |last| ts.saturating_sub(*last) < self.cfg.cooldown_ms)
    }
    fn mark(&self, idx: usize, ts: u64) { if let Ok(mut g) = self.recent.lock() { g.insert(idx, ts); } }

    fn evaluate(&self, idx: usize, ts: u64, block: u64) -> Option<(ExecutionSignal, U256)> {
        let legs = self.cycles.get(idx)?;
        let loan_token = legs.first()?.token_in;
        let cap = *self.cfg.max_loan.get(&loan_token)?;
        if cap.is_zero() { return None; }
        let probe = (cap / U256::from(10_000u64)).max(U256::from(1u64));
        if spot_gain(self.quoter.as_ref(), legs, probe, block)? <= 1.0 { return None; }   // prune
        let f = |x: U256| -> Option<I256> {
            let last = *run_legs(self.quoter.as_ref(), legs, x, block)?.last()?;
            signed_profit(last, x)
        };
        let (x, profit) = maximize(f, U256::from(1u64), cap)?;
        let amounts = run_legs(self.quoter.as_ref(), legs, x, block)?;
        let last_idx = legs.len().checked_sub(1)?;
        let mut steps = Vec::with_capacity(legs.len());
        for (i, l) in legs.iter().enumerate() {
            let min_out = if i == last_idx { x }                                   // never end below principal
                          else { slip(*amounts.get(i)?, self.cfg.slippage_bps) };
            steps.push(SwapStep { protocol_id: l.pool.protocol_id(), pool: l.pool.address,
                token_in: l.token_in, token_out: l.token_out, fee_or_tier: l.pool.fee_or_tier(), min_out });
        }
        let profit_tok = profit.into_raw();                                         // positive by construction
        let profit_wei = self.valuer.to_wei(loan_token, profit_tok)?;
        if profit_wei < self.cfg.min_profit_wei { return None; }
        Some((ExecutionSignal { spoke_id: self.id, loan_token, loan_amount: x, steps, liquidation: None,
            expected_profit_loan_token: profit_tok, expected_gross_profit_wei: profit_wei,
            event_timestamp_ms: ts }, profit_wei))
    }
}

impl SpokeNode for CycleSpoke {
    fn id(&self) -> u8 { self.id }
    fn name(&self) -> &'static str { "cycle-arb" }
    fn on_event(&self, ev: &Arc<OnChainEvent>) -> Option<ExecutionSignal> {
        let (pool, ts) = match ev.as_ref() {
            OnChainEvent::SequencerSwap { pool, timestamp_ms, .. } => (*pool, *timestamp_ms),
            _ => return None,
        };
        let block = self.block.load(Ordering::Relaxed);
        let mut best: Option<(ExecutionSignal, U256)> = None;
        // Only cycles that contain the pool whose price just changed can have become profitable.
        for idx in self.touching.get(&pool)? {
            if self.cooling(*idx, ts) { continue; }
            if let Some((sig, w)) = self.evaluate(*idx, ts, block) {
                if best.as_ref().map_or(true, |(_, bw)| w > *bw) { self.mark(*idx, ts); best = Some((sig, w)); }
            }
        }
        best.map(|(s, _)| s)
    }
}
```

#### 12.4.4 src/strategy/liq.rs

```text
// src/strategy/liq.rs  -- Aave V3 liquidation spoke
use std::collections::HashMap;
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use alloy::primitives::{Address, U256};
use crate::hub::{ExecutionSignal, LiquidationTarget, OnChainEvent, SpokeNode, SwapStep};
use super::cycle::{enumerate_paths, run_legs, slip, Valuer};
use super::pools::{PoolRegistry, Quoter};

pub const WAD: u64 = 1_000_000_000_000_000_000;

/// All values come from the node / Aave contracts. Every encoding below must be verified (ST-05).
pub trait AaveReader: Send + Sync {
    fn user_debt(&self, user: Address, debt_asset: Address) -> Option<U256>;
    fn user_collateral(&self, user: Address, collateral: Address) -> Option<U256>;
    fn close_factor_bps(&self, hf_wad: U256) -> u16;                 // 5000, or 10000 below the threshold (verify)
    fn liquidation_bonus_bps(&self, collateral: Address) -> Option<u16>; // e.g. 10500 = 5% bonus (verify encoding)
    fn price(&self, asset: Address) -> Option<U256>;                 // oracle price, one common base unit
    fn decimals(&self, asset: Address) -> Option<u8>;
}

/// collateral = cover * P_debt * 10^dec_col * bonus / (P_col * 10^dec_debt * 10_000)
pub fn expected_collateral(cover: U256, p_debt: U256, dec_debt: u8, p_col: U256, dec_col: u8, bonus_bps: u16) -> Option<U256> {
    let ten = U256::from(10u64);
    let num = cover.checked_mul(p_debt)?
        .checked_mul(ten.checked_pow(U256::from(dec_col))?)?.checked_mul(U256::from(bonus_bps))?;
    let den = p_col.checked_mul(ten.checked_pow(U256::from(dec_debt))?)?.checked_mul(U256::from(10_000u64))?;
    num.checked_div(den)
}

pub struct LiqConfig {
    pub max_loan: HashMap<Address, U256>, pub min_profit_wei: U256,
    pub slippage_bps: u64, pub flash_fee_bps: u64, pub route_max_len: usize,
}

pub struct LiqSpoke {
    pub id: u8, pub reg: Arc<PoolRegistry>, pub quoter: Arc<dyn Quoter>, pub valuer: Arc<dyn Valuer>,
    pub aave: Arc<dyn AaveReader>, pub cfg: LiqConfig, pub block: Arc<AtomicU64>,
}

impl SpokeNode for LiqSpoke {
    fn id(&self) -> u8 { self.id }
    fn name(&self) -> &'static str { "aave-liquidation" }
    fn on_event(&self, ev: &Arc<OnChainEvent>) -> Option<ExecutionSignal> {
        let OnChainEvent::HealthFactorDrop { borrower, collateral, debt, hf_wad, timestamp_ms } = ev.as_ref() else { return None };
        if *hf_wad >= U256::from(WAD) { return None; }
        let block = self.block.load(Ordering::Relaxed);

        let total_debt = self.aave.user_debt(*borrower, *debt)?;
        let cap = *self.cfg.max_loan.get(debt)?;
        let by_cf = total_debt.checked_mul(U256::from(self.aave.close_factor_bps(*hf_wad)))?.checked_div(U256::from(10_000u64))?;
        let cover = by_cf.min(cap);
        if cover.is_zero() { return None; }

        let col_out = expected_collateral(cover, self.aave.price(*debt)?, self.aave.decimals(*debt)?,
            self.aave.price(*collateral)?, self.aave.decimals(*collateral)?, self.aave.liquidation_bonus_bps(*collateral)?)?;
        if col_out > self.aave.user_collateral(*borrower, *collateral)? { return None; }   // conservative: skip partial-collateral cases

        // best route collateral -> debt asset
        let mut best: Option<(Vec<super::cycle::Leg>, Vec<U256>)> = None;
        for path in enumerate_paths(&self.reg, *collateral, *debt, self.cfg.route_max_len) {
            if let Some(a) = run_legs(self.quoter.as_ref(), &path, col_out, block) {
                let fin = *a.last()?;
                if best.as_ref().map_or(true, |(_, ba)| ba.last().map_or(true, |b| fin > *b)) { best = Some((path, a)); }
            }
        }
        let (path, amounts) = best?;
        let proceeds = *amounts.last()?;
        let fee = cover.checked_mul(U256::from(self.cfg.flash_fee_bps))?.checked_div(U256::from(10_000u64))?;
        let net = proceeds.checked_sub(cover.checked_add(fee)?)?;                 // None => not profitable
        if net.is_zero() { return None; }
        let net_wei = self.valuer.to_wei(*debt, net)?;
        if net_wei < self.cfg.min_profit_wei { return None; }

        let last = path.len().checked_sub(1)?;
        let mut steps = Vec::with_capacity(path.len());
        for (i, l) in path.iter().enumerate() {
            let min_out = if i == last { cover } else { slip(*amounts.get(i)?, self.cfg.slippage_bps) };
            steps.push(SwapStep { protocol_id: l.pool.protocol_id(), pool: l.pool.address, token_in: l.token_in,
                token_out: l.token_out, fee_or_tier: l.pool.fee_or_tier(), min_out });
        }
        Some(ExecutionSignal { spoke_id: self.id, loan_token: *debt, loan_amount: cover, steps,
            liquidation: Some(LiquidationTarget { borrower: *borrower, collateral_asset: *collateral,
                debt_asset: *debt, debt_to_cover: cover }),
            expected_profit_loan_token: net, expected_gross_profit_wei: net_wei, event_timestamp_ms: *timestamp_ms })
    }
}
```

#### 12.4.5 src/strategy/risk.rs

```text
// src/strategy/risk.rs  -- loss limits and reconciliation (OP-01, OP-02). Fails closed on poisoned locks.
use std::sync::Mutex;

#[derive(Clone, Copy, Debug)]
pub struct TradeOutcome { pub simulated_net_wei: i128, pub realized_net_wei: i128, pub reverted: bool }

#[derive(Debug, PartialEq, Eq)]
pub enum BreakerAction { Continue, PauseOnChain }

pub struct BreakerConfig {
    pub daily_loss_limit_wei: u128,
    pub max_consecutive_failures: u32,
    pub deviation_bps: u128,        // |realized - simulated| / |simulated| above this counts as a deviation
    pub max_deviations: usize,      // within window_ms
    pub window_ms: u64,
}
struct State { day_start_ms: u64, pnl_wei: i128, consec_fail: u32, deviations: Vec<u64>, tripped: bool }
pub struct CircuitBreaker { cfg: BreakerConfig, st: Mutex<State> }

impl CircuitBreaker {
    pub fn new(cfg: BreakerConfig, now_ms: u64) -> Self {
        Self { cfg, st: Mutex::new(State { day_start_ms: now_ms, pnl_wei: 0, consec_fail: 0, deviations: Vec::new(), tripped: false }) }
    }
    pub fn is_tripped(&self) -> bool { self.st.lock().map(|s| s.tripped).unwrap_or(true) }

    /// Call after every executed (or reverted) transaction. PauseOnChain => call executor.pause() via the guardian key.
    pub fn record(&self, now_ms: u64, o: TradeOutcome) -> BreakerAction {
        let Ok(mut s) = self.st.lock() else { return BreakerAction::PauseOnChain };
        if now_ms.saturating_sub(s.day_start_ms) >= 86_400_000 { s.day_start_ms = now_ms; s.pnl_wei = 0; }
        s.pnl_wei = s.pnl_wei.saturating_add(o.realized_net_wei);
        s.consec_fail = if o.reverted || o.realized_net_wei <= 0 { s.consec_fail.saturating_add(1) } else { 0 };

        let sim = o.simulated_net_wei.unsigned_abs();
        let dev = o.realized_net_wei.saturating_sub(o.simulated_net_wei).unsigned_abs();
        if sim > 0 && dev.saturating_mul(10_000).checked_div(sim).map_or(true, |d| d > self.cfg.deviation_bps) {
            s.deviations.push(now_ms);
        }
        let w = self.cfg.window_ms;
        s.deviations.retain(|t| now_ms.saturating_sub(*t) <= w);

        let loss_hit = s.pnl_wei < 0 && s.pnl_wei.unsigned_abs() >= self.cfg.daily_loss_limit_wei;
        if loss_hit || s.consec_fail >= self.cfg.max_consecutive_failures || s.deviations.len() >= self.cfg.max_deviations {
            s.tripped = true;
            return BreakerAction::PauseOnChain;
        }
        BreakerAction::Continue
    }
    /// Manual re-arm only, after review (never automatic).
    pub fn reset(&self, now_ms: u64) {
        if let Ok(mut s) = self.st.lock() { *s = State { day_start_ms: now_ms, pnl_wei: 0, consec_fail: 0, deviations: Vec::new(), tripped: false }; }
    }
}
```

#### 12.4.6 Wiring

```text
// main(): wiring sketch (inside the tokio runtime)
let reg = Arc::new(PoolRegistry::new(load_pools()?));                 // curated pool list, not "all pools"
let block = Arc::new(AtomicU64::new(0));
let quoter: Arc<dyn Quoter> = Arc::new(RoutingQuoter {
    v2: V2LocalQuoter { reg: reg.clone(), max_staleness_blocks: 2 }, onchain: v3_quoter });
let mut gov = Governor::new(5_000 /*cooldown ms*/, 2_000 /*budget us*/);
gov.register(Box::new(CycleSpoke::new(1, reg.clone(), quoter.clone(), valuer.clone(), cycle_cfg, block.clone())), 3);
gov.register(Box::new(LiqSpoke { id: 2, reg: reg.clone(), quoter, valuer, aave, cfg: liq_cfg, block: block.clone() }), 3);

// ingestion loop, per decoded feed tx that touches a tracked V2 pool:
//   reg.apply_virtual_swap(pool, token_in, amount_in, block_now);      // speculative overlay
//   for sig in gov.broadcast(Arc::new(event), now_ms) { handle_signal(...) }
// state refresh (separate task): subscribe to Sync logs / multicall getReserves each block
//   -> reg.update_v2(..); block.store(n)
// after every tx result: breaker.record(now_ms, outcome) -> on PauseOnChain call executor.pause()
```

### 12.5 Shadow-mode measurement (the test that decides whether to continue)
Run at least 7 days on live feed and live state, with no broadcast. Record, per signal: event time, decision-ready time, simulation-done time, simulated gross, estimated gas and L1 cost, net, and whether the opportunity still existed at blocks +1, +2, +3 after our decision time.
| KPI | Definition | Use |
|---|---|---|
| Decision latency | event timestamp -> signal ready, and -> simulation done (p50/p95/p99) | Compare against how long opportunities survive |
| Signals/day, sim-pass rate, guard-pass rate | Counts | Pipeline health |
| Would-send set | Signals with net after all costs above the minimum | Candidate trades |
| Survival rate S(L) | Fraction of would-send opportunities still profitable at the state our latency L would reach | Upper bound on win probability |
| Captured-by-others rate | Fraction where another transaction extracted the same cycle first (check later blocks) | Competition intensity |
| Sim-vs-state drift | Simulated profit vs profit recomputed at the next block | Model error |
Upper-bound expected value per month = would_send * S * avg_net - would_send * (1 - S) * avg_failure_cost - infrastructure_cost. Because S is an upper bound on the true win probability, this overstates reality. Proposed rule: go live only if this upper bound is at least 3x the monthly cost AND decision latency is within the survival window; the thresholds are proposals to be tuned on real data. If the rule fails, the answer is a no-go or an infrastructure upgrade from 12.6, not a smaller-size live test.

### 12.6 Levers firms use (not required for correctness; justify with 12.5 data)
| Lever | Why | Effort | Worth it when |
|---|---|---|---|
| Record and replay harness (feed + state, deterministic) | Regression tests and parameter tuning on real history | Medium | From day one |
| Own Arbitrum node near the feed, local state and eth_call | Removes RPC latency | High | Latency is dominated by RPC |
| Local V3/Algebra swap simulator with tick cache | Avoids eth_call per quote | High | V3 pools dominate the opportunities |
| Pre-built transaction templates, pre-warmed connections, several send endpoints | Cuts milliseconds on the send path | Medium | Survival window is tight |
| Executor gas optimization (Huff/Yul hot paths) | Lower gas per execution | High | Only after the full test suite is green (D-5) |
| Wider curated pool and venue list with on-chain whitelist automation | More opportunities | Medium | Signals per day are the bottleneck |
| Multi-region redundancy and failover | Availability | Medium | Live and profitable |
| Dynamic minimum profit by gas regime; better price sources | Fewer losing trades | Low | After first live data |
| Priority/express submission routes | Ordering advantage | Unknown | Only after the ordering policy is verified from primary sources |

### 12.7 What is outside this document
- The edge itself: which opportunities are worth taking and the tuned parameters. They come from data.
- Implementations of the adapters behind ChainView, ExecutorReader, Sender, SignerHandle, Valuer, AaveReader and the V3 quoter; they need a node, keys infrastructure and testing.
- Hosting, monitoring stack, capital, legal and tax matters, and an independent audit (gate G4).

### 12.8 Gate additions
G5 (Shadow) additionally requires the 12.5 KPIs to be recorded and the decision rule to be evaluated and written down. G2 additionally requires ST-02 and ST-05 on a fork. G1 additionally requires ST-01, ST-03, ST-04, ST-06 to ST-09, ST-11, ST-12.

