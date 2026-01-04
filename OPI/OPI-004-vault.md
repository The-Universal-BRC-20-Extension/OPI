# OPI: 4

**Title**: The "Vault" Standard: Automated Strategy Pools & Elastic Shares

**Author**: Laz1m0v <wtf@wbtc.lol>

**Discussions-To**: https://t.me/universalbrc20

**Status**: Draft

**Type**: Standards Track

**Created**: 2026-01-01

**Requires**: OPI-1 (Swap), OPI-2 (Curve), OPI-3 (W)

---

## Abstract

This document specifies the **"Vault" Standard**, a protocol for creating **Automated Strategy Vaults** on Bitcoin Layer 1. These virtual pools aggregate user assets to execute complex yield-farming strategies with automatic compounding, eliminating the need for centralized keepers or vulnerable treasuries. The protocol introduces **virtual vTokens** (elastic supply shares representing user equity that appreciate mathematically as rewards compound), **public harvest** (an incentivized mechanism allowing any user to trigger the strategy and earn a bounty), and **implicit tickers** (vaults identified directly by their pair, simplifying discovery and composability). Through integration with OPI-1 (swaps), OPI-2 (curve rewards), and OPI-3 (W collateral), vaults enable trustless yield automation where shares appreciate without pre-mine, strategies execute without admin keys, and the protocol remains self-sustaining through community participation. This standard transforms Bitcoin into an immutable yield factory where mathematical security replaces trust.

---

## Copyright

This OPI is released under the MIT License.

---

## Motivation

In legacy DeFi (Ethereum/Solana), vaults like Yearn or Convex are centralized: they rely on off-chain keepers, vulnerable treasuries, and admin upgrades that introduce rug pull or hack risks. Universal BRC-20, with its Event Sourcing architecture on OP_RETURN and trustless indexers, deserves better—a system where yield strategies are engraved in the timechain, auto-executed by the community, and where shares (vTokens) appreciate mathematically without pre-mine.

OPI-4 solves the "Yield Paradox": users want to maximize rewards (via compounding) without constantly monitoring curves (OPI-2) or wraps (OPI-3). By automating harvest/re-invest, we unlock dormant capital, boost massive adoption, and align incentives: anyone can "harvest" for a bounty, making the protocol self-sustaining. Link with OPI-3: vaults use W to virtualize BTC as liquid collateral, closing the loop of true DeFi on L1. This is the next building block for our revolution—transforming Bitcoin into an immutable yield factory.

---

## Specification

### Core Concepts

**Automated Strategy Vault**: A virtual pool that aggregates user assets to execute yield-farming strategies automatically. Users deposit assets and receive vTokens representing their share, which appreciate as rewards are harvested and compounded.

**Virtual vToken**: An elastic supply token representing a user's equity in a vault. vTokens have no `max_supply`—they are minted on deposit and burned on redemption. Their value appreciates as the vault's locked liquidity grows from harvested rewards. **Important:** The vToken uses the same ticker as the vault ID (implicit ticker format `"BASE/REWARD"`). For example, a vault with ID `"LOL/WTF"` issues vTokens with ticker `"LOL/WTF"`—no separate token deployment is required.

**Net Asset Value (NAV)**: The price of a vToken, calculated as `Total Locked Liquidity / Total vToken Supply`. As harvest operations add liquidity without minting new shares, NAV increases over time, creating the auto-compound effect.

**Public Harvest**: A permissionless mechanism where any user can trigger the vault's compounding strategy. The caller receives a bounty (1% of harvested rewards), aligning community incentives with protocol health.

**Implicit Ticker**: A vault identifier using the format `"BASE/REWARD"` (e.g., `"LOL/WTF"`). The pair itself serves as the ticker, eliminating the need for custom token deployments and simplifying discovery.

---

### Transaction Structure

#### Deploy Transaction

The `vault deploy` transaction MUST adhere to the following structure:

| Component | Description |
| --------- | ------------------------------------------- |
| Input[0] | UTXO from the deployer. The address controlling this input becomes the immutable **Vault Creator Address**. |
| Output[0] | OP_RETURN output containing the JSON payload with the `vault` operation |
| Output[1] | **Genesis Fee Definition (Optional, requires `fee_check: true`).** The exact amount of sats (e.g., 5000) that users must pay to the Vault Creator Address when calling `init`. If `fee_check` is `true` and this output exists, this value becomes the immutable fee constant for all future `init` operations. If `fee_check` is absent or `false`, this output is ignored. |
| Output[2] | **Genesis Fee Definition (Optional, requires `fee_check: true`).** The exact amount of sats (e.g., 2000) that users must pay to the Vault Creator Address when calling `exe`. If `fee_check` is `true` and this output exists, this value becomes the immutable fee constant for all future `exe` operations. If `fee_check` is absent or `false`, this output is ignored. |
| Output[3+] | Change back to deployer or other optional outputs |

**Vault Creator Address:** The address controlling Input[0] is resolved and stored by indexers at deployment. It has **no protocol power**—it cannot mint, burn, pause, or alter anything. Its sole function is as a universal anchor for tracking, discoverability, and receiving Genesis Fees (if defined).

#### Deposit Transaction (`vault init`)

| Component | Description |
| --------- | ------------------------------------------- |
| Input[0+] | User's UTXOs containing the token to deposit (e.g., `WTF`) |
| Output[0] | OP_RETURN output with the `vault init` payload |
| Output[1] | **Mandatory Genesis Fee** to Vault Creator Address (if `genesis_fee_init_sats > 0`). Value MUST equal the exact amount defined in `deploy.Output[1]`. If not, the transaction is **INVALID**. |
| Output[2+] | Change to user or other outputs |

#### Harvest Transaction (`vault harvest`)

| Component | Description |
| --------- | ------------------------------------------- |
| Input[0+] | Caller's UTXOs (for transaction fees) |
| Output[0] | OP_RETURN output with the `vault harvest` payload |
| Output[1+] | Change to caller or other outputs |

**Note:** The harvest operation is executed by the indexer. The caller receives a bounty (1% of harvested rewards) credited to their address.

#### Redemption Transaction (`vault exe`)

| Component | Description |
| --------- | ------------------------------------------- |
| Input[0+] | User's UTXOs containing the vToken (e.g., `LOL/WTF`) |
| Output[0] | OP_RETURN output with the `vault exe` payload |
| Output[1] | **Mandatory Genesis Fee** to Vault Creator Address (if `genesis_fee_exe_sats > 0`). Value MUST equal the exact amount defined in `deploy.Output[2]`. If not, the transaction is **INVALID**. |
| Output[2+] | Change to user or other outputs |

### OP_RETURN Payload

#### Deploy Payload

Initializes the strategy logic for a specific pair. The deployer defines immutable parameters, with no admin power post-deploy.

```json
{
  "p": "brc-20",
  "op": "vault",
  "deploy": "LOL/WTF",
  "strategy": {
    "target_pool": "CRV/WTF",
    "fee": "200"
  },
  "lock": "52560",
  "fee_check": true,
  "min_harvest_interval": "144"
}
```

**Required Fields:**
- `p`: MUST be `"brc-20"`
- `op`: MUST be `"vault"`
- `deploy`: Canonical vault ID in format `"BASE/REWARD"` (e.g., `"LOL/WTF"`)
- `strategy`: Object containing:
  - `target_pool`: AMM pool route (OPI-1) for selling rewards (format `"TICKER1/TICKER2"`)
  - `fee`: Performance fee in basis points (bps, e.g., `"200"` = 2%)
- `lock`: Default lock duration in blocks (e.g., `"52560"` ≈ 1 year at 10 min/block)

**Optional Fields:**
- `fee_check`: Boolean flag (default: `false`). If `true`, Output[1] and Output[2] are interpreted as Genesis Fees. If absent or `false`, Genesis Fees are ignored (set to 0).
- `min_harvest_interval`: Minimum blocks between harvest operations (default: `"144"` ≈ 24 hours at 10 min/block). Prevents harvest spam and bounty exploitation.

**Effect:** Creates a virtual vault identified by `LOL/WTF`. Genesis Fees (OPI-2 style): fixed fees in sats to the deployer for each future interaction (if defined in transaction outputs).

**Validation Rules:**
- The vault ID MUST be in format `"TICKER1/TICKER2"` (exactly one slash)
- The vault ID MUST NOT start with `'y'` (case-insensitive) to avoid conflicts with yTokens (OPI-2)
- The `target_pool` MUST reference an existing OPI-1 swap pool
- The `fee` MUST be between `0` and `10000` (0% to 100%)
- **Namespace Reservation**: The "/" character is reserved globally in Universal BRC-20. Any `op: deploy` (non-vault) containing "/" in ticker MUST be rejected as INVALID. Only `op: vault` with `deploy: "BASE/REWARD"` can use "/".

#### Deposit Payload (`vault init`)

The "Zap-In": Users deposit raw assets to mint virtual shares.

```json
{
  "p": "brc-20",
  "op": "vault",
  "init": "WTF,LOL/WTF",
  "amt": "10000",
  "lock": "1000"
}
```

**Required Fields:**
- `p`: MUST be `"brc-20"`
- `op`: MUST be `"vault"`
- `init`: Pair in format `"INPUT_ASSET,VAULT_ID"` (e.g., `"WTF,LOL/WTF"`)
- `amt`: Amount of input asset to deposit (string representation of integer)
- `lock`: (Optional) User-specific lock duration in blocks. If provided, MUST be ≥ the vault's default lock. If omitted, uses vault default.

**Effect:** Converts input to underlying liquidity (via OPI-1 swap if needed), mints vTokens based on current NAV (Net Asset Value). Lock prevents premature redemption for stability.

**Global Lock Overwrite Rule:** If a user performs multiple deposits into the same vault, any new `vault init` operation MUST update the `lock_until_block` for the **entire vToken balance** of that address to the most distant expiration date (`max(old_lock_until_block, new_lock_until_block)`). This ensures O(1) state lookup and deterministic consensus across all indexers.

**Virtual Shares Protection:** On the very first deposit (`total_vtoken_supply == 0`), the indexer MUST burn a fixed minimum of `VIRTUAL_SHARES_BURN = 1000` vTokens to the null address. This creates a liquidity floor that prevents First Depositor Inflation Attacks.

**Validation Rules:**
- The input asset MUST be one of the tokens in the vault pair
- The vault MUST exist (deployed)
- The user MUST have sufficient balance of the input asset
- If `lock` is provided, it MUST be ≥ vault's default lock

#### Harvest Payload (`vault harvest`)

The "Heartbeat": Anyone can trigger the compounding.

```json
{
  "p": "brc-20",
  "op": "vault",
  "harvest": "LOL/WTF"
}
```

**Required Fields:**
- `p`: MUST be `"brc-20"`
- `op`: MUST be `"vault"`
- `harvest`: Vault ID (e.g., `"LOL/WTF"`)

**Logic (Executed by Indexer):**
1. Claim accumulated rewards (e.g., CRV from OPI-2 Curve)
2. Sell rewards for underlying assets (via `target_pool` OPI-1 swap)
3. Re-invest assets (add to locked liquidity, without minting new vTokens)
4. **Pay Bounty:** 1% of harvested rewards to the caller (incentivizes participation)
5. Deduct performance fee (e.g., 2%) to deployer or treasury null (burn)

**Anti-Flash:** Based on state at block `b-1`, like OPI-2.

**Harvest Cooldown Protection:** If `min_harvest_interval` is defined (or defaults to 144 blocks), any harvest attempt before this interval has passed since the last successful harvest is **INVALID** and yields no bounty. This prevents harvest spam and bounty exploitation.

**Validation Rules:**
- The vault MUST exist
- The vault MUST have active positions (total vToken supply > 0)
- The vault MUST have accumulated rewards to harvest (from OPI-2 Curve)
- If `min_harvest_interval` is set, `current_block - last_harvest_block` MUST be ≥ `min_harvest_interval` (otherwise transaction is INVALID)

#### Redemption Payload (`vault exe`)

The "Exit": Burn vTokens to retrieve assets.

```json
{
  "p": "brc-20",
  "op": "vault",
  "exe": "WTF,LOL/WTF",
  "amt": "500"
}
```

**Required Fields:**
- `p`: MUST be `"brc-20"`
- `op`: MUST be `"vault"`
- `exe`: Pair in format `"OUTPUT_ASSET,VAULT_ID"` (e.g., `"WTF,LOL/WTF"`)
- `amt`: Amount of vTokens to burn (string representation of integer, or `"all"` for full position)

**Effect:** Burns vTokens, calculates share of liquidity (based on appreciating Price), outputs desired asset (swap via OPI-1 if needed). Respects lock to prevent dumps.

**Validation Rules:**
- The vault MUST exist
- The user MUST have sufficient vToken balance
- The user's lock period MUST have expired (if lock was set)
- The output asset MUST be one of the tokens in the vault pair


### Indexer Validation Rules

Indexers act as **Provers**, validating operations without trust:

#### 1. Deploy Validation
```python
def validate_vault_deploy(deploy_tx):
    deploy_data = parse_op_return(deploy_tx.outputs[0])
    
    # Validate JSON schema
    assert deploy_data.p == "brc-20"
    assert deploy_data.op == "vault"
    assert deploy_data.deploy is not None
    
    # Validate vault ID format
    vault_id = deploy_data.deploy
    assert "/" in vault_id and vault_id.count("/") == 1
    assert not vault_id.lower().startswith("y")  # Avoid yToken conflicts
    
    # Validate strategy parameters
    strategy = deploy_data.strategy
    assert strategy.target_pool is not None
    assert strategy.fee is not None
    fee_bps = int(strategy.fee)
    assert 0 <= fee_bps <= 10000  # 0% to 100%
    
    # Validate target_pool exists (OPI-1 swap pool)
    assert pool_exists(strategy.target_pool)
    
    # Validate lock duration
    lock_blocks = int(deploy_data.lock)
    assert lock_blocks > 0
    
    # Resolve Vault Creator Address from Input[0]
    creator_address = resolve_address(deploy_tx.inputs[0])
    
    # Extract Genesis Fees (optional, requires fee_check flag)
    genesis_fee_init = 0
    genesis_fee_exe = 0
    fee_check = getattr(deploy_data, 'fee_check', False)
    if fee_check and len(deploy_tx.outputs) > 1:
        genesis_fee_init = deploy_tx.outputs[1].value
    if fee_check and len(deploy_tx.outputs) > 2:
        genesis_fee_exe = deploy_tx.outputs[2].value
    
    # Extract min_harvest_interval (optional, default 144 blocks)
    min_harvest_interval = 144  # Default: ~24 hours
    if hasattr(deploy_data, 'min_harvest_interval'):
        min_harvest_interval = int(deploy_data.min_harvest_interval)
        assert min_harvest_interval > 0
    
    # Initialize vault state
    create_vault_constitution(
        vault_id=vault_id,
        target_pool=strategy.target_pool,
        fee_bps=fee_bps,
        default_lock=lock_blocks,
        creator_address=creator_address,
        genesis_fee_init_sats=genesis_fee_init,
        genesis_fee_exe_sats=genesis_fee_exe,
        min_harvest_interval=min_harvest_interval,
        total_locked_liquidity=0,
        total_vtoken_supply=0,
        liquidity_index=1e27,  # RAY precision
        last_harvest_block=0
    )
```

#### 2. Init Validation
```python
def validate_vault_init(init_tx, vault_id):
    init_data = parse_op_return(init_tx.outputs[0])
    
    # Validate JSON schema
    assert init_data.p == "brc-20"
    assert init_data.op == "vault"
    assert init_data.init is not None
    
    # Parse init field
    input_asset, target_vault = init_data.init.split(",", 1)
    assert target_vault == vault_id
    
    # Validate vault exists
    const = get_vault_constitution(vault_id)
    assert const is not None
    
    # Validate input asset is in vault pair
    base, reward = vault_id.split("/")
    assert input_asset in [base, reward]
    
    # Validate Genesis Fee (if required)
    if const.genesis_fee_init_sats > 0:
        op_return_index = find_op_return_index(init_tx.outputs)
        fee_output = init_tx.outputs[op_return_index + 1]
        assert fee_output.value == const.genesis_fee_init_sats
        assert fee_output.address == const.creator_address
    
    # Validate user balance
    user_address = resolve_address(init_tx.inputs[0])
    amount = Decimal(init_data.amt)
    assert get_balance(user_address, input_asset) >= amount
    
    # Validate lock (if provided)
    user_lock = const.default_lock
    if init_data.lock:
        user_lock = int(init_data.lock)
        assert user_lock >= const.default_lock
    
    # Global Lock Overwrite: Get existing lock and use max(old, new)
    user_info = get_or_create_vault_user_info(vault_id, user_address)
    current_block = get_current_block()
    new_lock_until = current_block + user_lock
    if user_info.lock_until_block > 0:
        # Use most distant expiration date
        user_info.lock_until_block = max(user_info.lock_until_block, new_lock_until)
    else:
        user_info.lock_until_block = new_lock_until
    
    # Process deposit
    process_vault_deposit(
        vault_id=vault_id,
        user_address=user_address,
        input_asset=input_asset,
        amount=amount,
        user_lock=user_lock
    )
```

#### 3. Harvest Validation
```python
def validate_vault_harvest(harvest_tx, vault_id):
    harvest_data = parse_op_return(harvest_tx.outputs[0])
    
    # Validate JSON schema
    assert harvest_data.p == "brc-20"
    assert harvest_data.op == "vault"
    assert harvest_data.harvest == vault_id
    
    # Validate vault exists
    const = get_vault_constitution(vault_id)
    assert const is not None
    
    # Validate vault has active positions
    assert const.total_vtoken_supply > 0
    
    # Validate vault has accumulated rewards (from OPI-2 Curve)
    rewards = get_accumulated_rewards(vault_id)
    assert rewards > 0
    
    # Validate harvest cooldown (min_harvest_interval)
    current_block = get_current_block()
    if const.last_harvest_block > 0:
        blocks_since_harvest = current_block - const.last_harvest_block
        if blocks_since_harvest < const.min_harvest_interval:
            raise HarvestTooSoon()  # Transaction INVALID, no bounty
    
    # Process harvest
    caller_address = resolve_address(harvest_tx.inputs[0])
    process_vault_harvest(
        vault_id=vault_id,
        caller_address=caller_address,
        current_block=current_block
    )
```

#### 4. Exe Validation
```python
def validate_vault_exe(exe_tx, vault_id):
    exe_data = parse_op_return(exe_tx.outputs[0])
    
    # Validate JSON schema
    assert exe_data.p == "brc-20"
    assert exe_data.op == "vault"
    assert exe_data.exe is not None
    
    # Parse exe field
    output_asset, source_vault = exe_data.exe.split(",", 1)
    assert source_vault == vault_id
    
    # Validate vault exists
    const = get_vault_constitution(vault_id)
    assert const is not None
    
    # Validate Genesis Fee (if required)
    if const.genesis_fee_exe_sats > 0:
        op_return_index = find_op_return_index(exe_tx.outputs)
        fee_output = exe_tx.outputs[op_return_index + 1]
        assert fee_output.value == const.genesis_fee_exe_sats
        assert fee_output.address == const.creator_address
    
    # Validate user vToken balance
    user_address = resolve_address(exe_tx.inputs[0])
    vtoken_amount = Decimal(exe_data.amt) if exe_data.amt != "all" else get_balance(user_address, vault_id)
    assert get_balance(user_address, vault_id) >= vtoken_amount
    
    # Validate lock period expired
    user_info = get_vault_user_info(vault_id, user_address)
    if user_info.lock_until_block > 0:
        assert get_current_block() >= user_info.lock_until_block
    
    # Validate output asset is in vault pair
    base, reward = vault_id.split("/")
    assert output_asset in [base, reward]
    
    # Process redemption
    process_vault_redemption(
        vault_id=vault_id,
        user_address=user_address,
        vtoken_amount=vtoken_amount,
        output_asset=output_asset
    )
```

#### 5. Constitution Integrity Checks
```python
def validate_vault_integrity(vault_id):
    const = get_vault_constitution(vault_id)
    
    # Ensure vToken supply is never negative
    assert const.total_vtoken_supply >= 0
    
    # Ensure locked liquidity is never negative
    assert const.total_locked_liquidity >= 0
    
    # Ensure NAV calculation uses current state
    # liquidity_index must be updated before operations
    assert const.liquidity_index >= 1e27  # Minimum RAY value
    
    # Anti-Flash Protection: Use state at block b-1
    # This is enforced by sequential block processing
    # Indexer MUST process blocks sequentially and update
    # liquidity_index before every operation
```

---

## Rationale

Why this design? To maximize decentralization:

**Implicit Tickers:** Simplifies UX—no need for custom tickers, the pair is the ID (discoverable via Indexer). This reduces friction and makes vaults composable with existing OPI-1 pools.

**Public Harvest:** Avoids centralized keepers; bounty aligns incentives like the Void Rule (OPI-2). Anyone can trigger the strategy, making the protocol self-sustaining without requiring dedicated infrastructure.

**Elastic without Pre-Mine:** Avoids artificial inflation; value comes from real yields. vTokens are minted only when users deposit, and burned only when they redeem. The appreciating price reflects actual yield accumulation.

**Locks:** Protects against volatility, encourages long-term holding (aligned with W failsafe in OPI-3). Prevents flash-loan attacks and ensures stability of the vault's liquidity.

**Alternatives Rejected:**
- **Governance:** No governance mechanism—vault parameters are immutable post-deploy. This eliminates attack vectors from admin keys or malicious upgrades.
- **Treasury:** No treasury—rewards are ex-nihilo via OPI-2, eliminating honeypot risks.
- **Centralized Keepers:** Public harvest with bounty ensures decentralization while maintaining strategy execution.

---

## Backwards Compatibility

OPI-4 is fully backwards compatible with OPI-1, OPI-2, and OPI-3:

- **OPI-1 Integration:** Uses OPI-1 for internal swaps (converting rewards to underlying assets, swapping output assets on redemption)
- **OPI-2 Integration:** Integrates with OPI-2 for claiming and selling rewards from Curve programs
- **OPI-3 Integration:** Leverages OPI-3 for W as BTC collateral, enabling BTC-native vaults

No breaking changes—vaults are added as a programmable extension. Legacy BRC-20 remains static, but Universal gains composability. Non-compliant indexers that do not recognize the `vault` operation will simply ignore it, treating it as an unrecognized operation.

---

## Reference Implementation

- **Repo**: https://github.com/universal-brc20/simplicity
- **Branch**: opi-4-vault-final
- **Examples**: The reference implementation includes:
  - Complete indexer module for processing `vault deploy` transactions
  - Deposit/redemption logic with NAV calculation
  - Harvest automation with bounty distribution
  - Integration with OPI-1 (swap), OPI-2 (curve rewards), OPI-3 (W wrap)
  - Test cases with sample transactions and expected state transitions

---


## Test Vectors

### 1. Vault Deploy Transaction
```
Raw Hex:
020000000001010000000000000000000000000000000000000000000000000000000000000000ffffffff00010000000000000000[OP_RETURN_LENGTH]6a[OP_RETURN_HEX]010000000000000000[GENESIS_FEE_INIT]000000000000000000[GENESIS_FEE_EXE]00000000

OP_RETURN Payload (hex-encoded JSON):
7b2270223a226272632d3230222c226f70223a227661756c74222c226465706c6f79223a224c4f4c2f575446222c227374726174656779223a7b227461726765745f706f6f6c223a224352562f575446222c22666565223a22323030227d2c226c6f636b223a223532353630227d

Decoded:
{"p":"brc-20","op":"vault","deploy":"LOL/WTF","strategy":{"target_pool":"CRV/WTF","fee":"200"},"lock":"52560","fee_check":true,"min_harvest_interval":"144"}
```

### 2. Vault Init Transaction
```
Raw Hex:
02000000000101[previous_outpoint:36][previous_vout:4]00000000fdffffff01[OP_RETURN_LENGTH]6a[OP_RETURN_HEX][GENESIS_FEE_OUTPUT]00000000

OP_RETURN Payload (hex-encoded JSON):
7b2270223a226272632d3230222c226f70223a227661756c74222c22696e6974223a225754462c4c4f4c2f575446222c22616d74223a22313030303030222c226c6f636b223a2231303030227d

Decoded:
{"p":"brc-20","op":"vault","init":"WTF,LOL/WTF","amt":"10000","lock":"1000"}
```

### 3. Vault Harvest Transaction
```
Raw Hex:
02000000000101[previous_outpoint:36][previous_vout:4]00000000fdffffff01[OP_RETURN_LENGTH]6a[OP_RETURN_HEX]00000000

OP_RETURN Payload (hex-encoded JSON):
7b2270223a226272632d3230222c226f70223a227661756c74222c2268617276657374223a224c4f4c2f575446227d

Decoded:
{"p":"brc-20","op":"vault","harvest":"LOL/WTF"}
```

### 4. Vault Exe Transaction
```
Raw Hex:
02000000000101[previous_outpoint:36][previous_vout:4]00000000fdffffff01[OP_RETURN_LENGTH]6a[OP_RETURN_HEX][GENESIS_FEE_OUTPUT]00000000

OP_RETURN Payload (hex-encoded JSON):
7b2270223a226272632d3230222c226f70223a227661756c74222c22657865223a225754462c4c4f4c2f575446222c22616d74223a22353030227d

Decoded:
{"p":"brc-20","op":"vault","exe":"WTF,LOL/WTF","amt":"500"}
```

### 5. NAV Calculation Example
```
Initial State:
- total_locked_liquidity = 0
- total_vtoken_supply = 0
- NAV = 1.0 (default for empty vault)

After Deposit (10000 WTF):
- total_locked_liquidity = 10000
- total_vtoken_supply = 10000 - 1000 = 9000 (Virtual Shares burned)
- NAV = 10000 / 9000 = 1.111...

After Harvest (500 WTF added):
- total_locked_liquidity = 10500
- total_vtoken_supply = 10000 (unchanged)
- NAV = 10500 / 10000 = 1.05

After Redemption (500 vTokens):
- Assets returned = 500 × 1.05 = 525 WTF
- total_locked_liquidity = 9975
- total_vtoken_supply = 9500
- NAV = 9975 / 9500 = 1.05 (unchanged)
```

---

## Security Properties

### Mathematical Security

The vault model achieves **mathematical correctness** through:

**Conservation of Mass:**
- vTokens are minted only when assets are deposited
- vTokens are burned only when assets are redeemed
- Total locked liquidity always equals sum of user positions

**NAV Invariance:**
- NAV increases only through harvest operations (adding liquidity)
- NAV decreases only through redemption (removing liquidity)
- No inflation possible—vToken supply cannot exceed locked liquidity

### Game-Theoretic Security

The protocol achieves a **Nash Equilibrium** where cooperation is optimal:

**Payoff Matrix** (Example: 10,000 WTF vault):
| Scenario | User Payoff | Harvester Payoff | Protocol Health |
|----------|-------------|------------------|-----------------|
| **Harvest regularly** | NAV appreciates | 1% bounty | Optimal compounding |
| **Harvest rarely** | NAV grows slowly | Missed bounties | Suboptimal yield |
| **No harvest** | NAV stagnates | No bounty | Rewards accumulate but don't compound |

**Result**: Regular harvesting is the dominant strategy for all parties.

### Debit-Before-Credit Accounting

Critical for preventing inflation (inherited from OPI-2 and OPI-3):

```python
# CORRECT - Debit-Before-Credit
def vault_redemption(user, vtoken_amount):
    # 1. CHECK: Verify user balance
    if get_balance(user, vault_id) < vtoken_amount:
        raise InsufficientBalance()
    
    # 2. DEBIT: Burn vTokens FIRST
    burn_vtokens(user, vtoken_amount)
    
    # 3. CALCULATE: Assets based on current NAV
    assets = vtoken_amount * current_nav
    
    # 4. DEBIT: Remove from locked liquidity
    debit_locked_liquidity(assets)
    
    # 5. CREDIT: Add to user balance AFTER
    credit_balance(user, output_asset, assets)
```

This ordering is enforced for ALL operations.

---

## Implementation Notes

### Constants
```python
# Precision
RAY = 10**27  # Aave Standard Precision for liquidity_index

# Bounty and Fees
HARVEST_BOUNTY_BPS = 100  # 1% of harvested rewards to caller
MIN_PERFORMANCE_FEE_BPS = 0  # 0% minimum
MAX_PERFORMANCE_FEE_BPS = 10000  # 100% maximum

# Virtual Shares Protection (First Depositor Attack Mitigation)
VIRTUAL_SHARES_BURN = 1000  # Fixed minimum shares burned on first deposit

# Lock durations (in blocks)
MIN_LOCK_BLOCKS = 10  # Minimum lock period (anti-flash)
DEFAULT_LOCK_BLOCKS = 52560  # ~1 year at 10 min/block

# Harvest cooldown (in blocks)
DEFAULT_MIN_HARVEST_INTERVAL = 144  # ~24 hours at 10 min/block

# Vault limits
MIN_VAULT_LIQUIDITY = 0  # No minimum (can start empty)
MAX_VAULT_LIQUIDITY = Decimal('1e27')  # Practical limit
```

### Data Structures
```python
class VaultConstitution:
    vault_id: str  # Format: "BASE/REWARD"
    target_pool: str  # OPI-1 swap pool for selling rewards
    fee_bps: int  # Performance fee in basis points
    default_lock: int  # Default lock duration in blocks
    creator_address: str  # Vault creator (for Genesis Fees)
    genesis_fee_init_sats: int  # Genesis fee for init (0 if not set)
    genesis_fee_exe_sats: int  # Genesis fee for exe (0 if not set)
    min_harvest_interval: int  # Minimum blocks between harvests (default: 144)
    total_locked_liquidity: Decimal  # Total assets locked
    total_vtoken_supply: Decimal  # Total vTokens minted
    liquidity_index: Decimal  # RAY precision (init = 1e27)
    start_block: int  # Block when vault was deployed
    last_harvest_block: int  # Last block where harvest occurred (0 if never)

class VaultUserInfo:
    vault_id: str
    user_address: str
    vtoken_balance: Decimal  # User's vToken balance
    lock_until_block: int  # Block when lock expires (0 if no lock)
```

### Integration Points

**OPI-1 Integration:**
- Use `swap.exe` to convert rewards to underlying assets during harvest
- Use `swap.exe` to convert output assets during redemption if needed

**OPI-2 Integration:**
- Claim rewards from Curve programs using `swap.exe` with yTokens
- Rewards are ex-nihilo, eliminating honeypot risks

**OPI-3 Integration:**
- Vaults can hold W tokens as BTC collateral
- Enables BTC-native yield strategies

---

## Rationale

### Why Implicit Tickers?

Using the pair format (`"BASE/REWARD"`) as the vault ID:
- **Simplifies Discovery**: No need to deploy custom tokens—the pair is the identifier
- **Enables Composability**: Vaults integrate naturally with OPI-1 swap pools
- **Reduces Friction**: Users don't need to remember custom ticker names
- **Prevents Conflicts**: Format validation ensures no collision with yTokens (OPI-2)

### Why Public Harvest?

Allowing anyone to trigger harvest with a bounty:
- **Eliminates Centralization**: No need for dedicated keepers or off-chain infrastructure
- **Aligns Incentives**: Bounty (1%) ensures harvesters are compensated for gas costs
- **Ensures Execution**: Strategy runs perpetually as long as rewards exist
- **Community-Driven**: Protocol health becomes a community responsibility

### Why Elastic Supply?

vTokens have no `max_supply`:
- **Reflects Reality**: Supply grows and shrinks with user activity
- **Prevents Pre-Mine**: No tokens exist until users deposit
- **Mathematical Correctness**: NAV appreciation comes from real yield, not artificial inflation
- **Composability**: Elastic supply enables integration with other DeFi protocols

### Why RAY Precision?

Using 10²⁷ precision for `liquidity_index`:
- **Sub-Satoshi Accuracy**: Prevents rounding errors in NAV calculations
- **Industry Standard**: Same precision as Aave's rebasing model
- **O(1) Complexity**: Enables efficient calculations without block iteration
- **Future-Proof**: High precision accommodates long-duration vaults

### Why Lock Mechanism?

User-specific locks prevent premature redemptions:
- **Stability**: Protects vault from volatility and flash-loan attacks
- **Long-Term Alignment**: Encourages users to hold vTokens for yield accumulation
- **Flexibility**: Users can set longer locks for potentially higher rewards
- **Consistency**: Aligns with OPI-3's time-locked recovery paths

---

## Security Considerations

### Attack Vectors and Mitigations

| Attack Vector | Mitigation |
|---------------|------------|
| **Flash-deposit attack** | State calculations use block `b-1`, requiring capital commitment before block begins |
| **Harvest spam** | `min_harvest_interval` cooldown prevents harvest spam and bounty exploitation |
| **NAV manipulation** | NAV calculated from on-chain state (OPI-1 reserves, OPI-2 rewards) |
| **Lock bypass** | Lock enforced at indexer level, preventing redemption before expiration |
| **First Depositor Inflation Attack** | Virtual Shares mechanism burns 1000 shares on first deposit, creating liquidity floor |
| **Inflation attack** | Debit-before-credit accounting ensures vToken supply ≤ locked liquidity |
| **Oracle manipulation** | Fully on-chain—no external oracles required |
| **Partial fill exploit** | OPI-1 swap integration handles partial fills with slippage tolerance |
| **Genesis Fee bypass** | Fee validation rejects transactions with incorrect amounts or addresses |

### On-Chain Verifiability

- **Genesis Fees**: Verifiable through `deploy` transaction outputs
- **Vault Parameters**: Immutably encoded in `deploy` transaction
- **State Tracking**: All operations recorded on-chain for full auditability
- **NAV Calculation**: Deterministic formula using on-chain state

### Economic Security

- **No Pre-Mine**: All vTokens minted only on deposit
- **Appreciating NAV**: Value comes from real yield accumulation
- **Performance Fee**: Sustainable funding without inflationary dilution
- **Bounty Mechanism**: Ensures perpetual execution without centralization

---

## Backwards Compatibility

OPI-4 maintains full backward compatibility:

1. **Selective Processing**: Only OPI-4 aware indexers process `vault` operations
2. **Standard BRC-20 Format**: Uses existing JSON payload structure
3. **Graceful Degradation**: Legacy indexers safely ignore vault operations
4. **State Isolation**: Vault operations don't affect other BRC-20 tokens
5. **Integration**: Composes with OPI-1, OPI-2, and OPI-3 without breaking changes

---

## Future Work

- **Multi-Asset Vaults**: Support for vaults with more than two assets
- **Strategy Templates**: Pre-defined strategy templates for common yield patterns
- **Governance**: Decentralized parameter updates (if community consensus is reached)
- **Cross-Vault Swaps**: Direct swaps between vault vTokens
- **Leverage**: Integration with lending protocols for leveraged yield strategies
- **Insurance**: Optional insurance pools for vault risk management

---

## Conclusion

The Vault Standard establishes Bitcoin as a foundation for **Automated Yield Strategies** where mathematical security replaces trust. Through elastic vTokens, public harvest, and implicit tickers, combined with integration across OPI-1, OPI-2, and OPI-3, it achieves:

1. **Trustless Automation**: Strategies execute without centralized keepers
2. **Mathematical Security**: NAV appreciation reflects real yield, not inflation
3. **Community Alignment**: Bounty mechanism ensures perpetual execution
4. **Full Composability**: Vaults integrate seamlessly with existing DeFi primitives

This is not just a vault protocol. It's a paradigm shift from "manual yield farming" to "automated yield factories" on Bitcoin.

**The age of Automated Strategy Vaults begins with OPI-4.**

---

## References

1. Nakamoto, S. (2008). "Bitcoin: A Peer-to-Peer Electronic Cash System"
2. Domo (2023). "BRC-20: An experimental fungible token standard"
3. Laz1m0v (2025). "OPI-1: The Swap Standard for Automated Market Makers"
4. Laz1m0v (2025). "OPI-2: The Curve Extension for Trustless Programmatic Token Distribution"
5. Laz1m0v, GaloisField2718 (2025). "OPI-3: The W Standard for State-Bound Wrapped Assets"
6. Aave Protocol (2020). "Aave: A Non-Custodial Liquidity Protocol"

---

## Appendix: Reference Implementation

```python
def calculate_nav(vault_id: str) -> Decimal:
    """Calculate Net Asset Value for a vault"""
    const = get_vault_constitution(vault_id)
    
    if const.total_vtoken_supply == 0:
        return Decimal("1.0")  # Default NAV for empty vault
    
    # NAV = Total Locked Liquidity / Total vToken Supply
    nav = const.total_locked_liquidity / const.total_vtoken_supply
    return nav

def process_vault_deposit(
    vault_id: str,
    user_address: str,
    input_asset: str,
    amount: Decimal,
    user_lock: int
):
    """Process vault deposit and mint vTokens"""
    const = get_vault_constitution(vault_id)
    current_block = get_current_block()
    
    # Update liquidity_index (lazy evaluation)
    update_liquidity_index(vault_id, current_block)
    
    # Calculate current NAV
    nav = calculate_nav(vault_id)
    
    # Convert input to liquidity (swap if needed via OPI-1)
    liquidity_added = convert_to_liquidity(input_asset, amount, vault_id)
    
    # Calculate vTokens to mint
    is_first_deposit = (const.total_vtoken_supply == 0)
    if is_first_deposit:
        # First deposit: 1:1 ratio
        vtokens_minted = liquidity_added
        # Virtual Shares Protection: Burn fixed minimum to prevent First Depositor Attack
        virtual_shares_burned = VIRTUAL_SHARES_BURN
        # Ensure user receives at least 1 vToken net (must deposit > VIRTUAL_SHARES_BURN)
        if vtokens_minted <= virtual_shares_burned:
            raise InsufficientDepositForVirtualShares()
    else:
        vtokens_minted = liquidity_added / nav
    
    # Update vault state
    const.total_locked_liquidity += liquidity_added
    const.total_vtoken_supply += vtokens_minted
    
    # Virtual Shares: Burn on first deposit
    if is_first_deposit:
        const.total_vtoken_supply -= virtual_shares_burned
        # Burn to null address (recorded in state, not credited to any user)
    
    # Update user info
    user_info = get_or_create_vault_user_info(vault_id, user_address)
    user_info.vtoken_balance += vtokens_minted
    user_info.lock_until_block = current_block + user_lock
    
    # Mint vTokens (credit user balance)
    credit_balance(user_address, vault_id, vtokens_minted)

def process_vault_harvest(
    vault_id: str,
    caller_address: str,
    current_block: int
):
    """Process vault harvest: claim, sell, re-invest, pay bounty"""
    const = get_vault_constitution(vault_id)
    
    # Update liquidity_index
    update_liquidity_index(vault_id, current_block)
    
    # Claim rewards from OPI-2 Curve
    rewards = claim_curve_rewards(vault_id)
    if rewards == 0:
        return  # No rewards to harvest
    
    # Sell rewards for underlying assets via target_pool (OPI-1)
    underlying_assets = sell_rewards_via_swap(
        rewards=rewards,
        target_pool=const.target_pool,
        vault_id=vault_id
    )
    
    # Calculate bounty (1% of underlying assets)
    bounty = underlying_assets * Decimal("0.01")
    
    # Calculate performance fee (e.g., 2% = 200 bps)
    performance_fee = underlying_assets * Decimal(const.fee_bps) / Decimal("10000")
    
    # Re-invest remaining assets (add to locked liquidity)
    reinvested = underlying_assets - bounty - performance_fee
    const.total_locked_liquidity += reinvested
    
    # Pay bounty to caller
    credit_balance(caller_address, get_base_asset(vault_id), bounty)
    
    # Pay performance fee to creator (or burn if treasury is null)
    if const.creator_address:
        credit_balance(const.creator_address, get_base_asset(vault_id), performance_fee)
    # else: burn (not implemented in this example)
    
    # Update last harvest block
    const.last_harvest_block = current_block

def process_vault_redemption(
    vault_id: str,
    user_address: str,
    vtoken_amount: Decimal,
    output_asset: str
):
    """Process vault redemption: burn vTokens, return assets"""
    const = get_vault_constitution(vault_id)
    current_block = get_current_block()
    
    # Update liquidity_index
    update_liquidity_index(vault_id, current_block)
    
    # Validate lock period
    user_info = get_vault_user_info(vault_id, user_address)
    if user_info.lock_until_block > 0 and current_block < user_info.lock_until_block:
        raise LockPeriodNotExpired()
    
    # Calculate current NAV
    nav = calculate_nav(vault_id)
    
    # Calculate assets to return
    assets_to_return = vtoken_amount * nav
    
    # Burn vTokens (debit user balance)
    debit_balance(user_address, vault_id, vtoken_amount)
    
    # Debit from locked liquidity
    const.total_locked_liquidity -= assets_to_return
    const.total_vtoken_supply -= vtoken_amount
    
    # Update user info
    user_info.vtoken_balance -= vtoken_amount
    
    # Convert to output asset (swap if needed via OPI-1)
    output_amount = convert_to_output_asset(
        assets_to_return,
        output_asset,
        vault_id
    )
    
    # Credit user with output asset
    credit_balance(user_address, output_asset, output_amount)
```
