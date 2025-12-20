OPI: 2

Title: The "Curve" Extension for Trustless Programmatic Token Distribution

Author: M3rl1n, GaloisField

Discussions-To: https://t.me/universalbrc20

Status: Active

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

**Implementation Note:** The code validates that Output[1] and Output[2] MUST go to the Genesis Address. If not, the transaction is REJECTED before creating any Deploy or CurveConstitution records. The indexer locates the OP_RETURN output first (which may not be at Output[0]), then validates that `Output[op_return_index + 1]` and `Output[op_return_index + 2]` contain the Genesis Fees with correct amounts and addresses.

#### Staking Transaction (`swap init`)

| Component | Description |
| --------- | ------------------------------------------- |
| Input[0+] | User's UTXOs containing the token to stake (e.g., `WTF`) |
| Output[0] | OP_RETURN output with the `swap init` payload |
| Output[1] | **Mandatory Genesis Fee** to Genesis Address. Value MUST equal the exact amount defined in `deploy.Output[1]`. If not, the transaction is **INVALID**. |
| Output[2+] | Change to user or other outputs |

**Implementation Note:** The code locates the OP_RETURN output first, then validates that `Output[op_return_index + 1]` contains the Genesis Fee with correct amount and address.

#### Claiming Transaction (`swap exe`)

| Component | Description |
| --------- | ------------------------------------------- |
| Input[0+] | User's UTXOs containing the yToken (e.g., `yWTF`) |
| Output[0] | OP_RETURN output with the `swap exe` payload |
| Output[1] | **Mandatory Genesis Fee** to Genesis Address. Value MUST equal the exact amount defined in `deploy.Output[2]`. If not, the transaction is **INVALID**. |
| Output[2+] | Change to user or other outputs |

**Implementation Note:** The code validates Genesis Fee only if `genesis_fee_exe_sats > 0`. If zero, validation is skipped. The code locates the OP_RETURN output first, then validates that `Output[op_return_index + 1]` contains the Genesis Fee with correct amount and address.

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
    - **Implementation:** Only the **first** ticker in the array (`stakes[0]`) is used. Additional tickers are ignored.

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
2. User receives yToken (e.g., `yWTF`) 1:1 with the staked amount (nominal).

**Implementation Note:** The yToken ticker is created as `f"y{staking_ticker}"` with lowercase 'y' prefix to prevent deploy conflicts. The yToken balance grows automatically as the `liquidity_index` increases, but the nominal amount minted equals the staked amount.

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
2. Original staking token principal (e.g., `WTF`) is unlocked and returned to the user (calculated using Nash Correction for proportional principal return).
3. `REWARD` tokens are minted **ex-nihilo** and credited to the user based on the accrued amount calculated from the rebasing index mechanics.

**Implementation Note:** The `slip` field is NOT required for Curve claims. The code routes Curve claims to a dedicated handler when the source ticker starts with `'y'` (case-insensitive).

### Emission Mechanics

Rewards are not stored; they are calculated. The emission mechanism operates on a per-block basis, where each block $b$ emits a quantity $E(b)$ of reward tokens according to the curve type defined at deployment.

#### Curve Types (`E(b)`)

Let $E(b)$ be the emission at block number $b$, where $b$ is relative to the deployment block $b_{deploy}$. The emission function depends on the curve `type` specified in the `deploy` transaction.

**Linear Curve**

The linear curve provides constant emission per block throughout the entire program duration.

$$E(b) = \frac{MAX\\_SUPPLY}{LOCK\\_DURATION}$$

**Properties:**
- Constant emission rate: $E(b)$ is identical for all blocks $b \in [b_{deploy}, b_{deploy} + LOCK\\_DURATION]$
- Predictable and fair distribution over time
- Suitable for long-term incentive programs

**Example:**
- `max = 21,000,000 × 10⁸ = 2,100,000,000,000,000 sats`
- `lock = 1,051,200 blocks` (≈ 2 years at 10 min/block)
- $E(b) = \frac{2,100,000,000,000,000}{1,051,200} \approx 1,997,997,256$ sats per block

**Exponential Curve (4-Phase Decay)**

The exponential curve implements a front-loaded emission schedule designed to accelerate early adoption. Let $P = \frac{b - b_{deploy}}{LOCK\\_DURATION}$ be the progress ratio (0 to 1).

The emission is divided into 4 phases with decreasing rates:

| Phase | Progress Range | % of Total Supply | Duration (blocks) | Emission Formula |
|-------|----------------|-------------------|-------------------|------------------|
| 1 | $0 \le P < 0.1$ | 50% | $0.1 \times LOCK$ | $E(b) = \frac{0.50 \times MAX\\_SUPPLY}{0.10 \times LOCK\\_DURATION}$ |
| 2 | $0.1 \le P < 0.3$ | 25% | $0.2 \times LOCK$ | $E(b) = \frac{0.25 \times MAX\\_SUPPLY}{0.20 \times LOCK\\_DURATION}$ |
| 3 | $0.3 \le P < 0.6$ | 12.5% | $0.3 \times LOCK$ | $E(b) = \frac{0.125 \times MAX\\_SUPPLY}{0.30 \times LOCK\\_DURATION}$ |
| 4 | $0.6 \le P \le 1.0$ | 12.5% | $0.4 \times LOCK$ | $E(b) = \frac{0.125 \times MAX\\_SUPPLY}{0.40 \times LOCK\\_DURATION}$ |

**Implementation Note:** The reference implementation calculates exponential phases using multipliers based on the linear rate:
- Phase 1: $E(b) = linear\\_rate \times 5$
- Phase 2: $E(b) = linear\\_rate \times 1.25$
- Phase 3: $E(b) = linear\\_rate \times 0.416666666666666667$
- Phase 4: $E(b) = linear\\_rate \times 0.3125$

Where $linear\\_rate = \frac{MAX\\_SUPPLY}{LOCK\\_DURATION}$. These multipliers are mathematically equivalent to the explicit phase formulas above.

**Mathematical Verification:**
The total emission across all phases equals $MAX\\_SUPPLY$:
$$0.50 \times MAX\\_SUPPLY + 0.25 \times MAX\\_SUPPLY + 0.125 \times MAX\\_SUPPLY + 0.125 \times MAX\\_SUPPLY = MAX\\_SUPPLY$$

**Example Calculation:**
- `max = 2,100,000,000,000,000 sats`
- `lock = 1,051,200 blocks`
- **Phase 1** (blocks 0 to 105,120): $E(b) = \frac{0.50 \times 2.1 \times 10^{15}}{105,120} \approx 9,986,280,000$ sats/block
- **Phase 2** (blocks 105,120 to 315,360): $E(b) = \frac{0.25 \times 2.1 \times 10^{15}}{210,240} \approx 2,496,570,000$ sats/block
- **Phase 3** (blocks 315,360 to 630,720): $E(b) = \frac{0.125 \times 2.1 \times 10^{15}}{315,360} \approx 416,095,000$ sats/block
- **Phase 4** (blocks 630,720 to 1,051,200): $E(b) = \frac{0.125 \times 2.1 \times 10^{15}}{420,480} \approx 208,047,500$ sats/block

**Key Insight:** Early stakers in Phase 1 receive approximately **48x** more rewards per block than late stakers in Phase 4, creating strong incentives for immediate participation.

**Emission Termination:**
After block $b_{deploy} + LOCK\\_DURATION$, $E(b) = 0$ forever. The total emitted equals exactly $MAX\\_SUPPLY$ (no more, no less).

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

### Data Structures

#### CurveConstitution

Stores immutable Curve program parameters and rebasing index state.

**Fields:**
- `ticker` (String, PK): Reward token ticker (FK to Deploy.ticker)
- `deploy_txid` (String, Unique): Transaction ID of the deploy operation
- `curve_type` (String): `"linear"` or `"exponential"`
- `lock_duration` (Integer): Lock duration in blocks
- `staking_ticker` (String): Staking token ticker (e.g., `"WTF"`)
- `max_supply` (Numeric(38,8)): Maximum supply (duplicated from Deploy)
- `max_stake_supply` (Numeric(38,8), Nullable): Max supply of staking ticker (for rho_g calculation)
- `rho_g` (Numeric(38,8), Nullable): Genesis Density Ratio (M_C / M_W)
- `genesis_fee_init_sats` (BigInteger): Genesis fee for init operation (in sats)
- `genesis_fee_exe_sats` (BigInteger): Genesis fee for exe operation (in sats)
- `genesis_address` (String): Genesis address (same as deployer_address)
- `start_block` (Integer): Block height when Curve program started
- `last_reward_block` (Integer): Last block height where rewards were calculated
- `liquidity_index` (Numeric(78,27)): Liquidity Index (RAY precision, init = 1e27)
- `total_staked` (Numeric(38,8)): Total staked amount
- `total_scaled_staked` (Numeric(78,27)): Total scaled balances (RAY precision)

**Implementation Note:** `max_stake_supply` and `rho_g` are calculated from the staking_ticker Deploy if available. If not available, `rho_g` defaults to `1` during claim operations.

#### CurveUserInfo

Stores user staking information and scaled balance.

**Fields:**
- `id` (Integer, PK, Auto): Primary key
- `ticker` (String, Indexed): Reward token ticker (FK to CurveConstitution.ticker)
- `user_address` (String, Indexed): User Bitcoin address
- `staked_amount` (Numeric(38,8)): Amount currently staked by user
- `scaled_balance` (Numeric(78,27)): Scaled balance (RAY precision)

**Unique Constraint:** (`ticker`, `user_address`)

### Implementation Guidelines (The Rebasing Index Model)

To avoid iterating through every block history during a claim (which is computationally expensive, $O(n)$ where $n$ is the number of blocks), indexers MUST implement the **Rebasing Index Model** with RAY precision, which achieves $O(1)$ complexity per operation.

#### Algorithm Overview

The rebasing index model maintains a global **liquidity index** that tracks cumulative growth of the staking pool via rebasing. This eliminates the need to iterate through historical blocks when calculating user rewards.

**Key Insight:** Instead of storing per-block rewards for each user, we maintain a single global liquidity index that grows with each block's emission, and track each user's **scaled balance** (their constant share of the pool). The user's real balance is calculated dynamically as `scaled_balance × liquidity_index / RAY`.

**Mathematical Equivalence:** This model is mathematically equivalent to the MasterChef algorithm but uses a different representation that simplifies transfer operations and eliminates the need for explicit `reward_debt` tracking.

#### Global State Variables

**`liquidity_index`** (Global Pool State)
- Type: `Numeric(precision=78, scale=27)` (RAY precision)
- Initial Value: `1e27` (represents 1.0)
- Purpose: Tracks cumulative growth of the staking pool via rebasing
- Update Formula:
  $$liquidity\\_index += \frac{growth\\_value \times RAY}{total\\_staked}$$
  
  Where:
  - $growth\\_value = \frac{emission}{rho_g}$ (yToken units added to pool)
  - $rho_g = \frac{max\\_supply}{max\\_stake\\_supply}$ (Genesis Density Ratio)
  - $RAY = 10^{27}$ (Aave Standard Precision)

**`total_staked`** (Global Pool State)
- Type: `Numeric(precision=38, scale=8)`
- Purpose: Total nominal amount staked by all users
- Updated on every stake/claim/transfer operation

**`total_scaled_staked`** (Global Pool State)
- Type: `Numeric(precision=78, scale=27)` (RAY precision)
- Purpose: Total scaled balances (for internal accounting and consistency checks)
- Updated on every stake/claim/transfer operation

**Important Notes:**
- `RAY = 10^{27}` (Aave Standard Precision) is used for all liquidity index and scaled balance calculations
- The liquidity index is updated lazily: only when a user interacts (stake/claim/transfer) or at regular intervals
- If $total\\_staked = 0$, the liquidity index is not updated (Void Rule)
- The indexer MUST process blocks sequentially and update `liquidity_index` before every operation

**Genesis Density Ratio (`rho_g`):**
- Purpose: Converts emission (reward tokens) to growth_value (yToken units)
- Formula: $rho_g = \frac{max\\_supply}{max\\_stake\\_supply}$
- Source: `max_stake_supply` is obtained from the `Deploy` record of the `staking_ticker`
- Fallback: If `max_stake_supply` is unavailable or zero, `rho_g` defaults to `1` during claim operations

#### User State Variables

**`scaled_balance`** (Per-User State)
- Type: `Numeric(precision=78, scale=27)` (RAY precision)
- Purpose: User's staked amount scaled by liquidity index at time of deposit
- Initial Value: `0`
- Update on Stake:
  $$scaled\\_balance += \frac{amount \times RAY}{liquidity\\_index}$$

**`staked_amount`** (Per-User State)
- Type: `Numeric(precision=38, scale=8)`
- Purpose: Nominal amount staked by user (for principal return calculation)
- Initial Value: `0`
- Updated on every stake/claim/transfer operation

**Interpretation:** `scaled_balance` represents the user's constant share of the pool. As `liquidity_index` increases with rewards, the user's real balance grows automatically without modifying `scaled_balance`. This eliminates the need for explicit `reward_debt` tracking.

#### Operation Flow

**On Pool Update (Lazy Evaluation)**

Before processing any user operation, update the global liquidity index for all blocks since the last update:

```python
# Pseudocode
def update_index(ticker, current_block):
    const = get_curve_constitution(ticker)
    
    if current_block <= const.last_reward_block:
        return const
    
    # Void Rule: If no stakers, emission is lost
    if const.total_staked == 0:
        const.last_reward_block = current_block
        return const
    
    # Calculate total emission for the period
    emission = get_emission_in_range(const, const.last_reward_block, current_block)
    
    # Calculate growth_value (yToken units)
    rho_g = const.rho_g  # M_C / M_W
    growth_value = emission / rho_g if rho_g > 0 else 0
    
    # Update liquidity_index
    index_increase_nominal = growth_value / const.total_staked
    index_increase = index_increase_nominal * RAY
    const.liquidity_index += index_increase
    const.last_reward_block = current_block
    
    return const
```

**On Stake Operation (`swap init`)**

When a user stakes tokens:

1. **Update Pool:** Call `update_index()` to bring `liquidity_index` up to date.

2. **Calculate Scaled Amount:**
   $$scaled\\_amount = \frac{amount \times RAY}{liquidity\\_index}$$

3. **Update User State:**
   - `user.scaled_balance += scaled_amount`
   - `user.staked_amount += amount`

4. **Update Global State:**
   - `const.total_scaled_staked += scaled_amount`
   - `const.total_staked += amount`

5. **Mint yToken:** User receives `amount` yTokens (1:1 nominal).

**On Claim Operation (`swap exe`)**

When a user claims (burns yToken to retrieve principal and rewards):

1. **Update Pool:** Call `update_index()`.

2. **Calculate Current Real Balance:**
   $$current\\_real\\_balance = \frac{user.scaled\\_balance \times liquidity\\_index}{RAY}$$

3. **Validate Burn Amount:** User MUST have sufficient yToken balance (with tolerance for rounding errors: `0.999999`).

4. **Calculate Scaled Burn:**
   $$scaled\\_burn = \frac{amount\\_ytoken\\_burn \times RAY}{liquidity\\_index}$$

5. **Calculate Principal Out (Nash Correction):**
   $$burn\\_ratio = \frac{amount\\_ytoken\\_burn}{current\\_real\\_balance}$$
   $$principal\\_out = user.staked\\_amount \times burn\\_ratio$$

7. **Calculate Yield:**
   $$yield\\_ytoken = amount\\_ytoken\\_burn - principal\\_out$$
   $$crv\\_out = yield\\_ytoken \times rho_g$$

8. **Update User State:**
   - `user.scaled_balance -= scaled_burn`
   - `user.staked_amount = max(0, user.staked_amount - principal_out)`

9. **Update Global State:**
   - `const.total_scaled_staked -= scaled_burn`
   - `const.total_staked = max(0, const.total_staked - principal_out)`

10. **Distribute:**
   - Burn yToken: Remove `amount_ytoken_burn` from user's balance
   - Return Principal: Credit `principal_out` staking tokens to user
   - Mint Rewards: Mint `crv_out` reward tokens to user ex-nihilo

**On Transfer Operation**

When yTokens are transferred:

1. **Update Pool:** Call `update_index()`.

2. **Calculate Current Real Balance (Sender):**
   $$current\\_real\\_balance\\_from = \frac{user\\_from.scaled\\_balance \times liquidity\\_index}{RAY}$$

3. **Calculate Scaled Transfer:**
   $$scaled\\_to\\_transfer = \frac{amount\\_ytoken \times RAY}{liquidity\\_index}$$

4. **Calculate Transfer Ratio:**
   $$transfer\\_ratio = \frac{scaled\\_to\\_transfer}{user\\_from.scaled\\_balance}$$

5. **Calculate Principal Transfer:**
   $$principal\\_transfer = user\\_from.staked\\_amount \times transfer\\_ratio$$

6. **Update Sender:**
   - `user_from.scaled_balance -= scaled_to_transfer`
   - `user_from.staked_amount -= principal_transfer`

7. **Update Receiver:**
   - `user_to.scaled_balance += scaled_to_transfer`
   - `user_to.staked_amount += principal_transfer`

**Implementation Note:** The new holder inherits all accrued rewards via the scaled balance mechanism. No explicit `reward_debt` transfer is needed. The transferred `scaled_balance` automatically includes all accumulated yield, as the `liquidity_index` is global and applies to all scaled balances equally.

### Constants

**RAY Precision:**
- `RAY = 10^{27}` (Aave Standard Precision)
- Used for all liquidity index and scaled balance calculations
- Prevents rounding errors in integer arithmetic
- Higher precision than the originally suggested `PRECISION ≥ 1e12`, ensuring sub-satoshi accuracy for large supplies and long durations

#### Mathematical Proof of Correctness

**Theorem:** The rebasing index model correctly calculates the same rewards as the naive $O(n)$ block-iteration method.

**Proof Sketch:**

For a user staking from block $b_1$ to $b_2$:

**Naive Method:**
$$D_u = \sum_{b = b_1}^{b_2} E(b) \times \frac{S_u(b-1)}{S_{total}(b-1)}$$

**Rebasing Index Method:**
At stake (block $b_1$):
- User deposits $S_u$ tokens

$$scaled\\_balance = \frac{S_u \times RAY}{liquidity\\_index_{b1}}$$

At claim (block $b_2$):
- `liquidity_index_{b2}` = `liquidity_index_{b1}`+ $\sum_{b = b_1+1}^{b_2} \frac{growth\\_value(b) \times RAY}{total\\_staked(b-1)}$

$$Real balance = \frac{scaled\\_balance \times liquidity\\_index_{b2}}{RAY} = S_u \times \frac{liquidity\\_index_{b2}}{liquidity\\_index_{b1}}$$
$$Yield = Real balance - S_u = S_u \times (\frac{liquidity\\_index_{b2}}{liquidity\\_index_{b1}} - 1)$$

Expanding the index ratio:

$$\frac{liquidity\\_index_{b2}}{liquidity\\_index_{b1}} = 1 + \sum_{b = b_1+1}^{b_2} \frac{growth\\_value(b)}{total\\_staked(b-1) \times liquidity\\_index_{b1}}$$

Since $growth\\\_value(b) = \frac{E(b)}{rho_g}$ and $liquidity\\\_index_{b1} \approx \frac{total\\_staked(b_1)}{total\\_staked(b_1)}$ (normalized), this simplifies to the naive calculation when $S_u$ is constant.

#### Implementation Considerations

**Precision Requirements:**
- Use `RAY = 10^{27}` (Aave Standard Precision) for all liquidity index and scaled balance calculations
- Prevents rounding errors when dealing with large supplies and long durations
- Example: For `max = 2.1e15 sats` over `1,051,200 blocks`, RAY precision ensures sub-satoshi accuracy

**Sequential Processing:**
- The rebasing model REQUIRES sequential block processing
- The indexer MUST call `update_index()` before every operation to ensure `liquidity_index` is up to date
- Between two operations, `total_staked` is constant by definition (sequential processing guarantee)

**Gas/Optimization:**
- Update index lazily (only on user interactions or periodic updates)
- Cache `last_reward_block` to avoid redundant calculations
- Batch updates for multiple users in the same block

**Edge Cases:**
- Handle division by zero when $total\\_staked = 0$ (Void Rule)
- Handle integer overflow for very large `liquidity_index` values
- Tolerance for rounding errors (0.999999) in balance validation during claim operations
- Ensure `scaled_burn` never exceeds `user.scaled_balance` (sanity check)

### The yToken Standard

The `yToken` (e.g., `yWTF`, `yBTC`) is a technical representation of the staked position.

- **Fungibility:** All `yWTF` tokens are identical.
- **Transferability:** `yWTF` can be transferred like any BRC-20. The new holder inherits the claim to the principal and *all* accrued rewards attached to that principal (calculated via `scaled_balance` transfer mechanism).
- **Dynamic Balance:** The yToken balance grows automatically as the `liquidity_index` increases. The real balance is calculated as `scaled_balance × liquidity_index / RAY`, ensuring rewards are always up-to-date without requiring per-user updates.
- **Liquidity Note:** While transferable, liquidity is not guaranteed by the protocol. A secondary market (e.g., a Swap Pool `yWTF/WTF`) must be formed by market participants for instant exits. Without a pool, the yToken is simply a receipt held until maturity.

---

## Indexer Validation Rules

Indexers MUST validate the following before processing a state change:

1. **On `deploy+curve`:**
   - Validate JSON schema: `p` MUST be `"brc-20"`, `op` MUST be `"deploy"`, `curve` object MUST be present with required fields (`type`, `lock`, `stakes`)
   - Validate `type` is either `"linear"` or `"exponential"`
   - Validate `lock` is a positive integer string
   - Validate `stakes` is a non-empty array of ticker strings
   - Validate ticker does NOT start with `'y'` (case-insensitive) - reserved for yTokens
   - Resolve and store the Genesis Address from Input[0]
   - Extract and store Genesis Fee amounts from Output[1] (staking fee) and Output[2] (claiming fee)
   - Validate that Output[1] and Output[2] go to the Genesis Address
   - Calculate and store `max_stake_supply` and `rho_g` (Genesis Density Ratio) if available
   - Initialize `liquidity_index = 1e27` (RAY precision)
   - Mint nothing at deployment

2. **On `swap init` (staking):**
   - Validate JSON schema: `p` MUST be `"brc-20"`, `op` MUST be `"swap"`, `init` MUST be present, `wrap` MUST be `true`
   - Verify the transaction contains an output to the Genesis Address with value exactly matching the stored staking fee from `deploy.Output[1]`
   - If fee validation fails → **INVALID**, reject the transaction
   - Verify the staking token is in the `stakes` whitelist from the deploy (only first ticker is checked)
   - Process staking: lock user's tokens, calculate and store `scaled_balance`, mint yToken 1:1 (nominal)

3. **On `swap exe` (claiming):**
   - Validate JSON schema: `p` MUST be `"brc-20"`, `op` MUST be `"swap"`, `exe` MUST be present
   - Verify the input token is a yToken (starts with `"y"` prefix, case-insensitive)
   - If `genesis_fee_exe_sats > 0`, verify the transaction contains an output to the Genesis Address with value exactly matching the stored claiming fee from `deploy.Output[2]`
   - If fee validation fails → **INVALID**, reject the transaction
   - Calculate pending rewards using rebasing index mechanics (see Specification)
   - Verify global total minted ≤ `max` supply (if exceeded, prorate on remainder)
   - Process claim: burn yToken, calculate principal return using Nash Correction, unlock principal, mint rewards ex-nihilo

4. **Constitution Integrity:**
   - Ensure `max` supply is never exceeded globally across all claims
   - Emission stops strictly at `b_{deploy} + lock` blocks

5. **Anti-Flash Protection:**
   - Ensure rewards are calculated based on the state at the *start* of the block (or end of previous block `b-1`), preventing same-block deposit/claim exploits
   - The rebasing model inherently uses sequential block processing—the indexer MUST process blocks sequentially and update `liquidity_index` before every operation

---

## Rationale

**Ex-Nihilo Minting:** By minting rewards only at the moment of claim (exit), we remove the attack vector of a "Honey Pot" treasury. No tokens exist until they are earned, eliminating risks of theft from pre-mined supplies, insider trading with pre-allocated tokens, and centralized control over token distribution.

**Genesis Address:** Using the first input of deployment as a permanent anchor allows for a fee destination that cannot be changed, rotated, or hacked via admin keys. It is a "dead" address in terms of protocol power (cannot mint, burn, pause, or alter anything), but "live" for revenue. This design provides sustainable funding for protocol creators without compromising decentralization.

**Immutable Economics:** Once deployed, the emission curve (`type`, `lock`, `max`, `stakes`) cannot be changed. Post-deployment modifications are mathematically impossible—no admin keys or governance votes can alter emission schedules. This eliminates rug pulls, malicious modifications, and centralized control points.

**Context-Aware Swap Operations:** Reusing the existing `swap` OPI minimizes developer friction and enables immediate support from existing wallets and AMMs. The `wrap: true` parameter signals contextual interpretation, allowing the same infrastructure to handle both liquidity pools and staking programs without fragmentation.

**Rebasing Index Model:** The rebasing index model with RAY precision achieves $O(1)$ complexity per operation instead of $O(n)$ block iteration, making the system scalable for long-duration programs while maintaining mathematical correctness. This model simplifies transfer operations by eliminating the need for explicit `reward_debt` tracking—new holders automatically inherit accrued rewards via the `scaled_balance` mechanism.

**Previous Block State (`b-1`):** Using the state at the end of the previous block prevents flash-stake attacks within the same block. Attackers must commit capital before a block begins to benefit from its emission, making attacks unprofitable due to transaction costs and confirmation requirements.

**The Void Rule:** If no stakers exist at block $b-1$, the emission for block $b$ is voided (not emitted). This preserves the scarcity schedule, creates economic incentives for continuous participation, and ensures emission only occurs when there is actual capital at risk.

---

## Backwards Compatibility

This OPI is **fully backwards compatible**. Non-compliant indexers that do not recognize the `curve` extension will simply ignore the `curve` object in `deploy` transactions and process them as standard BRC-20 deploys. Similarly, `swap` operations with `wrap: true` will be ignored by non-compliant indexers, treating them as invalid or unrecognized operations. This ensures no disruption to existing infrastructure while allowing compliant indexers to implement the new functionality.

The contextual interpretation of `swap` operations (staking vs. liquidity pool) is determined by the presence of `wrap: true` and the existence of a corresponding `curve` program. Non-compliant indexers will not process these contextual swaps, but this does not break existing `swap` functionality for standard AMM operations.

---

## Reference Implementation

- **Repo**: https://github.com/The-Universal-BRC-20-Extension/Simplicity
- **Examples**: The reference implementation includes:
  - Complete indexer module for processing `deploy+curve` transactions
  - Emission calculation logic for both linear and exponential curves
  - Rebasing index model implementation with RAY precision and lazy evaluation
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

- **Transfer Safety:** yTokens are standard BRC-20 tokens with no special privileges or backdoors. When transferred, the new holder inherits all accrued rewards via the `scaled_balance` mechanism—the transferred scaled balance automatically includes all accumulated yield.

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

- **Precision Requirements:** The rebasing index model uses `RAY = 10^{27}` (Aave Standard Precision) to prevent rounding errors that could lead to incorrect reward calculations or supply inflation.

- **Sequential Processing:** The rebasing model REQUIRES sequential block processing. Indexers MUST call `update_index()` before every operation to ensure `liquidity_index` is up to date.

- **Edge Case Handling:** Indexers must properly handle:
  - Division by zero when $total\\_staked = 0$ (Void Rule)
  - Integer overflow for very large `liquidity_index` values
  - Tolerance for rounding errors (0.999999) in balance validation during claim operations
  - Sanity checks ensuring `scaled_burn` never exceeds `user.scaled_balance`

- **Backwards Compatibility:** Non-compliant indexers simply ignore contextual swaps, ensuring no disruption to existing infrastructure while maintaining security for compliant implementations.
