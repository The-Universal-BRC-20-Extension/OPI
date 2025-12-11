OPI: 2

Title: The "Curve" Extension for Trustless Programmatic Token Distribution

Author: M3rl1n

Discussions-To: https://t.me/universalbrc20

Status: Draft

Type: Standards Track

Created: 2025-12-10

---

## Abstract

This OPI specifies the `"curve"` extension, a standard for **trustless, finite, and algorithmic token emission** on Bitcoin Layer 1. It allows deployers to define an immutable distribution constitution via a specific `deploy` payload. Rewards are **minted ex-nihilo by indexers just-in-time at the moment of claim**, eliminating pre-mined supplies and treasury risks. User interactions are handled via context-aware usage of the `swap` OPI, enabling the creation of **yTokens**—native **Yield-Bearing Receipt Tokens**. A fixed, on-chain **"Genesis Fee"** mechanism ensures sustainable development for protocol creators without compromising decentralization or granting administrative keys.

---

## Copyright

This OPI is released under the MIT License.

---

## Motivation

Token distribution mechanisms in DeFi have historically suffered from critical vulnerabilities:

1. **Treasury Risk:** Pre-mined tokens stored in multi-sig wallets are prone to hacks or insider theft (rug pulls).
2. **Centralization:** Emission schedules are often alterable by admin keys.
3. **Unsustainability:** Protocol developers often rely on inflationary developer shares or venture capital, misaligning incentives.

OPI-2 resolves these by introducing a purely mathematical model where:
- **Trust is Code:** No tokens exist until they are earned.
- **Immutable Economics:** Once deployed, the emission curve cannot be changed.
- **Sustainable Funding:** Hardcoded "Genesis Fees" provide revenue to the creator based strictly on protocol usage.

This extension not only mitigates historical DeFi risks but also positions Bitcoin as a competitive base layer for sophisticated yield farming and liquidity mining, fostering innovation while upholding the network's security and decentralization principles.

---

## Specification

### Transaction Structure

#### Deploy Transaction

The `deploy` transaction MUST adhere to the following structure to define the Genesis Fees:

| Component | Description |
| --------- | ------------------------------------------- |
| Input[0] | UTXO from the deployer. The address controlling this input becomes the immutable **Genesis Address**. |
| Output[0] | OP_RETURN output containing the JSON payload with the `curve` extension |
| Output[1] | **Staking Fee Definition.** The exact amount of sats (e.g., 3000) that users must pay to the Genesis Address when calling `init`. This value becomes the immutable fee constant for all future `init` operations. |
| Output[2] | **Claiming Fee Definition.** The exact amount of sats (e.g., 1000) that users must pay to the Genesis Address when calling `exe`. This value becomes the immutable fee constant for all future `exe` operations. |
| Output[3+] | Change back to deployer or other optional outputs |

**Genesis Address:** The address controlling Input[0] is resolved and stored by indexers at deployment. It has **no protocol power**—it cannot mint, burn, pause, or alter anything. Its sole function is as a universal anchor for tracking, discoverability, and receiving Genesis Fees.

#### Staking Transaction (`swap init`)

| Component | Description |
| --------- | ------------------------------------------- |
| Input[0+] | User's UTXOs containing the token to stake (e.g., `WTF`) |
| Output[0] | OP_RETURN output with the `swap init` payload |
| Output[1] | **Mandatory Genesis Fee** to Genesis Address. Value MUST equal the exact amount defined in `deploy.Output[1]`. If not, the transaction is **INVALID**. |
| Output[2+] | Change to user or other outputs |

#### Claiming Transaction (`swap exe`)

| Component | Description |
| --------- | ------------------------------------------- |
| Input[0+] | User's UTXOs containing the yToken (e.g., `yWTF`) |
| Output[0] | OP_RETURN output with the `swap exe` payload |
| Output[1] | **Mandatory Genesis Fee** to Genesis Address. Value MUST equal the exact amount defined in `deploy.Output[2]`. If not, the transaction is **INVALID**. |
| Output[2+] | Change to user or other outputs |

### OP_RETURN Payload

#### Deploy Payload

To initialize a Curve program, the deployer embeds a `curve` object in the standard BRC-20 `deploy` inscription:

```json
{
  "p": "brc-20",
  "op": "deploy",
  "tick": "REWARD",
  "max": "4200000000",
  "curve": {
    "type": "linear",
    "lock": "105120",
    "stakes": ["WTF"]
  }
}
```

**Required Fields:**
- `p`: MUST be `"brc-20"`
- `op`: MUST be `"deploy"`
- `tick`: The reward token ticker (e.g., `"REWARD"`)
- `max`: Maximum supply to be emitted over the program duration (string representation of integer)
- `curve`: Object containing:
  - `type`: Emission schedule (`"linear"` or `"exponential"`)
  - `lock`: Total duration of the program in Bitcoin blocks (string representation of integer)
  - `stakes`: Array of tickers accepted for staking
    - *Note:* To maintain mathematical trustlessness without price oracles, V1 of this standard recommends only **one** ticker per curve.

#### Staking Payload (`swap init`)

```json
{
  "p": "brc-20",
  "op": "swap",
  "init": "WTF,REWARD",
  "amt": "1000",
  "wrap": true
}
```

**Required Fields:**
- `p`: MUST be `"brc-20"`
- `op`: MUST be `"swap"`
- `init`: Pair of tokens in format `"STAKING_TOKEN,REWARD_TOKEN"` (e.g., `"WTF,REWARD"`)
- `amt`: Amount of staking token to stake (string representation of integer)
- `wrap`: MUST be `true` to signal this is a Curve staking operation (not a liquidity pool initialization)

**Effect:**
1. User's staking token balance (e.g., `WTF`) is decremented (locked).
2. User receives yToken (e.g., `yWTF`) 1:1 with the staked amount.

#### Claiming Payload (`swap exe`)

```json
{
  "p": "brc-20",
  "op": "swap",
  "exe": "yWTF,REWARD",
  "amt": "1000"
}
```

**Required Fields:**
- `p`: MUST be `"brc-20"`
- `op`: MUST be `"swap"`
- `exe`: Pair of tokens in format `"yTOKEN,REWARD_TOKEN"` where the first token is a yToken (e.g., `"yWTF,REWARD"`)
- `amt`: Amount of yToken to burn (string representation of integer)

**Effect:**
1. yToken (e.g., `yWTF`) is burned (removed from supply).
2. Original staking token principal (e.g., `WTF`) is unlocked and returned to the user.
3. `REWARD` tokens are minted **ex-nihilo** and credited to the user based on the accrued amount calculated from the emission mechanics.

### Emission Mechanics

Rewards are not stored; they are calculated. The emission mechanism operates on a per-block basis, where each block $b$ emits a quantity $E(b)$ of reward tokens according to the curve type defined at deployment.

#### Curve Types (`E(b)`)

Let $E(b)$ be the emission at block number $b$, where $b$ is relative to the deployment block $b_{deploy}$. The emission function depends on the curve `type` specified in the `deploy` transaction.

**Linear Curve**

The linear curve provides constant emission per block throughout the entire program duration.

$$E(b) = \frac{MAX\_SUPPLY}{LOCK\_DURATION}$$

**Properties:**
- Constant emission rate: $E(b)$ is identical for all blocks $b \in [b_{deploy}, b_{deploy} + LOCK\_DURATION]$
- Predictable and fair distribution over time
- Suitable for long-term incentive programs

**Example:**
- `max = 21,000,000 × 10⁸ = 2,100,000,000,000,000 sats`
- `lock = 1,051,200 blocks` (≈ 2 years at 10 min/block)
- $E(b) = \frac{2,100,000,000,000,000}{1,051,200} \approx 1,997,997,256$ sats per block

**Exponential Curve (4-Phase Decay)**

The exponential curve implements a front-loaded emission schedule designed to accelerate early adoption. Let $P = \frac{b - b_{deploy}}{LOCK\_DURATION}$ be the progress ratio (0 to 1).

The emission is divided into 4 phases with decreasing rates:

| Phase | Progress Range | % of Total Supply | Duration (blocks) | Emission Formula |
|-------|----------------|-------------------|-------------------|------------------|
| 1 | $0 \le P < 0.1$ | 50% | $0.1 \times LOCK$ | $E(b) = \frac{0.50 \times MAX\_SUPPLY}{0.10 \times LOCK\_DURATION}$ |
| 2 | $0.1 \le P < 0.3$ | 25% | $0.2 \times LOCK$ | $E(b) = \frac{0.25 \times MAX\_SUPPLY}{0.20 \times LOCK\_DURATION}$ |
| 3 | $0.3 \le P < 0.6$ | 12.5% | $0.3 \times LOCK$ | $E(b) = \frac{0.125 \times MAX\_SUPPLY}{0.30 \times LOCK\_DURATION}$ |
| 4 | $0.6 \le P \le 1.0$ | 12.5% | $0.4 \times LOCK$ | $E(b) = \frac{0.125 \times MAX\_SUPPLY}{0.40 \times LOCK\_DURATION}$ |

**Mathematical Verification:**
The total emission across all phases equals $MAX\_SUPPLY$:
$$0.50 \times MAX + 0.25 \times MAX + 0.125 \times MAX + 0.125 \times MAX = MAX\_SUPPLY$$

**Example Calculation:**
- `max = 2,100,000,000,000,000 sats`
- `lock = 1,051,200 blocks`
- **Phase 1** (blocks 0 to 105,120): $E(b) = \frac{0.50 \times 2.1 \times 10^{15}}{105,120} \approx 9,986,280,000$ sats/block
- **Phase 2** (blocks 105,120 to 315,360): $E(b) = \frac{0.25 \times 2.1 \times 10^{15}}{210,240} \approx 2,496,570,000$ sats/block
- **Phase 3** (blocks 315,360 to 630,720): $E(b) = \frac{0.125 \times 2.1 \times 10^{15}}{315,360} \approx 416,095,000$ sats/block
- **Phase 4** (blocks 630,720 to 1,051,200): $E(b) = \frac{0.125 \times 2.1 \times 10^{15}}{420,480} \approx 208,047,500$ sats/block

**Key Insight:** Early stakers in Phase 1 receive approximately **48x** more rewards per block than late stakers in Phase 4, creating strong incentives for immediate participation.

**Emission Termination:**
After block $b_{deploy} + LOCK\_DURATION$, $E(b) = 0$ forever. The total emitted equals exactly $MAX\_SUPPLY$ (no more, no less).

#### Pro-Rata Allocation & The Void Rule

At any block $b$, the rewards $E(b)$ are allocated to stakers proportionally based on their share of the total staked pool. The allocation uses the state at the **end of the previous block** ($b-1$) to prevent flash-loan attacks.

**Allocation Formula**

For a user $u$ at block $b$:

$$UserReward_u(b) = E(b) \times \frac{S_u(b-1)}{S_{total}(b-1)}$$

Where:
- $S_u(b-1)$ = amount staked by user $u$ at the end of block $b-1$
- $S_{total}(b-1)$ = total amount staked by all users at the end of block $b-1$
- $E(b)$ = emission for block $b$ (from curve formula)

**Why Use Previous Block State ($b-1$)?**

Using the state at block $b-1$ prevents **flash-stake attacks**:

1. **Attack Scenario (if using current block):** An attacker could deposit 99% of the pool in the same block as emission, capture almost all rewards, then withdraw immediately—all within a single block.

2. **Mitigation (using $b-1$):** The attacker must commit capital **before** block $b$ begins to benefit from its emission. This requires:
   - Paying transaction fees
   - Waiting for block confirmation
   - Maintaining the position across block boundaries

This makes flash attacks economically unprofitable due to transaction costs and time requirements.

**The Void Rule**

**Definition:** If $S_{total}(b-1) = 0$ (no stakers at the end of block $b-1$), then $E(b)$ is **not emitted**. These tokens are effectively lost/burned, preserving the scarcity schedule.

**Rationale:**
- Prevents emission to non-existent stakers
- Maintains total supply cap: if tokens are "voided" due to no stakers, they cannot be claimed later
- Creates economic incentive for continuous staking participation
- Ensures that emission only occurs when there is actual capital at risk

**Mathematical Expression:**
$$UserReward_u(b) = \begin{cases}
E(b) \times \frac{S_u(b-1)}{S_{total}(b-1)} & \text{if } S_{total}(b-1) > 0 \\
0 & \text{if } S_{total}(b-1) = 0
\end{cases}$$

**Total Rewards Calculation**

When a user claims (executes `swap exe`), the indexer calculates the total rewards accumulated over their entire staking period:

$$D_u = \sum_{b = b_{stake}}^{b_{claim}} UserReward_u(b)$$

Where:
- $b_{stake}$ = block when user initiated staking
- $b_{claim}$ = block when user executes claim
- The sum includes all blocks where the user's position was active

**Example (Linear Curve, 3 blocks):**
- Program: `max = 3,000 sats`, `lock = 3 blocks` → $E(b) = 1,000$ sats/block
- User stakes 500 tokens at block 1 when total pool = 500 tokens
- At block 2, another user adds 500 tokens → total pool = 1,000 tokens
- User claims at block 3

| Block $b$ | $E(b)$ | $S_{total}(b-1)$ | $S_u(b-1)$ | $UserReward_u(b)$ |
|-----------|--------|------------------|------------|-------------------|
| 1 | 1,000 | 0 (start) | 500 | $1,000 \times \frac{500}{500} = 1,000$ |
| 2 | 1,000 | 500 | 500 | $1,000 \times \frac{500}{1,000} = 500$ |
| 3 | 1,000 | 1,000 | 500 | $1,000 \times \frac{500}{1,000} = 500$ |

**Total rewards:** $D_u = 1,000 + 500 + 500 = 2,000$ sats minted ex-nihilo at claim.

### Implementation Guidelines (The "MasterChef" Algorithm)

To avoid iterating through every block history during a claim (which is computationally expensive, $O(n)$ where $n$ is the number of blocks), indexers SHOULD implement the **Cumulative Reward Per Share** algorithm, which achieves $O(1)$ complexity per operation.

#### Algorithm Overview

The MasterChef algorithm maintains a global accumulator (`acc_reward_per_share`) that tracks the cumulative rewards per staked token. This eliminates the need to iterate through historical blocks when calculating user rewards.

**Key Insight:** Instead of storing per-block rewards for each user, we maintain a single global accumulator that grows with each block's emission, and track each user's "debt" (rewards already accounted for).

#### Global State Variables

**`acc_reward_per_share`** (Global Pool State)
- Type: Integer (scaled by `PRECISION`)
- Purpose: Running total of rewards per staked token, accumulated across all blocks
- Update Formula:
  $$acc\_reward\_per\_share += \frac{E(current\_block) \times PRECISION}{S_{total}(current\_block - 1)}$$

**Important Notes:**
- `PRECISION` should be at least $10^{12}$ (1e12) to prevent rounding errors in integer arithmetic
- The accumulator is updated lazily: only when a user interacts (stake/unstake/claim) or at regular intervals
- If $S_{total} = 0$, the accumulator is not updated (Void Rule)

#### User State Variables

**`reward_debt`** (Per-User State)
- Type: Integer (scaled by `PRECISION`)
- Purpose: Bookkeeping value representing rewards already "accounted for" when the user last interacted
- Initial Value: 0 (for new stakers)

**Interpretation:** `reward_debt` represents the user's share of the accumulated rewards at the time of their last stake/unstake operation. It is subtracted from the user's current entitlement to prevent double-counting.

#### Operation Flow

**On Pool Update (Lazy Evaluation)**

Before processing any user operation, update the global accumulator for all blocks since the last update:

```python
# Pseudocode
def update_pool(current_block, last_update_block):
    for b in range(last_update_block + 1, current_block + 1):
        if total_stake(b - 1) > 0:  # Void Rule check
            emission = get_emission(b)
            acc_reward_per_share += (emission * PRECISION) // total_stake(b - 1)
    last_update_block = current_block
```

**On Stake Operation (`swap init`)**

When a user stakes tokens:

1. **Update Pool:** Call `update_pool()` to bring `acc_reward_per_share` up to date.

2. **Calculate Pending Rewards (if user already has a position):**
   $$Pending = \frac{UserStake \times acc\_reward\_per\_share}{PRECISION} - reward\_debt$$
   
   If $Pending > 0$, mint these rewards to the user.

3. **Update User Stake:** Increment `UserStake` by the new staked amount.

4. **Update Debt:**
   $$reward\_debt = \frac{NewUserStake \times acc\_reward\_per\_share}{PRECISION}$$

**On Unstake Operation (Partial Withdrawal)**

When a user unstakes (withdraws part of their position):

1. **Update Pool:** Call `update_pool()`.

2. **Calculate Pending:**
   $$Pending = \frac{UserStake \times acc\_reward\_per\_share}{PRECISION} - reward\_debt$$
   
   Mint `Pending` rewards to user.

3. **Update User Stake:** Decrement `UserStake` by the unstaked amount.

4. **Update Debt:**
   $$reward\_debt = \frac{NewUserStake \times acc\_reward\_per\_share}{PRECISION}$$

**On Claim Operation (`swap exe`)**

When a user claims (burns yToken to retrieve principal and rewards):

1. **Update Pool:** Call `update_pool()`.

2. **Calculate Pending Rewards:**
   $$Pending = \frac{UserStake \times acc\_reward\_per\_share}{PRECISION} - reward\_debt$$

3. **Distribute Rewards:** Mint `Pending` amount of `REWARD` tokens to the user.

4. **Return Principal:** Unlock and return the original staked tokens (e.g., `WTF`).

5. **Reset State:** Set `UserStake = 0` and `reward_debt = 0` (position closed).

#### Mathematical Proof of Correctness

**Theorem:** The MasterChef algorithm correctly calculates the same rewards as the naive $O(n)$ block-iteration method.

**Proof Sketch:**

For a user staking from block $b_1$ to $b_2$:

**Naive Method:**
$$D_u = \sum_{b = b_1}^{b_2} E(b) \times \frac{S_u(b-1)}{S_{total}(b-1)}$$

**MasterChef Method:**
At claim (block $b_2$):
- `acc_reward_per_share` contains: $\sum_{b = b_{deploy}}^{b_2} \frac{E(b) \times PRECISION}{S_{total}(b-1)}$
- User's entitlement: $\frac{S_u \times acc\_reward\_per\_share}{PRECISION}$
- User's debt: $\frac{S_u \times acc\_reward\_per\_share_{at\_b1}}{PRECISION}$
- Pending: $\frac{S_u \times (acc\_reward\_per\_share_{b2} - acc\_reward\_per\_share_{b1})}{PRECISION}$

Expanding the difference:
$$\frac{S_u \times PRECISION}{PRECISION} \times \sum_{b = b_1}^{b_2} \frac{E(b)}{S_{total}(b-1)} = S_u \times \sum_{b = b_1}^{b_2} \frac{E(b)}{S_{total}(b-1)}$$

If $S_u$ is constant (no intermediate stake changes), this equals the naive calculation. If $S_u$ changes, the debt mechanism correctly accounts for the user's varying stake over time.

#### Implementation Considerations

**Precision Requirements:**
- Use integer arithmetic with `PRECISION ≥ 1e12`
- Prevents rounding errors when dealing with large supplies and long durations
- Example: For `max = 2.1e15 sats` over `1,051,200 blocks`, precision of 1e12 ensures sub-satoshi accuracy

**Gas/Optimization:**
- Update pool lazily (only on user interactions or periodic updates)
- Cache `last_update_block` to avoid redundant calculations
- Batch updates for multiple users in the same block

**Edge Cases:**
- Handle division by zero when $S_{total} = 0$ (Void Rule)
- Handle integer overflow for very large `acc_reward_per_share` values
- Ensure `reward_debt` never exceeds user's entitlement (sanity check)

### The yToken Standard

The `yToken` (e.g., `yWTF`, `yBTC`) is a technical representation of the staked position.

- **Fungibility:** All `yWTF` tokens are identical.
- **Transferability:** `yWTF` can be transferred like any BRC-20. The new holder inherits the claim to the principal and *all* accrued rewards attached to that principal (calculated via `reward_debt` transfer logic).
- **Liquidity Note:** While transferable, liquidity is not guaranteed by the protocol. A secondary market (e.g., a Swap Pool `yWTF/WTF`) must be formed by market participants for instant exits. Without a pool, the yToken is simply a receipt held until maturity.

---

## Indexer Validation Rules

Indexers MUST validate the following before processing a state change:

1. **On `deploy+curve`:**
   - Validate JSON schema: `p` MUST be `"brc-20"`, `op` MUST be `"deploy"`, `curve` object MUST be present with required fields (`type`, `lock`, `stakes`)
   - Validate `type` is either `"linear"` or `"exponential"`
   - Validate `lock` is a positive integer string
   - Validate `stakes` is a non-empty array of ticker strings
   - Resolve and store the Genesis Address from Input[0]
   - Extract and store Genesis Fee amounts from Output[1] (staking fee) and Output[2] (claiming fee)
   - Mint nothing at deployment

2. **On `swap init` (staking):**
   - Validate JSON schema: `p` MUST be `"brc-20"`, `op` MUST be `"swap"`, `init` MUST be present, `wrap` MUST be `true`
   - Verify the transaction contains an output to the Genesis Address with value exactly matching the stored staking fee from `deploy.Output[1]`
   - If fee validation fails → **INVALID**, reject the transaction
   - Verify the staking token is in the `stakes` whitelist from the deploy
   - Process staking: lock user's tokens, mint yToken 1:1

3. **On `swap exe` (claiming):**
   - Validate JSON schema: `p` MUST be `"brc-20"`, `op` MUST be `"swap"`, `exe` MUST be present
   - Verify the input token is a yToken (starts with `"y"` prefix)
   - Verify the transaction contains an output to the Genesis Address with value exactly matching the stored claiming fee from `deploy.Output[2]`
   - If fee validation fails → **INVALID**, reject the transaction
   - Calculate pending rewards using emission mechanics (see Specification)
   - Verify global total minted ≤ `max` supply (if exceeded, prorate on remainder)
   - Process claim: burn yToken, unlock principal, mint rewards ex-nihilo

4. **Constitution Integrity:**
   - Ensure `max` supply is never exceeded globally across all claims
   - Emission stops strictly at `b_{deploy} + lock` blocks

5. **Anti-Flash Protection:**
   - Ensure rewards are calculated based on the state at the *start* of the block (or end of previous block `b-1`), preventing same-block deposit/claim exploits

---

## Rationale

**Ex-Nihilo Minting:** By minting rewards only at the moment of claim (exit), we remove the attack vector of a "Honey Pot" treasury. No tokens exist until they are earned, eliminating risks of theft from pre-mined supplies, insider trading with pre-allocated tokens, and centralized control over token distribution.

**Genesis Address:** Using the first input of deployment as a permanent anchor allows for a fee destination that cannot be changed, rotated, or hacked via admin keys. It is a "dead" address in terms of protocol power (cannot mint, burn, pause, or alter anything), but "live" for revenue. This design provides sustainable funding for protocol creators without compromising decentralization.

**Immutable Economics:** Once deployed, the emission curve (`type`, `lock`, `max`, `stakes`) cannot be changed. Post-deployment modifications are mathematically impossible—no admin keys or governance votes can alter emission schedules. This eliminates rug pulls, malicious modifications, and centralized control points.

**Context-Aware Swap Operations:** Reusing the existing `swap` OPI minimizes developer friction and enables immediate support from existing wallets and AMMs. The `wrap: true` parameter signals contextual interpretation, allowing the same infrastructure to handle both liquidity pools and staking programs without fragmentation.

**MasterChef Algorithm:** The Cumulative Reward Per Share algorithm achieves $O(1)$ complexity per operation instead of $O(n)$ block iteration, making the system scalable for long-duration programs while maintaining mathematical correctness.

**Previous Block State (`b-1`):** Using the state at the end of the previous block prevents flash-stake attacks within the same block. Attackers must commit capital before a block begins to benefit from its emission, making attacks unprofitable due to transaction costs and confirmation requirements.

**The Void Rule:** If no stakers exist at block $b-1$, the emission for block $b$ is voided (not emitted). This preserves the scarcity schedule, creates economic incentives for continuous participation, and ensures emission only occurs when there is actual capital at risk.

---

## Backwards Compatibility

This OPI is **fully backwards compatible**. Non-compliant indexers that do not recognize the `curve` extension will simply ignore the `curve` object in `deploy` transactions and process them as standard BRC-20 deploys. Similarly, `swap` operations with `wrap: true` will be ignored by non-compliant indexers, treating them as invalid or unrecognized operations. This ensures no disruption to existing infrastructure while allowing compliant indexers to implement the new functionality.

The contextual interpretation of `swap` operations (staking vs. liquidity pool) is determined by the presence of `wrap: true` and the existence of a corresponding `curve` program. Non-compliant indexers will not process these contextual swaps, but this does not break existing `swap` functionality for standard AMM operations.

---

## Reference Implementation

- **Repo**: https://github.com/universal-brc20/simplicity
- **Branch**: opi-2-curve-final
- **Examples**: The reference implementation includes:
  - Complete indexer module for processing `deploy+curve` transactions
  - Emission calculation logic for both linear and exponential curves
  - MasterChef algorithm implementation with lazy evaluation
  - Validation rules for Genesis Fee enforcement
  - Test cases with sample transactions and expected state transitions

---

## Security Considerations

### On-Chain Verifiability

- **Genesis Fees:** All fees are verifiable on-chain through the `deploy` transaction outputs. No trust in indexers is required for fee validation—anyone can verify that Output[1] and Output[2] define the immutable fee constants.

- **Emission Rules:** All emission parameters (`max`, `lock`, `type`, `stakes`) are immutably encoded in the `deploy` transaction. The emission schedule is deterministic and can be verified by anyone.

- **State Tracking:** All staking and claiming operations are recorded on-chain, enabling full auditability of the program's state and reward distribution.

### Attack Mitigations

**No Pre-mine:** All tokens are minted ex-nihilo based on immutable rules. This eliminates risks of:
- Theft from pre-mined supplies
- Insider trading with pre-allocated tokens
- Centralized control over token distribution

**Flash Attack Prevention:** Using end-of-previous-block state ($b-1$) prevents:
- Flash-stake attacks within the same block
- Same-block reward manipulation
- Economic exploitation through timing attacks

Attackers must commit capital **before** a block begins to benefit from its emission, making attacks unprofitable due to transaction costs and confirmation requirements.

**Immutable Rules:** Post-deployment modifications are mathematically impossible:
- No admin keys or governance votes can alter emission schedules
- No rug pulls or malicious modifications
- No centralized control points

**Genesis Fee Enforcement:** The mandatory Genesis Fees on every `init` and `exe` transaction:
- Mitigate spam attacks by requiring a fixed cost per operation
- Provide sustainable funding for creators without inflationary dilution
- Are fixed and unchangeable post-deployment (defined in `deploy` outputs)

### yToken Security

- **Trustless Accrual:** yToken value accrual is calculated deterministically from on-chain state. No external oracles or trusted parties required.

- **Transfer Safety:** yTokens are standard BRC-20 tokens with no special privileges or backdoors. When transferred, the new holder inherits all accrued rewards via the `reward_debt` mechanism.

- **Burn Verification:** yToken burns during `exe` are verifiable on-chain, ensuring proper principal return and reward distribution.

### Economic Security

- **Finite Supply:** Total emission is capped at `max`, preventing infinite inflation.

- **Time-Limited:** Programs have fixed duration (`lock`), ensuring predictable and bounded emissions.

- **Proportional Distribution:** Rewards are distributed proportionally based on stake share, preventing whale manipulation of individual blocks.

- **The Void Rule:** Tokens not emitted due to no stakers are permanently lost, preserving scarcity and incentivizing continuous participation.

### Implementation Security

- **Indexer Validation:** All indexers must validate:
  - Genesis Fee amounts match `deploy` outputs
  - Genesis Address receives fees correctly
  - Emission calculations follow canonical formulas
  - State transitions are consistent
  - Supply cap is never exceeded

- **Precision Requirements:** The MasterChef algorithm uses `PRECISION ≥ 1e12` to prevent rounding errors that could lead to incorrect reward calculations or supply inflation.

- **Edge Case Handling:** Indexers must properly handle:
  - Division by zero when $S_{total} = 0$ (Void Rule)
  - Integer overflow for very large `acc_reward_per_share` values
  - Sanity checks ensuring `reward_debt` never exceeds user's entitlement

- **Backwards Compatibility:** Non-compliant indexers simply ignore contextual swaps, ensuring no disruption to existing infrastructure while maintaining security for compliant implementations.
