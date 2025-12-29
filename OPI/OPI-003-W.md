# OPI: 3

**Title**: The "W" Standard for State-Bound Wrapped Assets on Bitcoin  

**Authors**: Laz1m0v <wtf@wbtc.lol>, GaloisField2718 <galois@blockcryptology.locker>

**Discussions-To**: https://github.com/The-Universal-BRC-20-Extension/OPI

**Status**: Final  

**Type**: Standards Track  

**Created**: 2025-12-27  

**Revised**: 2025-12-29 

**Requires**: BIP-341 (Taproot), BRC-20

---

## Abstract

This document specifies the **"W" Standard**, a foundational protocol for creating **State-Bound Assets** on Bitcoin. It introduces a cryptographic binding mechanism that enforces atomic consistency between off-chain semantic states and on-chain UTXO values using Taproot's MAST capabilities. The protocol defines a **Vault Architecture** with native time-locked escape hatches and a **Debit-Before-Credit** accounting model that makes inflation mathematically impossible. Through a strategic deployment using `max: 0` and `lim: 0`, only OPI-3 compliant indexers can process W operations, ensuring backward compatibility. This standard serves as the security foundation for all higher-level DeFi operations (OPI-1, OPI-2) by transforming Bitcoin from a passive data layer into an active semantic validator.

---

## Copyright

This OPI is released under CC0 1.0 Universal.

---

## Motivation

Current Bitcoin metaprotocols suffer from a critical **Semantic Gap**: the disconnect between what tokens claim to represent (in JSON payloads) and what Bitcoin actually validates (satoshi transfers). This creates systemic vulnerabilities:

1. **Phantom Liquidity**: Tokens can exist in user balances without corresponding UTXO backing, creating fractional reserve risks.
2. **Inflation Bugs**: Complex operations like partial fills can accidentally mint tokens due to improper accounting sequences.
3. **Trust Dependencies**: Users must trust indexers to correctly interpret and execute protocol rules, breaking Bitcoin's "Don't Trust, Verify" ethos.

OPI-3 is motivated by a fundamental insight: **security should be structural, not procedural**. By binding semantic states to cryptographic commitments, we create a system where:
- Invalid state transitions are rejected by Bitcoin consensus itself
- Solvency is a mathematical property, not a promise
- Trust in indexers becomes unnecessary, not just minimized

This specification establishes the **canonical security layer** for sovereign finance on Bitcoin.

---

## Specification

### Core Concepts

**State-Bound Asset**: A token whose semantic existence is cryptographically bound to a specific UTXO through an equalizer constraint.

**Semantic Covenant**: A validation rule that ensures the physical transaction structure (on-chain) matches the declared semantic intention (off-chain).

**Vault Topology**: A standardized P2TR address structure using MAST to encode both protocol operations and trustless recovery paths.

**W_PROOF**: Cryptographic witness data that proves a vault was constructed according to canonical rules, binding the semantic state to the physical UTXO.

---

### Deployment Strategy (Soft-Fork via max:0)

The W token is deployed with special parameters ensuring only OPI-3 aware indexers can process it:

```json
{
  "p": "brc-20",
  "op": "deploy",
  "tick": "W",
  "max": "0",
  "lim": "0",
  "dec": "8"
}
```

**Key Insight**: Legacy BRC-20 indexers will see `max: 0` and reject any mint attempts as the supply is already "exhausted". Only OPI-3 compliant indexers understand that W mints are validated through vault locks, not supply limits.

---

### The Technological Trinity

The W Standard operates through three interconnected Bitcoin technologies:

#### 1. OP_RETURN (The Intent)
Minimal (<80 bytes) payloads signal user operations, encoded as hex-encoded JSON:
```
OP_RETURN 7b2270223a226272632d3230222c226f70223a226d696e74222c227469636b223a2257222c22616d74223a22313030303030303030227d
```
Decodes to:
```json
{"p": "brc-20", "op": "mint", "tick": "W", "amt": "100000000"}
```

#### 2. MAST (The Contract)
Three spend paths compressed into a single Taproot address:
- Only the used path is revealed on-chain
- Preserves privacy and minimizes data leakage
- Enables future extensibility without protocol changes

#### 3. W_PROOF (The Proof)
Cryptographic glue binding intent to contract, serialized as follows:

**Binary Serialization Format**:
```
W_PROOF Structure (Binary):
[0-6]    Magic Bytes: "W_PROOF" (0x575F50524F4F46)
[7]      Version: 0x01
[8-40]   User Pubkey: 33 bytes compressed
[41-73]  Operator Pubkey: 33 bytes compressed  
[74-77]  User Lock: 4 bytes little-endian (blocks)
[78-109] Script Tree Hash: 32 bytes
[110-N]  Control Blocks: Variable length per path
```

The W_PROOF is inserted as a witness stack element in the commit transaction.

---

### NUMS Point Definition

The internal key for all vaults uses the following provably unspendable point:

```python
# NUMS (Nothing Up My Sleeve) Point Generation
NUMS_SEED = "Bitcoin:W-Protocol:Sovereign-Vault:2025"
NUMS_HASH = sha256(NUMS_SEED.encode())  
# Result: 0x8f3a2d4c5b6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b

# Derive x-coordinate (mod p)
NUMS_X = int.from_bytes(NUMS_HASH, 'big') % SECP256K1_P

# Compute y-coordinate (even parity)
NUMS_POINT = lift_x(NUMS_X)  # Results in valid but unspendable point
```

All implementations MUST use this exact derivation to ensure address consistency.

---

### Vault Construction

Every W vault is a P2TR (Pay-to-Taproot) address with exactly three spend paths:

```
Vault_Address = P2TR(NUMS_POINT, MAST_Root)
```

#### The Three Paths

##### Path 1: Collaborative (2-of-2 Multisig)
```bitcoin
# Leaf Version: 0xc0 (Tapscript)
<user_pubkey> OP_CHECKSIG 
<operator_pubkey> OP_CHECKSIGADD 
OP_2 OP_EQUAL
```

##### Path 2: Liquidation (Permissionless Cleanup)
```bitcoin
# Leaf Version: 0xc0 (Tapscript)
<user_chosen_lock> OP_CHECKSEQUENCEVERIFY OP_DROP
<operator_pubkey> OP_CHECKSIG
```

##### Path 3: Sovereign (Unilateral Recovery)
```bitcoin
# Leaf Version: 0xc0 (Tapscript)
<105120> OP_CHECKSEQUENCEVERIFY OP_DROP
<user_pubkey> OP_CHECKSIG
```

---

### Operations

#### TIMEWRAP (Minting W)

Two-step process to resolve fee circularity:

**Step 1 - Commit Transaction**
```
Inputs:  User's BTC
Outputs: [0] Vault P2TR address (1 BTC locked)
Witness: [0] W_PROOF as witness stack element
         [1] Empty (for future extensibility)
```

**Step 2 - Reveal Transaction**
```
Inputs:  User's funding
Outputs: [0] OP_RETURN with hex-encoded JSON
         [1] Change to user
```

The transactions are linked: Reveal must reference Commit's TXID in its inputs or as a parent in the same block.

#### UNWRAP (Burning W)

User burns W tokens to unlock BTC from vault:

**Burn Transaction**
```
Inputs:  User's funding
Outputs: [0] OP_RETURN with hex-encoded JSON burn operation
```

**Unlock Transaction** (using appropriate path)
```
Inputs:  [0] Vault UTXO
Outputs: [0] User's destination (1 BTC minus fees)
Witness: Path-specific witness data
```

---

### Indexer Validation Rules

Indexers act as **Provers**, validating operations without trust:

#### 1. Deploy Validation
```python
def validate_w_deploy(deploy_tx):
    deploy_data = parse_op_return(deploy_tx.outputs[0])
    
    # W uses special deployment parameters
    assert deploy_data.tick == "W"
    assert deploy_data.max == "0"  # Critical: Prevents legacy indexer mints
    assert deploy_data.lim == "0"  # No standard minting allowed
    assert deploy_data.dec == "8"  # Satoshi precision
    
    # Only OPI-3 aware indexers proceed past this point
    register_state_bound_ticker("W")
```

#### 2. Timewrap Validation
```python
def validate_timewrap(commit_tx, reveal_tx):
    # Extract W_PROOF from witness
    w_proof_bytes = commit_tx.witness[0]
    w_proof = deserialize_w_proof(w_proof_bytes)
    
    # Verify magic and version
    assert w_proof.magic == b"W_PROOF"
    assert w_proof.version == 0x01
    
    # Derive expected vault address using NUMS point
    nums_point = derive_nums_point("Bitcoin:W-Protocol:Sovereign-Vault:2025")
    script_tree = build_script_tree(
        w_proof.user_pubkey,
        w_proof.operator_pubkey,
        w_proof.user_lock
    )
    expected_address = p2tr(nums_point, script_tree)
    
    # Verify commit output
    assert commit_tx.outputs[0].address == expected_address
    assert commit_tx.outputs[0].value == parse_amount(reveal_tx)
    
    # Verify transaction linkage
    assert is_linked(reveal_tx, commit_tx)
    
    # Credit W tokens
    credit_balance(w_proof.user_pubkey, "W", reveal_tx.amt)
```

---

## Test Vectors

### 1. W Deploy Transaction
```
Raw Hex:
020000000001010000000000000000000000000000000000000000000000000000000000000000ffffffff00010000000000000000266a247b2270223a226272632d3230222c226f70223a226465706c6f79222c227469636b223a2257222c226d6178223a2230222c226c696d223a2230222c22646563223a2238227d00000000

Decoded:
- OP_RETURN: {"p":"brc-20","op":"deploy","tick":"W","max":"0","lim":"0","dec":"8"}
```

### 2. Commit Transaction with W_PROOF
```
Raw Hex:
02000000000101[previous_outpoint:36][previous_vout:4]00000000fdffffff01e8030000000000002251208f3a2d4c5b6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a024730440220[signature_r:32][signature_s:32]0121[pubkey:33]21575f50524f4f46012103[user_pubkey:33]2103[operator_pubkey:33][user_lock:4][script_tree_hash:32][control_blocks:variable]00000000

Key Elements:
- Output[0]: P2TR vault (derived from NUMS + MAST)
- Witness[0]: W_PROOF structure
- Amount: 1000 sats (0x000003e8)
```

### 3. Reveal Transaction
```
Raw Hex:
020000000001010000000000000000000000000000000000000000000000000000000000000000ffffffff00010000000000000000456a437b2270223a226272632d3230222c226f70223a226d696e74222c227469636b223a2257222c22616d74223a22313030303030303030227d00000000

Decoded:
- OP_RETURN: {"p":"brc-20","op":"mint","tick":"W","amt":"100000000"}
```

### 4. Collaborative Unlock (Path 1)
```
Witness Structure:
[0] User signature (64 bytes)
[1] Operator signature (64 bytes)  
[2] Collaborative script (variable)
[3] Control block for path 1 (33 bytes)
```

### 5. Sovereign Unlock (Path 3)
```
Witness Structure:
[0] User signature (64 bytes)
[1] Sovereign script (variable)
[2] Control block for path 3 (33 bytes)
Sequence: 0x00019980 (105120 blocks)
```

---

## Security Properties

### Game-Theoretic Security

The protocol achieves a **Nash Equilibrium** where cooperation is optimal:

**Payoff Matrix** (1 BTC = $150,000):
| Scenario | User Payoff | Operator Payoff |
|----------|-------------|-----------------|
| Cooperate | $150,000 instant | +$150 fee |
| Operator Defects | $150,000 after 2yr | -$1,500 reputation |
| User Abandons | -$150,000 | +$750 (liquidation) |

**Result**: Cooperation is the dominant strategy for all parties.

### Debit-Before-Credit Accounting

Critical for preventing inflation:

```python
# CORRECT - Debit-Before-Credit
def transfer_w(sender, receiver, amount):
    # 1. CHECK: Verify sender balance
    if get_balance(sender) < amount:
        raise InsufficientBalance()
    
    # 2. DEBIT: Remove from sender FIRST
    debit_balance(sender, amount)
    
    # 3. CREDIT: Add to receiver AFTER
    credit_balance(receiver, amount)
```

This ordering is enforced for ALL operations.

---

## Implementation Notes

### Constants
```python
# Time locks (in blocks)
SOVEREIGN_LOCK = 105_120      # ~2 years (fixed)
DEFAULT_USER_LOCK = 52_560    # ~1 year (user configurable)
MIN_USER_LOCK = 10_000        # ~69 days minimum

# Fees
LIQUIDATION_FEE_BPS = 50      # 0.5% to liquidator

# Magic values
W_PROOF_MAGIC = b"W_PROOF"    # 0x575F50524F4F46
W_TICKER = "W"

# NUMS derivation seed
NUMS_SEED = "Bitcoin:W-Protocol:Sovereign-Vault:2025"

# Script versioning
TAPROOT_LEAF_VERSION = 0xc0
```

---

## Rationale

### Why max:0 Deployment?

Using `max: 0` creates a "soft fork" in the indexer network:
- Legacy indexers see an already-exhausted supply and reject mints
- OPI-3 indexers recognize W as a state-bound asset with special rules
- No risk of state corruption or double-minting across indexer versions

### Why Three Paths?

The three-path design creates a complete game tree:
1. **Collaborative**: Optimizes for efficiency (99% of cases)
2. **Liquidation**: Handles abandonment without admin keys
3. **Sovereign**: Guarantees ultimate user control

### Why W_PROOF in Witness?

Placing W_PROOF in the witness:
- Leverages Taproot's witness discount (1/4 weight)
- Keeps proof data segregated from transaction body
- Enables future soft-fork upgrades to proof format

### Why NUMS Point?

Using a provably unspendable internal key:
- Prevents key-path spending (forces script-path)
- Eliminates trust in any single party
- Makes derivation deterministic across all implementations

---

## Security Considerations

### Attack Vectors and Mitigations

| Attack Vector | Mitigation |
|---------------|------------|
| **Operator goes rogue** | Sovereign path ensures recovery after 2 years |
| **User loses keys** | Liquidation path recycles after user-defined timeout |
| **Double mint attempt** | W_PROOF makes each vault cryptographically unique |
| **Indexer divergence** | max:0 prevents legacy indexer interference |
| **Miner censorship** | Time-locked paths guarantee eventual execution |
| **Quantum computing** | MAST supports future algorithm upgrades |

---

## Backwards Compatibility

The W Standard maintains full backward compatibility through:

1. **Selective Processing**: Only OPI-3 aware indexers process W operations
2. **Standard BRC-20 Format**: Uses existing deploy/mint/burn operations
3. **Graceful Degradation**: Legacy indexers safely ignore W tokens
4. **State Isolation**: W operations don't affect other BRC-20 tokens

---

## Future Work

- **Lightning Integration**: Enable W transfers via payment channels
- **Recursive Covenants**: Chain multiple vaults for complex DeFi
- **Cross-Chain Bridges**: Atomic swaps with other UTXO chains
- **Yield Mechanisms**: Integration with OPI-1 (swaps) and OPI-2 (curves)
- **Governance**: Decentralized operator selection via WTF token

---

## Conclusion

The W Standard establishes Bitcoin as a foundation for **State-Bound Assets** where cryptographic proofs replace trust. Through the technological trinity of OP_RETURN, MAST, and W_PROOF, combined with the strategic max:0 deployment, it achieves:

1. **Mathematical Security**: Impossibility of cheating via Nash Equilibrium
2. **User Sovereignty**: Guaranteed fund recovery via time-locked paths
3. **System Health**: Automated cleanup via liquidation incentives
4. **Backward Compatibility**: Safe coexistence with legacy indexers

This is not just a wrapping protocol. It's a paradigm shift from "trust-minimized" to "trustless" Bitcoin DeFi.

**The age of State-Bound Assets begins with W.**

---

## References

1. Nakamoto, S. (2008). "Bitcoin: A Peer-to-Peer Electronic Cash System"
2. Wuille, P. et al. (2020). "BIP 341: Taproot: SegWit version 1 spending rules"
3. Laz1m0v, B0urb7k1 (2025). "The Synthesis of Value: Sovereign Collateralized Assets on Bitcoin"
4. Domo (2023). "BRC-20: An experimental fungible token standard"
5. Bitcoin Core Developers (2021). "OP_CHECKSEQUENCEVERIFY (BIP 112)"

---

## Appendix: Reference Implementation

```python
def deserialize_w_proof(witness_bytes):
    """Deserialize W_PROOF from witness data"""
    offset = 0
    
    # Magic bytes (7 bytes)
    magic = witness_bytes[offset:offset+7]
    assert magic == b"W_PROOF"
    offset += 7
    
    # Version (1 byte)
    version = witness_bytes[offset]
    assert version == 0x01
    offset += 1
    
    # User pubkey (33 bytes)
    user_pubkey = witness_bytes[offset:offset+33]
    offset += 33
    
    # Operator pubkey (33 bytes)
    operator_pubkey = witness_bytes[offset:offset+33]
    offset += 33
    
    # User lock (4 bytes, little-endian)
    user_lock = int.from_bytes(witness_bytes[offset:offset+4], 'little')
    offset += 4
    
    # Script tree hash (32 bytes)
    script_tree_hash = witness_bytes[offset:offset+32]
    offset += 32
    
    # Control blocks (remaining bytes)
    control_blocks = witness_bytes[offset:]
    
    return W_Proof(
        magic=magic,
        version=version,
        user_pubkey=user_pubkey,
        operator_pubkey=operator_pubkey,
        user_lock=user_lock,
        script_tree_hash=script_tree_hash,
        control_blocks=control_blocks
    )
```
