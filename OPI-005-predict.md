# OPI: 5

**Title**: The "Predict" Standard: Trustless Prediction Markets on Bitcoin Layer 1 (LMSR Edition)

**Author**: Laz1m0v <wtf@wbtc.lol>

**Discussions-To**: https://t.me/universalbrc20

**Status**: Draft

**Type**: Standards Track

**Created**: 2026-01-20

**Requires**: BRC-20, OPI-3 (W Standard - for vault architecture)

**Note**: OPI-5 uses **LMSR (Logarithmic Market Scoring Rule)** bonding curves for pricing, providing price = probability for prediction markets. Each market has isolated pools with LMSR pricing, identified via vault addresses (template model).

---

## Abstract

This document specifies the **"Predict" Standard** (OPI-5), a protocol for creating **trustless prediction markets** directly on Bitcoin Layer 1. The standard enables users to create binary markets (YES/NO outcomes) where participants can trade positions using **LMSR (Logarithmic Market Scoring Rule)** bonding curves, with outcomes resolved through oracle inputs and payouts distributed automatically based on market resolution.

The protocol introduces dedicated trading operations (`predict buy` and `predict sell`) that leverage **isolated market pools with LMSR pricing** while maintaining market identification via vault addresses (template model). By using Bitcoin's immutable timechain as the settlement layer and LMSR bonding curves specifically designed for prediction markets, OPI-5 creates the first native prediction market protocol on Bitcoin that requires no trusted intermediaries, no smart contracts, and no Layer 2 dependencies.

**Key Innovation**: Prediction markets use **isolated pools per market** with **LMSR bonding curves** for pricing, where **price = probability**. Market identification via vault addresses (template model) eliminates the need for explicit market metadata in OP_RETURN payloads. Markets are identified by the vault address in transaction outputs, following the same principle as resolution transactions.

**Trust-Minimized Architecture**: The protocol achieves trust-minimization by cryptographically binding outcomes and payout distributions directly to Bitcoin L1:
- **Outcome Hash**: Embedded in resolution scripts, cryptographically binding outcome to vault
- **Merkle Root**: Committed in OP_RETURN, enabling independent verification of payouts by traders
- **Cryptographic Verification**: All validations are cryptographic (hash verification, Merkle proofs), not interpretation-based

**Trust Assumptions**:
- **Oracle Dependency**: The protocol requires trust in the oracle to resolve markets correctly. Bitcoin Script verifies the oracle's signature and outcome hash, but does NOT verify that the outcome matches real-world events.
- **Indexer Dependency**: Indexers validate cryptographic proofs and maintain virtual state. While cryptographic proofs are independently verifiable, indexers must correctly interpret transaction structure and maintain consensus.

This design eliminates the need for complex smart contracts while maintaining cryptographic guarantees through Bitcoin's consensus layer, achieving trust-minimization with clear documentation of remaining trust assumptions.

---

## Copyright

This OPI is released under the MIT License.

---

## Motivation

Prediction markets have proven to be powerful tools for information aggregation, risk hedging, and decentralized governance. However, existing implementations on Ethereum and other smart contract platforms suffer from:

1. **Smart Contract Risk**: Vulnerable to bugs, hacks, and upgrade risks
2. **Oracle Dependencies**: Require trusted oracles or complex oracle networks
3. **High Gas Costs**: Expensive to create and trade markets
4. **Layer 2 Complexity**: Often require Layer 2 solutions for scalability
5. **Pricing Issues**: AMM-based pricing creates disconnect between price and probability

OPI-5 solves these problems by:

- **No Smart Contracts**: Markets are virtual state managed by indexers, following the proven meta-protocol model of Ordinals and BRC-20
- **Bitcoin Security**: Settlement happens on Bitcoin's immutable timechain
- **LMSR Pricing**: Uses Logarithmic Market Scoring Rule where **price = probability** (essential for prediction markets)
- **Isolated Pools**: Each market has its own isolated pool, preventing cross-market manipulation
- **Low Cost**: Minimal on-chain footprint (OP_RETURN payloads)
- **Trustless Resolution**: Oracle inputs are verifiable on-chain, with multi-sig or decentralized oracle support
- **Template Model**: Market identification via vault addresses (no explicit market field needed)

This standard enables a new class of applications on Bitcoin: from political prediction markets to sports betting, from weather derivatives to corporate event markets, all while maintaining Bitcoin's core principles of self-custody and cryptographic security.

---

## Specification

### Core Concepts

**Prediction Market**: A binary market where participants trade YES and NO shares, each representing a fixed stake of 100,000 sats (0.001 BTC). The market resolves to either YES (outcome occurred) or NO (outcome did not occur), with winning share holders receiving the full pool of sats locked in the market.

**YES Share**: A fixed-value share representing the "YES" outcome. Each YES share is worth exactly 100,000 sats (0.001 BTC) when purchased. When the market resolves to YES, YES share holders receive payouts proportional to their share count, with each share receiving its proportional share of the total pool.

**NO Share**: A fixed-value share representing the "NO" outcome. Each NO share is worth exactly 100,000 sats (0.001 BTC) when purchased. When the market resolves to NO, NO share holders receive payouts proportional to their share count, with each share receiving its proportional share of the total pool.

**Fixed Share Model**: All YES and NO shares have a fixed claim value of 100,000 sats at resolution. When a user buys a share, they pay the current market price (determined by LMSR, which equals probability), but the share itself represents a claim to 100,000 sats of the total pool at resolution. This creates a simple, predictable model where:
- Each share represents a claim to 100,000 sats at resolution (fixed claim value)
- Purchase price is determined by LMSR (varies with probability)
- Example: If YES price = 0.6 (60% probability), buying 1 YES share costs ~60,000 sats
- At resolution, if YES wins, each YES share receives: `(total_pool_sats) / (total_yes_shares)`
- The 100,000 sats is the **claim value**, not the purchase price

**LMSR (Logarithmic Market Scoring Rule)**: A bonding curve specifically designed for prediction markets where:
- **Price = Probability**: The price of YES shares equals the probability that YES will occur
- **YES + NO = 100%**: By mathematical construction, price_yes + price_no always equals 100%
- **Better for Low Liquidity**: LMSR works well even with minimal liquidity, unlike AMM
- **Battle-Tested**: Used by Augur, Gnosis, Polymarket

**Isolated Market Pools**: Each prediction market has its own isolated pool with LMSR pricing:
- **Market-Specific Pool**: Each market has its own YES and NO share tracking
- **No Cross-Market Manipulation**: Isolation prevents cross-market attacks
- **LMSR Pricing**: Price calculated using LMSR formula based on yes_shares and no_shares
- **Market Identification**: Markets identified via vault address (template model)

**Resolution**: The process of determining the market outcome (YES or NO) based on oracle input. Once resolved, the market is closed and all sats locked in the pool are distributed to winning share holders proportionally.

**Lock Period**: The duration in Bitcoin blocks before which the market cannot be resolved. This prevents premature resolution and ensures sufficient time for trading.

---

## Transaction Structure

### Deploy Transaction

A `predict deploy` transaction creates a new prediction market:

| Component | Description |
| --------- | ------------------------------------------- |
| **Input[0]** | UTXO from the creator. The address controlling this input becomes the **Market Creator Address**. |
| **Output[0]** | `OP_RETURN` output containing the JSON payload with the `predict` operation |
| **Output[1+]** | Change back to creator or other optional outputs |

**Market Creator Address** (KISS - Template Model):
- Creator address is **deduced from Input[0]** of deploy transaction
- No explicit `creator` field needed in OP_RETURN (simpler, more KISS)
- Address controlling Input[0] = creator address (stored in market record)
- Creator receives royalties from each share purchase (incentivizes market creation)
- Similar to NFP operator model - address is deduced from transaction structure
- Creator has **no protocol power**—it cannot resolve markets or alter parameters

#### Resolution Transaction

A `predict resolve` transaction resolves an active market:

| Component | Description |
| --------- | ------------------------------------------- |
| **Input[0]** | Market Vault UTXO (identifies market via vault address) |
| **Output[0]** | `OP_RETURN` output containing the JSON payload with the `resolve` operation |
| **Output[1+]** | Payouts to winning share holders (verifiable via Merkle proof) |
| **Output[N]** | Oracle fee for resolution |

**Resolver Authority**: Oracle address is discovered from the first share purchase transaction. Only transactions signed by this address (or a multi-sig) can resolve the market.

#### Claim Transaction (Optional)

A `predict claim` transaction allows users to claim payouts after resolution:

| Component | Description |
| --------- | ------------------------------------------- |
| **Input[0+]** | User's UTXOs (for transaction fees) |
| **Output[0]** | `OP_RETURN` output containing the JSON payload with the `claim` operation |
| **Output[1+]** | Change back to user or other optional outputs |

**Note**: Payouts can also be distributed automatically by the indexer without requiring explicit claim transactions. The claim mechanism provides flexibility for users who prefer explicit control.

---

## OP_RETURN Payloads

### Deploy Payload

Initializes a new prediction market:

```json
{
  "p": "brc-20",
  "op": "predict",
  "deploy": "BTC_100K_2026",
  "question": "Will Bitcoin reach $100,000 USD before 2027?",
  "lock": "52560"
}
```

**Required Fields:**
- `p`: MUST be `"brc-20"`
- `op`: MUST be `"predict"`
- `deploy`: Market identifier (alphanumeric, max 64 chars). Used as prefix for YES/NO tokens
- `question`: Human-readable question describing the market (max 512 chars)
- `lock`: Lock period in Bitcoin blocks (string representation of integer, minimum 144 blocks ≈ 1 day)

**Optional Fields:**
- `category`: Market category (e.g., `"crypto"`, `"politics"`, `"sports"`)
- `description`: Extended description (max 2048 chars)
- `oracle_fee_bps`: Oracle fee in basis points per share purchase (default: 100 = 1%)
- `creator_royalty_bps`: Creator royalty in basis points per share purchase (default: 50 = 0.5%)
- `oracle_resolution_fee_bps`: Oracle fee in basis points for resolution (default: 100 = 1% of total pool)
- `lmsr_b`: LMSR liquidity parameter (default: 100000). Higher `b` = more liquidity, less price impact

**Creator Address** (KISS - Template Model):
- Creator address is **deduced from deploy transaction** (like NFP operator)
- Address controlling `Input[0]` of deploy transaction = creator address
- No need for explicit `creator` field in OP_RETURN
- Creator receives royalties from each share purchase
- Creator address stored in market record for validation

**Oracle Address** (KISS - Template Model):
- Oracle address is **NOT in the deploy JSON** - it is discovered automatically via the template model
- Oracle address appears in **every transaction** of the market:
  - In share purchase transactions: Oracle receives fees (output to oracle address)
  - In resolution transaction: Oracle signs the transaction (signature from oracle address)
- Indexer discovers oracle address from the first share purchase transaction (where oracle receives fees)
- Oracle address is then stored in market record and validated on all subsequent transactions
- This follows the NFP template model - address is deduced from transaction structure, not explicit fields
- **Benefits**: Simpler JSON (KISS), enables market discovery via oracle address, consistent with NFP pattern

**Effect:**
1. Creates market record with identifier `BTC_100K_2026`
2. **Extracts creator address** from deploy transaction: `creator_address = address_controlling_input[0]` (Template Model - like NFP)
3. Stores creator address in market record for royalty distribution
4. **Oracle address is NOT stored yet** - it will be discovered from the first share purchase transaction (template model)
5. **Automatically deploys YES share token**: `BTC_100K_2026_YES` (standard BRC-20 deploy, created by indexer)
   - Token ticker format: `{market_id}_YES` (e.g., `BTC_100K_2026_YES`)
   - Deployed as standard BRC-20 token with:
     - `max_supply`: Unlimited (elastic supply)
     - `dec`: 0 (integer shares only)
     - Deployed automatically by indexer (not user action)
6. **Automatically deploys NO share token**: `BTC_100K_2026_NO` (standard BRC-20 deploy, created by indexer)
   - Token ticker format: `{market_id}_NO` (e.g., `BTC_100K_2026_NO`)
   - Deployed as standard BRC-20 token with:
     - `max_supply`: Unlimited (elastic supply)
     - `dec`: 0 (integer shares only)
     - Deployed automatically by indexer (not user action)
7. **Derives Market Vault address** from market_txid (txid of deploy transaction):
   - Vault address is deterministically derived: `vault_address = derive_vault_address(market_txid)`
   - Uses BIP 341 standard NUMS point
   - Vault address stored in market record for market identification (template model)
8. **Initializes isolated market pool** with LMSR pricing:
   - Pool starts with zero YES and NO shares
   - LMSR parameter `b` set from deploy (default: 100000)
   - Initial prices: 50% YES, 50% NO (equal probability)

---

## LMSR Bonding Curve Architecture

### Overview

OPI-5 uses **LMSR (Logarithmic Market Scoring Rule)** bonding curves for pricing prediction markets. Each market has its own isolated pool with LMSR pricing, providing:

- **Price = Probability**: The price of YES shares equals the probability that YES will occur
- **YES + NO = 100%**: By mathematical construction, price_yes + price_no always equals 100%
- **Isolation**: Each market is isolated, preventing cross-market manipulation
- **Low Liquidity Friendly**: LMSR works well even with minimal liquidity

### LMSR Formula

The LMSR pricing formula calculates prices based on the number of YES and NO shares:

```python
import math

def calculate_lmsr_price(yes_shares, no_shares, b=100000):
    """
    Calculate LMSR prices for YES and NO shares.
    
    Args:
        yes_shares: Number of YES shares outstanding
        no_shares: Number of NO shares outstanding
        b: Liquidity parameter (default: 100000)
    
    Returns:
        (price_yes, price_no): Prices as probabilities (0.0 to 1.0)
    """
    if yes_shares == 0 and no_shares == 0:
        # Empty market: initial 50/50 probability
        return 0.5, 0.5
    
    # Calculate exponential terms
    exp_yes = math.exp(yes_shares / b)
    exp_no = math.exp(no_shares / b)
    denominator = exp_yes + exp_no
    
    # Calculate prices (probabilities)
    price_yes = exp_yes / denominator
    price_no = exp_no / denominator
    
    # Property: price_yes + price_no = 100% (always)
    assert abs(price_yes + price_no - 1.0) < 1e-10
    
    return price_yes, price_no
```

**Properties**:
- ✅ **Price = Probability**: `price_yes` equals the probability that YES will occur
- ✅ **YES + NO = 100%**: `price_yes + price_no = 1.0` (always, by construction)
- ✅ **Symmetric**: If `yes_shares = no_shares`, then `price_yes = price_no = 0.5`
- ✅ **Monotonic**: More YES shares → higher `price_yes`, lower `price_no`

### Cost Calculation

The cost to buy shares is calculated using the LMSR cost function:

```python
def calculate_lmsr_cost(yes_shares_before, no_shares_before, 
                        yes_shares_after, no_shares_after, b=100000):
    """
    Calculate the cost (in sats) to move from one state to another.
    
    Cost = b × ln(exp(yes_after/b) + exp(no_after/b)) 
           - b × ln(exp(yes_before/b) + exp(no_before/b))
    """
    before = b * math.log(math.exp(yes_shares_before / b) + 
                          math.exp(no_shares_before / b))
    after = b * math.log(math.exp(yes_shares_after / b) + 
                         math.exp(no_shares_after / b))
    return after - before
```

**Example**:
- Initial state: 0 YES, 0 NO shares
- User buys 100 YES shares
- Cost = `calculate_lmsr_cost(0, 0, 100, 0, b=100000)`
- Result: User pays cost in sats, receives 100 YES shares

### Market Pool State

Each market has its own isolated pool state:

```python
class IsolatedMarketPool:
    def __init__(self, market_id, lmsr_b=100000):
        self.market_id = market_id
        self.yes_shares = Decimal("0")  # Total YES shares outstanding
        self.no_shares = Decimal("0")    # Total NO shares outstanding
        self.lmsr_b = Decimal(str(lmsr_b))  # LMSR liquidity parameter
        
    def get_prices(self):
        """Get current YES and NO prices (probabilities)"""
        return calculate_lmsr_price(
            yes_shares=self.yes_shares,
            no_shares=self.no_shares,
            b=self.lmsr_b
        )
    
    def buy_yes(self, amount_sats):
        """
        Buy YES shares with sats.
        Returns: (yes_shares_received, cost_paid)
        """
        # Calculate how many YES shares to mint
        # This uses iterative calculation or lookup table
        # For simplicity, we calculate cost for incremental shares
        
        # Start from current state
        yes_before = self.yes_shares
        no_before = self.no_shares
        
        # Calculate target YES shares (incremental)
        # In practice, this uses binary search or lookup
        target_yes = yes_before + self._calculate_shares_for_cost(amount_sats, "YES")
        
        # Calculate actual cost
        cost = calculate_lmsr_cost(
            yes_before, no_before,
            target_yes, no_before,
            b=self.lmsr_b
        )
        
        # Update state
        yes_shares_received = target_yes - yes_before
        self.yes_shares = target_yes
        
        return yes_shares_received, cost
    
    def buy_no(self, amount_sats):
        """Buy NO shares with sats. Similar to buy_yes."""
        # ... (similar implementation)
        pass
```

### Market Identification via Vault Address

**Template Model** (KISS Principle):
- Markets are identified by **vault address** in transaction outputs
- No explicit `market` field needed in OP_RETURN
- Indexer looks up market by vault address: `market = get_market_by_vault_address(vault_address)`
- Vault address is deterministically derived from `market_txid` at deploy
- This follows the NFP template model - address deduced from transaction structure

**Mapping at Deploy**:
```python
def process_predict_deploy(tx, op_data):
    """
    Process market deployment.
    Creates vault address mapping and isolated pool.
    """
    market_txid = tx.txid  # Txid of deploy transaction
    market_id = op_data["deploy"]
    lmsr_b = op_data.get("lmsr_b", 100000)  # Default LMSR parameter
    
    # Derive vault address
    vault_address = derive_market_vault_address(market_txid)
    
    # Create market record
    market = Market(
        market_txid=market_txid,
        market_id=market_id,
        vault_address=vault_address,  # Index for lookup
        lmsr_b=lmsr_b,
        ...
    )
    
    # Create isolated pool with LMSR
    pool = IsolatedMarketPool(
        market_id=market_id,
        lmsr_b=lmsr_b
    )
    
    # Store mapping
    vault_to_market[vault_address] = market_txid
    market_pools[market_id] = pool
```

---

## Trading Operations

### Buying YES Shares

To buy YES shares (bet that YES will win):

```json
{
  "p": "brc-20",
  "op": "predict",
  "buy": "YES",
  "amt": "100000",
  "slip": "0.5"
}
```

**Transaction Structure** (Template Model):
```
Inputs:  [0] User's native Bitcoin UTXOs (sats from user's wallet)
Outputs: [0] OP_RETURN (predict buy payload)
         [1] Market Vault P2TR (sats locked - MUST be first non-OP_RETURN)
         [2] Creator royalty output (e.g., 500 sats = 0.5% of 100,000)
         [3] Oracle fee output (e.g., 1,000 sats = 1% of 100,000)
           - **Oracle address discovered here** (first share purchase transaction)
         [4] Change to user (if any)
```

**Important**: 
- User sends **native Bitcoin sats** (UTXOs from their wallet) - NOT a "BTC token"
- Oracle address is **discovered** from the oracle fee output in the first share purchase transaction
- Indexer stores oracle address in market record after first share purchase
- All subsequent transactions must use the same oracle address (validated by indexer)
- Market is identified by **vault address** in transaction outputs (template model)

**LMSR Pricing Calculation**:
1. Indexer identifies market from vault address: `market = get_market_by_vault_address(vault_address)`
2. Indexer gets isolated pool: `pool = market_pools[market.market_id]`
3. Indexer calculates YES shares using LMSR:
   - Current state: `yes_shares`, `no_shares`
   - User pays: `amount_sats` (after fees)
   - Calculate: `yes_shares_received = pool.buy_yes(amount_sats)`
4. Indexer mints YES tokens and credits to user
5. Indexer updates pool state

**Fee Calculation**:
- Each share purchase: 100,000 sats base value
- Oracle fee: `oracle_fee_bps / 10000 × 100000` (e.g., 100 bps = 1% = 1,000 sats)
- Creator royalty: `creator_royalty_bps / 10000 × 100000` (e.g., 50 bps = 0.5% = 500 sats)
- Locked in vault: `100000 - oracle_fee - creator_royalty` (e.g., 98,500 sats)
- Oracle receives: `oracle_fee` (e.g., 1,000 sats)
- Creator receives: `creator_royalty` (e.g., 500 sats)

### Buying NO Shares

To buy NO shares (bet that NO will win):

```json
{
  "p": "brc-20",
  "op": "predict",
  "buy": "NO",
  "amt": "100000",
  "slip": "0.5"
}
```

**Important**: Same as buying YES - you use **native Bitcoin sats**, and NO tokens are minted and credited to your wallet. The market is identified by the vault address in the transaction outputs.

### Selling YES Shares

To sell YES shares (exit position):

```json
{
  "p": "brc-20",
  "op": "predict",
  "sell": "YES",
  "amt": "100000",
  "slip": "0.5"
}
```

**LMSR Pricing Calculation**:
1. Indexer identifies market from token ticker or vault address
2. Indexer gets isolated pool: `pool = market_pools[market.market_id]`
3. Indexer calculates sats received using LMSR:
   - Current state: `yes_shares`, `no_shares`
   - User sells: `yes_shares_to_sell`
   - Calculate: `sats_received = pool.sell_yes(yes_shares_to_sell)`
4. User receives native Bitcoin sats
5. YES tokens are burned
6. Indexer updates pool state

### Selling NO Shares

To sell NO shares (exit position):

```json
{
  "p": "brc-20",
  "op": "predict",
  "sell": "NO",
  "amt": "100000",
  "slip": "0.5"
}
```

**Important**: Same as selling YES - you receive **native Bitcoin sats** directly, and your NO tokens are burned. The market is identified from the token ticker or vault address (template model).

**Price Discovery**: Prices are determined by the **LMSR formula** in the isolated market pool. As more users buy YES shares, the price of YES shares increases (probability increases) and the price of NO shares decreases (probability decreases), naturally reflecting market probability estimates. The LMSR ensures that **price = probability** and **YES + NO = 100%** always.

---

## Market Vault Architecture

### Problem Statement

Unlike standard AMM pools where liquidity is virtual (tracked by indexers), prediction markets require **physical sats locking** to guarantee that payouts can be distributed at resolution. We need a mechanism that:

1. **Locks sats physically** when shares are purchased
2. **Prevents premature withdrawal** until market resolution
3. **Guarantees distribution** to winning share holders
4. **Maintains composability** with other OPI standards

### Solution: Market Vault Architecture

Each prediction market uses a **Market Vault** (similar to OPI-3 W vaults) that physically locks sats in a Taproot address with spend paths that enforce payout distribution.

#### Market Vault Structure

The Market Vault is a P2TR (Pay-to-Taproot) address with three spend paths:

```
Market_Vault_Address = P2TR(NUMS_POINT, MAST_Root)
```

**NUMS Point**: Provably unspendable point (BIP 341 standard):
```python
# BIP 341 Standard NUMS Point (Security Fix - prevents quantum attack surface multiplication)
# From BIP 341: H = lift_x(0x50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0)
# This is derived from hashing the secp256k1 base point G
NUMS_POINT = lift_x(0x50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0)
```

**Why Standard NUMS Point**: Using market-specific NUMS points multiplies the quantum attack surface (each market becomes a separate target). The BIP 341 standard NUMS point is provably unspendable and reduces attack surface to a single global point.

#### The Three Spend Paths

**Design Principle**: KISS (Keep It Simple, Stupid). Each path is minimal, verifiable, and elegant.

##### Path 1: Resolution Payout (YES Outcome)
```bitcoin
# Leaf Version: 0xc0 (Tapscript)
OP_SHA256 <outcome_yes_hash> OP_EQUALVERIFY
<oracle_pubkey> OP_CHECKSIG
```

**Spend Condition**:
- Outcome hash verification (cryptographically binds outcome to vault)
- Oracle signature required (prevents unauthorized spending)

**Outcome Hash** (calculated at deploy):
- `outcome_yes_hash = SHA256("YES:" + market_id)`
- This hash is embedded in the script at deploy, cryptographically binding the YES outcome to the vault

##### Path 2: Resolution Payout (NO Outcome)
```bitcoin
# Leaf Version: 0xc0 (Tapscript)
OP_SHA256 <outcome_no_hash> OP_EQUALVERIFY
<oracle_pubkey> OP_CHECKSIG
```

**Spend Condition**:
- Outcome hash verification (cryptographically binds outcome to vault)
- Oracle signature required (prevents unauthorized spending)

**Outcome Hash** (calculated at deploy):
- `outcome_no_hash = SHA256("NO:" + market_id)`
- This hash is embedded in the script at deploy, cryptographically binding the NO outcome to the vault

##### Path 3: Emergency Refund (Time-Locked)
```bitcoin
# Leaf Version: 0xc0 (Tapscript)
<lock_period + grace_period> OP_CHECKSEQUENCEVERIFY OP_DROP
<market_creator_pubkey> OP_CHECKSIG
```

**Spend Condition**:
- Time-lock must expire (lock_period + grace_period blocks)
- Creator signature required (allows proportional refund if oracle fails)

**Purpose**: Safety net if oracle fails to resolve. Users can recover funds after time-lock expires.

---

## Resolution Process

### Oracle Resolution Transaction

The oracle creates a resolution transaction with:

**Input**:
- Market Vault UTXO (identifies market via vault address)

**Outputs**:
- **Output[0]**: OP_RETURN with:
  ```json
  {
    "p": "brc-20",
    "op": "predict",
    "outcome": "YES",
    "payouts_merkle_root": "<merkle_root_hex>",
    "total_pool": <total_pool>,
    "total_winning_shares": <total_winning_shares>
  }
  ```
- **Output[1+]**: Payouts to winning share holders (verifiable via Merkle proof)
- **Output[N]**: Oracle fee for resolution (e.g., 1% of total pool)

**Witness**:
- Oracle signature
- Outcome preimage: `"YES:" + market_id` (revealed in witness)
- Path 1 or Path 2 script + control block

**How It Works** (Cryptographic Verification):
1. Oracle creates resolution transaction with outcome preimage in witness
2. **Bitcoin Script verifies** (Layer 1 - Cryptographic Verification):
   - `OP_SHA256 <outcome_preimage> OP_EQUALVERIFY` → Verifies outcome hash matches
   - `<oracle_pubkey> OP_CHECKSIG` → Verifies oracle signature
3. Transaction executes, sats distributed to winning share holders
4. Market marked as resolved
5. User balances updated (payouts credited)

---

## Indexer Validation Rules

Indexers that support this OPI **MUST** validate:

### Deploy Validation

1. **Market ID Uniqueness**: Market identifier MUST be unique (not already deployed)
2. **Question Format**: Question MUST be non-empty string (max 512 chars)
3. **Lock Period**: Lock MUST be >= 144 blocks (minimum 1 day)
4. **Token Deployment**: YES and NO tokens MUST be **automatically deployed** by indexer as standard BRC-20 tokens
5. **Pool Initialization**: Isolated market pool with LMSR MUST be **automatically initialized** by indexer (starts with 0 YES and 0 NO shares)
6. **Oracle Address** (KISS - Template Model):
   - Oracle address is **NOT in deploy OP_RETURN** - it is discovered automatically
   - Oracle address is discovered from the **first share purchase transaction** (oracle fee output)
   - Indexer stores oracle address in market record after first share purchase
   - All subsequent transactions must use the same oracle address (validated by indexer)
   - Similar to NFP operator model - address deduced from transaction structure
7. **Creator Address** (KISS - Template Model):
   - Creator address is **deduced from Input[0]** of deploy transaction
   - Indexer extracts: `creator_address = address_controlling_input[0]`
   - No explicit field needed - simpler and more KISS
   - Similar to NFP operator model - address deduced from transaction structure
   - Creator address stored in market record for royalty validation
8. **Vault Address Derivation**: Market Vault address MUST be derived deterministically from market_id using BIP 341 standard NUMS point
9. **Share Value**: Fixed share value MUST be 100,000 sats (enforced by indexer)
10. **LMSR Parameter**: `lmsr_b` MUST be > 0 (default: 100000)

### Trading Validation

1. **Market Existence**: Market MUST exist and not be resolved
2. **Vault Address Validation**: Vault address in transaction outputs MUST match `market.vault_address`
3. **LMSR Price Calculation**: Prices MUST be calculated using LMSR formula with correct `lmsr_b` parameter
4. **YES + NO = 100%**: Indexer MUST verify that `price_yes + price_no = 1.0` (within floating point tolerance)
5. **Fee Validation**: Creator royalty and oracle fee outputs MUST match calculated amounts
6. **Vault Locking**: Sats MUST be locked in vault according to share count and fees
7. **Token Minting/Burning**: YES/NO tokens MUST be minted on buy, burned on sell

### Resolution Validation

1. **Market Existence**: Market MUST exist and not be already resolved
2. **Lock Period**: Current block height MUST be >= `deploy_height + lock`
3. **Resolver Authority**: Resolver address (from signature) MUST match `market.oracle_address` (discovered from first share purchase)
4. **Outcome Format**: Outcome MUST be exactly `"YES"` or `"NO"`
5. **Vault State**: Market Vault MUST exist and contain locked sats
6. **Spend Path Validation**:
   - Transaction MUST use correct spend path (Path 1 for YES, Path 2 for NO)
   - Oracle signature MUST be valid and match `market.oracle_address`
   - OP_RETURN MUST contain valid resolution payload
7. **Payout Verification**:
   - Sum of payout outputs MUST equal vault input (minus fees)
   - Payout amounts MUST match calculated proportions
   - Each winning share holder MUST receive proportional payout
8. **Share Burning**: After resolution, indexer MUST mark shares as burned (prevents double-claiming)

---

## Data Structures

### Market Record

Stores market metadata and state:

```python
class Market(Base):
    market_id: str  # Primary key
    question: str
    creator_address: str  # Deduced from deploy Input[0] (Template Model - KISS)
    deploy_txid: str
    deploy_height: int
    lock_blocks: int  # Lock period in blocks
    vault_address: str  # Deterministically derived (Template Model - for market identification)
    
    yes_ticker: str  # "MARKET_ID_YES"
    no_ticker: str   # "MARKET_ID_NO"
    
    lmsr_b: int  # LMSR liquidity parameter (default: 100000)
    
    oracle_address: str | None  # Discovered from first share purchase transaction (Template Model - KISS)
    creator_royalty_bps: int  # Creator royalty in basis points (default: 50 = 0.5%)
    oracle_fee_bps: int  # Oracle fee per share purchase in basis points (default: 100 = 1%)
    oracle_resolution_fee_bps: int  # Oracle resolution fee in basis points (default: 100 = 1%)
    
    resolved: bool
    outcome: str | None  # "YES" or "NO"
    resolution_txid: str | None
    resolution_height: int | None
```

### Isolated Market Pool

Stores LMSR pool state for each market:

```python
class IsolatedMarketPool(Base):
    market_id: str  # Primary key (FK to Market)
    yes_shares: Decimal  # Total YES shares outstanding
    no_shares: Decimal   # Total NO shares outstanding
    lmsr_b: Decimal      # LMSR liquidity parameter
    
    def get_prices(self) -> Tuple[float, float]:
        """Get current YES and NO prices (probabilities)"""
        return calculate_lmsr_price(
            yes_shares=self.yes_shares,
            no_shares=self.no_shares,
            b=self.lmsr_b
        )
```

### Share Balance Record

Stores user share balances per market:

```python
class ShareBalance(Base):
    user_address: str
    market_id: str
    yes_shares: Decimal
    no_shares: Decimal
```

### Payout Record

Stores payout information for resolved markets:

```python
class PayoutRecord(Base):
    market_id: str
    user_address: str
    share_type: str  # "YES" or "NO"
    shares: Decimal
    payout_amount: Decimal  # In sats
    payout_txid: str | None
    merkle_proof: str | None  # Hex-encoded Merkle proof
```

---

## Advantages of LMSR over AMM

### 1. Price = Probability

**LMSR**: Price directly equals probability (essential for prediction markets)
- `price_yes = probability(YES will occur)`
- Intuitive for users: price reflects actual probability

**AMM**: Price determined by liquidity ratio (not probability)
- `price = reserve_ratio` (not necessarily probability)
- Can diverge from true probability

### 2. YES + NO = 100% (Always)

**LMSR**: By mathematical construction, `price_yes + price_no = 100%` always
- Guaranteed property of LMSR formula
- No arbitrage opportunities from price mismatch

**AMM**: `price_yes + price_no` can diverge from 100%
- Creates arbitrage opportunities
- Can lead to market inefficiencies

### 3. Better for Low Liquidity

**LMSR**: Works well even with minimal liquidity
- No need for initial liquidity provision
- Prices remain meaningful even with few trades

**AMM**: Requires significant liquidity for accurate pricing
- Low liquidity → high slippage
- Prices can be manipulated easily

### 4. Isolation

**LMSR with Isolated Pools**: Each market is completely isolated
- No cross-market manipulation possible
- Security maximized

**AMM with Global Pools**: Cross-market manipulation possible
- Vulnerability identified in counter-audit
- Requires pools isolation anyway

---

## Economic Risks and Considerations

### Price/Value Relationship in LMSR

**Issue**: With LMSR, price = probability, but share value at resolution is still 100k sats.

**Example Scenario**:
```
Market deploys: 0 YES, 0 NO shares (50/50 probability)
User A buys 100 YES @ price = 0.5 (50% probability) = 50k sats per share
LMSR adjusts: YES now costs ~0.6 (60% probability) = 60k sats per share
User B buys 100 YES @ price = 0.6 = 60k sats per share
Market resolves YES
User A receives: 100 × 147.75k = 14.775M sats (195% ROI)
User B receives: 100 × 147.75k = 14.775M sats (97% ROI)
```

**Impact**: 
- Early buyers can exploit probability changes
- But this reflects genuine information aggregation (not manipulation)
- LMSR ensures price = probability (more accurate than AMM)

**Mitigation**:
- This is a **feature, not a bug** - it reflects market dynamics and information aggregation
- Users should understand that purchase price reflects current probability estimate
- LMSR pricing naturally adjusts to reflect collective probability estimates
- Price = Probability ensures accurate information aggregation

**Recommendation**: Users should carefully consider market probability estimates when entering positions.

### Isolated Pools (No Cross-Market Risk)

**Architectural Choice**: Each market has isolated pools with LMSR pricing.

**Benefits**:
- ✅ **No Cross-Market Manipulation**: Complete isolation prevents cross-market attacks
- ✅ **Security Maximized**: Bug in one market doesn't affect others
- ✅ **Price Accuracy**: Each market's price reflects only that market's probability

**Trade-off**:
- ⚠️ **Liquidité Fragmentée**: Each market has its own liquidity
- ⚠️ **Slippage**: Can be higher than global pools (but acceptable for security)

**Recommendation**: Isolation is worth the trade-off for security.

---

## Fee Structure

### Share Purchase Fees

- **Oracle Fee**: `oracle_fee_bps / 10000 × 100000` sats per share (default: 1% = 1,000 sats)
- **Creator Royalty**: `creator_royalty_bps / 10000 × 100000` sats per share (default: 0.5% = 500 sats)
- **Total Fees per Share**: `(oracle_fee_bps + creator_royalty_bps) / 10000 × 100000`
- **Locked in Vault**: `100000 - total_fees` sats per share

### Resolution Fee

- **Oracle Resolution Fee**: `oracle_resolution_fee_bps / 10000 × total_pool` sats (default: 1%)
- Paid from vault when market resolves

---

## Security Considerations

### Attack Vectors and Mitigations

| Attack Vector | Mitigation |
|---------------|------------|
| **Oracle Griefing** | Time-lock emergency refund path (Path 3) |
| **False Resolution** | Cryptographic outcome hash binding + Oracle signature verification |
| **Merkle Proof Manipulation** | Merkle root committed in OP_RETURN, independently verifiable |
| **LMSR Price Manipulation** | Isolated pools prevent cross-market manipulation |
| **Share Double-Spending** | Indexer marks shares as burned after resolution |
| **Vault Address Spoofing** | Deterministic derivation from market_txid, verifiable off-chain |

---

## Trust Assumptions and Security Model

### Explicit Trust Dependencies

OPI-5 achieves **trust-minimization**, not "100% trustlessness". The following trust assumptions remain:

#### 1. Oracle Trust

**What Bitcoin Script Verifies**:
- ✅ Oracle signature validity
- ✅ Outcome hash matches preimage
- ✅ Transaction structure compliance

**What Bitcoin Script Does NOT Verify**:
- ❌ Outcome matches real-world events
- ❌ Oracle is honest
- ❌ Payout recipients are correct winners

**Risk**: Oracle can resolve incorrectly (e.g., resolve YES when real-world outcome is NO).

**Mitigation**:
- Oracle reputation system (future work)
- Multi-oracle consensus (M-of-N) for high-stakes markets (future work)
- Dispute period for challenging resolutions (future work)
- Emergency refund path (Path 3) if oracle fails to resolve

**Oracle Griefing Risk**:
- **Attack Vector**: Market creator bribes oracle to delay resolution, forcing users to wait for emergency refund (time-locked)
- **Impact**: Users' funds locked until time-lock expires, even if market outcome is clear
- **Mitigation**:
  - Oracle has financial incentive to resolve (resolution fees)
  - Time-lock prevents indefinite locking (emergency refund available after lock period + grace period)
  - Users should verify oracle reputation before participating
- **Trade-off**: Simplicity (no slashing mechanism) vs. oracle accountability

**Recommendation**: Users should only participate in markets with trusted oracles or wait for multi-oracle support. Verify oracle reputation and track resolution history.

#### 2. Indexer Trust

**What Indexers Do**:
- Validate cryptographic proofs (Merkle roots, outcome hashes)
- Maintain virtual state (token balances, pool reserves)
- Calculate LMSR prices
- Interpret transaction structure

**What Indexers Cannot Do**:
- Modify Bitcoin Script validation (L1 enforces)
- Change cryptographic proofs (Merkle roots are on-chain)
- Alter payout calculations (Merkle proofs are independently verifiable)
- Change LMSR formula (mathematical, verifiable)

**Risk**: Indexers may diverge in state interpretation or LMSR calculation.

**Mitigation**:
- Cryptographic proofs are independently verifiable (users can verify Merkle proofs)
- LMSR formula is mathematical and verifiable
- First valid indexer establishes canonical state
- Bitcoin blockchain is ultimate source of truth
- Users can challenge indexer state with cryptographic proofs

**Recommendation**: Users should verify Merkle proofs independently or use multiple indexers.

### Security Guarantees

**Cryptographic Guarantees** (L1 Enforced):
- ✅ Outcome hash binding (embedded in Tapscript)
- ✅ Oracle signature verification (Bitcoin Script)
- ✅ Merkle root commitment (in OP_RETURN)
- ✅ Merkle proof verification (independent verification possible)

**Mathematical Guarantees** (LMSR):
- ✅ Price = Probability (LMSR formula)
- ✅ YES + NO = 100% (always, by construction)
- ✅ Isolated pools (no cross-market manipulation)

**Operational Guarantees** (Indexer Behavior):
- ✅ Transaction structure validation
- ✅ Cryptographic proof verification
- ✅ LMSR price calculation (mathematical, verifiable)
- ⚠️ Virtual state consistency (depends on indexer consensus)

### Security Model Summary

| Component | Trust Level | Verification Method |
|-----------|-------------|---------------------|
| **Outcome Hash** | ✅ Cryptographic | Bitcoin Script (L1) |
| **Oracle Signature** | ✅ Cryptographic | Bitcoin Script (L1) |
| **Merkle Root** | ✅ Cryptographic | Independent verification |
| **LMSR Prices** | ✅ Mathematical | Formula verification |
| **Outcome Correctness** | ⚠️ Oracle Trust | No L1 verification |
| **Payout Recipients** | ⚠️ Oracle Trust | Merkle proof verifies structure, not correctness |
| **Indexer State** | ⚠️ Indexer Trust | Cryptographic proofs independently verifiable |

**Conclusion**: OPI-5 achieves **trust-minimization** through cryptographic binding and mathematical guarantees (LMSR), but explicit trust in oracle and indexer remains. Users should understand these assumptions before participating.

---

## Indexer Consensus Mechanism

**Problem**: Multiple indexers may process transactions in different orders or interpret edge cases differently. A consensus mechanism is required to ensure consistent state across all indexers.

**Consensus Rules**:

1. **First Valid Indexer Wins**: The first indexer to validate and process a transaction correctly establishes the canonical state. Other indexers MUST accept this state if cryptographic proofs are valid.
   - **Ordering**: Based on block height and transaction index within block
   - **Validation**: Indexer must verify all cryptographic proofs before establishing state
   - **Persistence**: Once established, state persists unless challenged with valid cryptographic proof

2. **Cryptographic Verification**: All indexers MUST independently verify:
   - Bitcoin Script validation (signatures, hashes)
   - Merkle root/proof verification
   - Transaction structure compliance
   - LMSR price calculations (mathematical verification)
   - Vault state invariants (locked_sats == sum(shares × share_value_after_fees))

3. **State Reconciliation**: If indexers diverge:
   - **Step 1**: Compare cryptographic proofs (Merkle roots, outcome hashes)
   - **Step 2**: Indexer with valid cryptographic proof wins
   - **Step 3**: If both have valid proofs but different interpretations:
     - Compare block heights (earlier block wins)
     - Compare transaction indices (earlier transaction wins)
     - First-to-confirm wins (based on canonical ordering)
   - **Step 4**: If divergence persists, Bitcoin blockchain is canonical source

4. **Dispute Resolution**: Users can challenge indexer state by providing cryptographic proofs (Merkle proofs, transaction signatures). Indexers MUST accept valid cryptographic challenges.
   - **Challenge Format**: User provides Merkle proof + transaction signature
   - **Validation**: Indexer verifies proof against on-chain Merkle root
   - **Resolution**: If challenge is valid, indexer updates state accordingly

5. **Canonical State Source**: Bitcoin blockchain is the ultimate source of truth. Indexers MUST derive state from on-chain data, not from other indexers.
   - **On-chain Data**: Transaction outputs, OP_RETURN payloads, witness data
   - **Derived State**: Token balances, pool reserves, market state, LMSR prices
   - **Verification**: All derived state must be verifiable from on-chain data

6. **Persistent Divergence Handling**: If indexers never converge:
   - **User Action**: Users can query multiple indexers and compare results
   - **Cryptographic Proof**: Users can independently verify using Merkle proofs
   - **Mathematical Verification**: Users can verify LMSR price calculations
   - **Finality**: Bitcoin blockchain provides finality (on-chain transactions are immutable)
   - **Recommendation**: Users should use indexers with good reputation and track record

**Implementation Note**: This consensus mechanism is similar to BRC-20's approach - first valid indexer establishes state, others follow if cryptographic proofs are valid. The mechanism relies on social consensus (indexer reputation) rather than cryptographic consensus (like Bitcoin's proof-of-work), which is acceptable for meta-protocols that operate on top of Bitcoin.

---

## Backwards Compatibility

OPI-5 is fully backwards compatible:

- Uses standard BRC-20 JSON format
- YES/NO tokens are standard BRC-20 tokens (can be traded on any BRC-20 exchange)
- Legacy indexers will ignore `predict` operations (graceful degradation)
- No breaking changes to existing BRC-20 or OPI standards
- Market vault addresses are standard P2TR (compatible with Bitcoin wallets)

---

## Reference Implementation

- **Repo**: https://github.com/universal-brc20/simplicity
- **Branch**: opi-5-predict
- **Examples**: 
  - Market deployment transactions
  - Share purchase/sale transactions
  - Resolution transactions with Merkle proofs
  - LMSR price calculation examples

---

## Test Vectors

### 1. Market Deploy Transaction

**OP_RETURN Payload**:
```json
{
  "p": "brc-20",
  "op": "predict",
  "deploy": "BTC_100K_2026",
  "question": "Will Bitcoin reach $100,000 before 2027?",
  "lock": "52560",
  "lmsr_b": "100000"
}
```

**Expected Behavior**:
- Market ID: `BTC_100K_2026`
- YES token: `BTC_100K_2026_YES` (auto-deployed)
- NO token: `BTC_100K_2026_NO` (auto-deployed)
- Vault address: Derived from deploy transaction txid
- Initial pool state: 0 YES shares, 0 NO shares
- Initial prices: 50% YES, 50% NO (equal probability)

### 2. Buy YES Shares Transaction

**OP_RETURN Payload**:
```json
{
  "p": "brc-20",
  "op": "predict",
  "buy": "YES",
  "amt": "100000"
}
```

**Transaction Structure**:
- Input[0]: User UTXOs (sats)
- Output[0]: OP_RETURN (0 sats)
- Output[1]: Market Vault P2TR (sats locked)
- Output[2]: Creator royalty
- Output[3]: Oracle fee (oracle address discovered here)

**LMSR Calculation Example**:
- Initial state: 0 YES, 0 NO shares
- User pays: 100,000 sats (after fees)
- LMSR calculation: `yes_shares_received = calculate_lmsr_shares(100000, "YES", 0, 0, b=100000)`
- Result: ~100 YES shares received (exact amount depends on LMSR formula)
- New state: 100 YES, 0 NO shares
- New prices: ~100% YES, ~0% NO (probability shifted)

### 3. Resolution Transaction

**OP_RETURN Payload**:
```json
{
  "p": "brc-20",
  "op": "predict",
  "outcome": "YES",
  "payouts_merkle_root": "0xabc123...",
  "total_pool": "10000000",
  "total_winning_shares": "500"
}
```

**Transaction Structure**:
- Input[0]: Market Vault UTXO
- Output[0]: OP_RETURN (resolution payload)
- Output[1+]: Payouts to winners (Merkle proof verifiable)
- Output[N]: Oracle fee

**Witness**:
- Oracle signature
- Outcome preimage: `"YES:BTC_100K_2026"`
- Script (Path 1: YES resolution)
- Control block

**Validation**:
- Outcome hash: `SHA256("YES:BTC_100K_2026")` matches script
- Oracle signature: Valid Schnorr signature
- Merkle root: Matches OP_RETURN payload
- Payouts: Verifiable via Merkle proofs

---

## Conclusion

OPI-5 establishes Bitcoin as a foundation for **trust-minimized prediction markets** where cryptographic security reduces trust requirements. Through **isolated market pools with LMSR pricing**, template-based market identification, and cryptographic outcome/payout binding, it achieves:

1. **Simplicity**: Binary markets with clear YES/NO outcomes, market identification via vault addresses (template model)
2. **Security**: Bitcoin's immutable timechain as settlement layer, vault addresses cryptographically bound to markets, isolated pools prevent cross-market manipulation
3. **Trust-Minimization**: 
   - Outcome cryptographically bound to vault (hash in script)
   - Payouts cryptographically verifiable (Merkle root in OP_RETURN)
   - Cryptographic verification reduces indexer interpretation dependency
   - Traders can verify independently (Merkle proofs)
   - **Trust Assumptions Documented**: Oracle dependency and indexer consensus requirements clearly stated
4. **Accurate Pricing**: LMSR ensures price = probability (essential for prediction markets)
5. **Composability**: Integrates seamlessly with OPI-2, OPI-3, OPI-4, and B.R.A.I.N.
6. **Low Cost**: Minimal on-chain footprint via OP_RETURN, no explicit market fields needed (KISS)

This is not just a prediction market protocol. It's a paradigm shift toward **trust-minimized** information markets on Bitcoin, with **isolated pools and LMSR pricing**, template-based identification, and cryptographic guarantees that reduce (but do not eliminate) dependencies on indexer interpretation and oracle honesty.

**The age of Bitcoin-native prediction markets begins with OPI-5.**

---

## References

- BIP 340: Schnorr Signatures
- BIP 341: Taproot
- BIP 342: Tapscript
- BRC-20: Token Standard
- OPI-1: Swap Standard
- OPI-2: Curve Extension
- OPI-3: W Standard
- OPI-4: Vault Standard
- LMSR: Logarithmic Market Scoring Rule (Hanson, 2003)

