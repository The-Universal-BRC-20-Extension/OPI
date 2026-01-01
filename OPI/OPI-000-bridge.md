# OPI: 0

**Title**: The Bridge Standard: Namespace Migration for Legacy BRC-20 Ordinals  
**Author**: Laz1m0v <wtf@wbtc.lol>  
**Discussions-To**: https://t.me/universalbrc20  
**Status**: Draft  
**Type**: Standards Track
**Created**: 2025-07-07
**Revised**: 2026-01-01
**Requires**: BRC-20

---

## Abstract

This document specifies the **Bridge Standard** (OPI-0), enabling users to migrate legacy BRC-20 tickers from the Ordinals ecosystem into Universal BRC-20 through an **atomic burn-to-mint** mechanism enforced by Bitcoin Script.

**Key Innovation**: This is **not a wrapper**. Instead, it is a **Hostile Takeover** of the ticker namespace. On Universal, tickers are **virgin** until a Legacy ticker is destroyed. The protocol authorizes deployment **IF AND ONLY IF** proof of destruction is provided through a **Hash Preimage Covenant** enforced by Bitcoin Script.

The bridge operates through: (1) **Commitment** (off-chain): user calculates `H = SHA256(Burn_Payload)`, (2) **Lock** (on-chain): ordinal sent to Taproot vault with Hash Preimage Covenant, (3) **Burn & Reveal** (on-chain): vault spent by revealing Burn JSON preimage in Witness Data, triggering Universal mint.

---

## Copyright

This OPI is released under the MIT License.

---

## Motivation

Legacy BRC-20 tickers exist in the Ordinals namespace but cannot be used in Universal BRC-20. Users want to migrate their tickers while maintaining the same ticker identity.

**The Solution**: **Namespace Migration via Hostile Takeover**. The Bridge Standard enables atomic migration through:
- **Namespace Migration**: Universal ticker is **virgin** until Legacy ticker is destroyed
- **Atomic Burn-to-Mint**: Legacy ordinal locked in Taproot vault with Hash Preimage Covenant. Vault **cannot be spent** without revealing Burn JSON preimage
- **Cryptographic Enforcement**: Bitcoin Script enforces atomicity cryptographically

---

## Specification

### Core Concepts

**Namespace Migration**: Transferring a ticker from Legacy BRC-20 (Ordinals) to Universal BRC-20. Ticker identity preserved (e.g., `ORDI` remains `ORDI`).

**Hash Preimage Covenant**: A Bitcoin Script technique that cryptographically links an on-chain spend to a specific piece of data (the "preimage") that must be revealed in the transaction witness. The covenant enforces that a hash value `H = SHA256(Data)` is committed to in the script, and to spend the UTXO, the actual data (preimage) must be provided in the witness. Bitcoin Script validates that `SHA256(Provided_Data) == H` before allowing the spend.

**Hash Preimage Covenant Script**: The vault uses a Taproot script that enforces atomicity by requiring the revelation of the Burn JSON preimage before allowing the vault UTXO to be spent. The script is:

```bitcoin
OP_SHA256 <commitment_hash> OP_EQUALVERIFY <user_pubkey> OP_CHECKSIG
```

**Script Execution Flow**:
1. **`OP_SHA256`**: Hashes the top stack item (the preimage from Witness[1])
2. **`<commitment_hash>`**: Pushes the committed hash value (32 bytes) onto the stack
3. **`OP_EQUALVERIFY`**: Compares the two hashes. If they don't match, the script fails and Bitcoin **rejects the transaction**
4. **`<user_pubkey> OP_CHECKSIG`**: Validates the user's signature

**Critical Property**: The user **cannot** spend the UTXO without revealing the exact data that hashes to the commitment hash. If the preimage doesn't match, `OP_EQUALVERIFY` fails and Bitcoin **rejects the transaction**. This creates a **cryptographic guarantee** of atomicity—the burn must occur before the mint can happen.

**Vault Address**: Deterministic P2TR address derived from user pubkey, commitment hash, and standardized tweak `SHA256("UniversalBridge")`. The address can be calculated off-chain before the bridge operation, enabling trustless verification.

### Transaction Structure

#### Transaction 1: Lock (Commitment On-Chain)

| Component | Description |
| --------- | ------------------------------------------- |
| **Input[0]** | SegWit input containing legacy BRC-20 `transfer` inscription |
| **Output[0]** | `OP_RETURN` with minimal `bridge lock` payload |
| **Output[1]** | **Genesis Fee (Optional)** - If protocol has fees defined |
| **Output[2]** | **Vault P2TR Address** - Ordinal sent here (locked in vault) |
| **Witness[0]** | `<commitment_hash_bytes>` - 32 bytes raw |

#### Transaction 2: Burn & Reveal (Unlock Vault)

| Component | Description |
| --------- | ------------------------------------------- |
| **Input[0]** | Vault UTXO (locked ordinal) |
| **Output[0]** | **Burn Address** (e.g., Satoshi address) - Ordinal burned here |
| **Witness[0]** | `<user_signature>` |
| **Witness[1]** | `<burn_payload_json_bytes>` - **THE PREIMAGE** (Burn JSON as bytes) |
| **Witness[2]** | `<script>` - Hash Preimage Covenant script |
| **Witness[3]** | `<control_block>` |

**Critical**: Bitcoin Script validates `SHA256(Witness[1]) == commitment_hash` before allowing spend. No OP_RETURN needed—data in Witness saves ~75% fees.

### OP_RETURN Payload

#### Bridge Lock Payload (Transaction 1)

```json
{
  "p": "brc-20",
  "op": "bridge",
  "tick": "ORDI",
  "amt": "100"
}
```

**Witness[0]**: `<commitment_hash_bytes>` - SHA256 hash of Burn JSON (32 bytes raw, not hex-encoded).

**Burn JSON Format** (for commitment hash calculation):
```json
{
  "p": "brc-20",
  "op": "bridge",
  "burn": "ORDI",
  "amt": "100"
}
```

### Commitment Hash Calculation

The commitment hash is a **32-byte SHA256 hash** of the Burn JSON payload that will be revealed in Transaction 2. This hash is calculated **off-chain** before Transaction 1 and committed to in the vault script.

**Step-by-Step Process**:

1. **Prepare Burn JSON Payload**:
   ```json
   {
     "p": "brc-20",
     "op": "bridge",
     "burn": "ORDI",
     "amt": "100"
   }
   ```

2. **Serialize JSON (No Whitespace)**:
   - Use `json.dumps(burn_json, separators=(',', ':'))` to serialize without whitespace
   - Result: `{"p":"brc-20","op":"bridge","burn":"ORDI","amt":"100"}`

3. **Compute SHA256 Hash**:
   - Convert serialized JSON to bytes: `burn_payload.encode()`
   - Compute hash: `commitment_hash = SHA256(burn_payload.encode()).digest()`
   - Result: 32-byte raw hash (not hex-encoded)

4. **Store Commitment Hash**:
   - The commitment hash is placed in **Witness[0]** of Transaction 1 (32 bytes raw)
   - This hash is also embedded in the vault script as `<commitment_hash>`

**Critical**: The commitment hash must be calculated **exactly** as described. Any variation in JSON serialization (whitespace, field order, etc.) will result in a different hash, causing the vault script validation to fail in Transaction 2.

### Vault Address Derivation

The vault address is **deterministically derived** from publicly available data, allowing the indexer to recalculate and verify the address independently. This ensures trustless verification that the ordinal was sent to the correct vault.

**Complete Derivation Process (5 Steps)**:

**Step 1: Extract Inputs**
- **User pubkey**: Extract 33-byte public key from Input[0] of Transaction 1
- **Commitment hash**: Extract 32-byte hash from Witness[0] of Transaction 1
- **Standardized Bridge tweak**: `SHA256("UniversalBridge")` (32 bytes, constant)

**Step 2: Build Hash Preimage Covenant Script**
- Construct the Bitcoin Script: `OP_SHA256 <commitment_hash> OP_EQUALVERIFY <user_pubkey> OP_CHECKSIG`
- Script bytes: `commitment_hash (32 bytes) + OP_SHA256 (0xa8) + OP_EQUALVERIFY (0x87) + user_pubkey (33 bytes) + OP_CHECKSIG (0xac)`

**Step 3: Create TapLeaf**
- Create a TapLeaf with version `0xc0` and the script from Step 2
- Compute the leaf hash: `leaf_hash = TapLeaf(version=0xc0, script=script).hash()`

**Step 4: Include Standardized Tweak in MAST Root**
- **CRITICAL**: Include the standardized Bridge tweak in the MAST root calculation
- Compute: `mast_root = SHA256(leaf_hash + BRIDGE_TWEAK)`
- This ensures the vault address is **provably linked** to the Bridge protocol
- Without this tweak, the address would be a standard P2TR address, and the indexer would reject the migration

**Step 5: Derive Taproot Output Key and Encode Address**
- Derive NUMS Point (same as OPI-3, using seed `"Bitcoin:W-Protocol:Sovereign-Vault:2025"`)
- Compute Taproot output key: `output_key = compute_taproot_output_key(internal_key=NUMS_POINT, script_tree_hash=mast_root)`
- Encode as Bech32m address: `address = encode_bech32m(output_key, "bc", 1)`

**Result**: A deterministic P2TR address (e.g., `bc1p...`) that can be calculated by anyone with the user pubkey and commitment hash, enabling trustless verification.

**Why This Matters**: Because the vault address is deterministically derived from the commitment hash (which is in Witness[0]), the indexer can **mathematically prove** that the ordinal was sent to the correct address. This prevents address spoofing and ensures the atomic burn-to-mint sequence is properly enforced.

#### Bridge Burn Payload (Transaction 2)

**Witness[1] Payload** (The Preimage):
```json
{
  "p": "brc-20",
  "op": "bridge",
  "burn": "ORDI",
  "amt": "100"
}
```

**Required Fields:**
- `p`: MUST be `"brc-20"`
- `op`: MUST be `"bridge"`
- `burn`: Legacy BRC-20 ticker being burned
- `amt`: Amount being burned (must match Transaction 1)

---

## Indexer Validation Rules

### Bridge Lock Validation (Transaction 1)

1. **OP_RETURN Detection**: Locate OP_RETURN output. Payload MUST contain `"p": "brc-20"`, `"op": "bridge"`, `"tick"`, and `"amt"`.

2. **Commitment Hash Extraction**: Extract 32-byte `commitment_hash` from Witness[0] of Input[0].

3. **Legacy Inscription Validation**: Verify Input[0] contains valid legacy BRC-20 `transfer` inscription matching ticker and amount.

4. **Universal Ticker Virginity**: Ensure Universal ticker (same name) is **virgin** (not deployed).

5. **Vault Address Derivation**: Recalculate vault address from user pubkey, commitment hash, and standardized tweak `SHA256("UniversalBridge")`.

6. **Vault Address Verification**: Verify ordinal sent to correct vault address.

7. **Inscription Ownership**: Verify via Ordinals API that inscription is at vault address.

8. **State Recording**: Record bridge lock. **NOTE**: Universal tokens NOT minted in Transaction 1.

### Bridge Burn Validation (Transaction 2)

1. **Vault UTXO Identification**: Identify if input is known bridge vault UTXO.

2. **Witness Data Parsing**: Extract Burn JSON preimage from Witness[1].

3. **Hash Preimage Validation**: Verify `SHA256(Witness[1])` matches `commitment_hash` from lock record.

4. **Burn JSON Validation**: Parse and validate structure matches lock record.

5. **Inscription Ownership Re-verification**: Verify inscription still at vault address before minting.

6. **Debit-Before-Credit Accounting**:
   - **DEBIT**: Mark lock as burned
   - **VALIDATE**: Ensure Universal ticker is virgin
   - **DEBIT**: Deploy Universal ticker atomically
   - **CREDIT**: Mint Universal tokens to user

7. **Namespace Migration Tracking**: Record the successful migration, linking Legacy and Universal namespaces.

### Protocol Integrity Checks

Indexers SHOULD periodically validate protocol state integrity:

- **No Duplicate Locks**: Each inscription can only be locked once (tracked by inscription hash)
- **Conservation of Mass**: Total Universal tokens minted MUST equal total Legacy ordinals burned
- **Virgin Ticker Enforcement**: Each Universal ticker MUST have been virgin before migration
- **Vault Address Consistency**: All vault addresses MUST use the standardized Bridge tweak (verifiable by recalculating addresses)

---

## Rationale

### Why Namespace Migration Instead of Wrapper?

**Hostile Takeover**: Not a wrapper creating `bORDI`. Instead, `ORDI` on Universal is **virgin** until Legacy `ORDI` is destroyed. Creates **1:1 mapping** preventing ticker conflicts.

**Ticker Identity Preservation**: Ticker remains same across namespaces. No wrapper tokens needed.

### Why Hash Preimage Covenant?

**Cryptographic Atomicity**: Bitcoin Script enforces atomicity. User **cannot** spend vault without revealing exact Burn JSON preimage. Zero trust required.

**Advantages**: Cryptographic guarantee, zero trust, immutability, fee efficiency (~75% savings), deterministic verification.

### Why Standardized Tweak?

**Provable Protocol Link**: Tweak `SHA256("UniversalBridge")` guarantees vault addresses are **provably linked** to Bridge protocol, preventing confusion with standard P2TR addresses.

---

## Data Structures

### ProtocolConstitution

Global protocol configuration (immutable after first deploy), storing:
- `operator_address`: Protocol operator address (for Genesis Fees)
- `genesis_fee_lock_sats`: Optional fee for bridge lock operations (0 if not set)
- `genesis_fee_burn_sats`: Optional fee for bridge burn operations (0 if not set)
- `created_block`: Block when protocol was initialized
- `bridge_tweak`: Standardized tweak `SHA256("UniversalBridge")` (32 bytes)

### BridgeLockRecord

Tracks bridge lock operations (Transaction 1), storing:
- `txid`: Lock transaction ID
- `sender_address`: Address of user initiating lock
- `ticker`: Legacy ticker being migrated (e.g., "ORDI")
- `amount`: Amount from ordinal inscription
- `commitment_hash`: SHA256 hash of Burn JSON (hex-encoded for storage)
- `vault_address`: P2TR address where ordinal is locked (deterministic from user pubkey)
- `vault_utxo`: Outpoint (txid:vout) of vault UTXO
- `inscription_hash`: SHA256 of inscription data (for duplicate detection)
- `block_height`: Block height of lock transaction
- `timestamp`: Timestamp of lock transaction
- `burned`: Boolean flag (false until Transaction 2)
- `burn_txid`: Transaction ID of burn (None until burned)
- `burn_block_height`: Block height of burn (None until burned)

### NamespaceMigrationRecord

Tracks successful namespace migrations (after Transaction 2), storing:
- `legacy_ticker`: Legacy ticker name (e.g., "ORDI")
- `universal_ticker`: Universal ticker name (same as legacy)
- `amount`: Amount migrated
- `lock_txid`: Transaction 1 ID
- `burn_txid`: Transaction 2 ID
- `user_address`: Address receiving Universal tokens
- `migration_block`: Block height of migration
- `timestamp`: Timestamp of migration

## Implementation Constants

### Protocol Constants

**Standardized Bridge Tweak (CRITICAL)**:
- Seed: `"UniversalBridge"`
- Tweak: `SHA256("UniversalBridge")` (32 bytes)
- **Purpose**: Guarantees vault addresses are provably linked to Bridge protocol

**NUMS Point**: Same derivation as OPI-3, using seed `"Bitcoin:W-Protocol:Sovereign-Vault:2025"`.

**Genesis Fees**: Optional fees defined at protocol initialization. No minimum, practical maximum of 1 BTC.

**Migration Limits**: Minimum 1 token, practical maximum of 10^18 tokens (18 decimals).

**Ordinals API Configuration**:
- Configurable endpoint for inscription ownership verification
- Timeout: 10 seconds
- Retry attempts: 3
- Fallback: On-chain verification if API unavailable

## Integration Points

### BRC-20 Legacy Integration

- Extract inscriptions from Input[0] witness data (Ordinals envelope format)
- Validate inscription format matches legacy BRC-20 `transfer` operation
- Track inscription transfer to vault address using Ordinals rules

### Ordinals API Integration (CRITICAL)

**Purpose**: Double-check inscription ownership before minting Universal tokens.

**When**:
- Transaction 1 (Lock): Verify inscription is at vault address
- Transaction 2 (Burn): Verify inscription is still at vault address before minting

**API Endpoint**: `GET /inscription/{inscription_id}/address`

**Validation**: Ensure inscription address matches expected vault address.

**Fallback**: If API unavailable, indexer can use on-chain verification (slower but trustless).

### Universal BRC-20 Integration

- Deploy Universal ticker atomically with mint (same ticker name as Legacy)
- Mint tokens to user address (address-bound state)
- Track Universal token balances

### Taproot Integration

- Derive vault addresses using standardized tweak
- Validate vault addresses belong to Bridge protocol (recalculate and compare)
- Support Hash Preimage Covenant script: `OP_SHA256 <commitment_hash> OP_EQUALVERIFY <user_pubkey> OP_CHECKSIG`
- Parse Witness Data to extract Burn JSON preimage

### Indexer Integration

- Process blocks sequentially
- Validate atomic sequence (Lock → Burn)
- Parse Witness Data to extract Burn JSON preimage
- Bitcoin Script enforces cryptographic validation (preimage must match commitment hash)
- **CRITICAL**: Verify inscription ownership via Ordinals API before minting

## Backwards Compatibility

This OPI is **fully non-breaking**:
- Legacy indexers see inscription locked in Taproot address
- Universal-compliant indexers recognize `bridge` operation and mint when burn validated
- Genesis Fees are optional
- No token deployment required

---

## Reference Implementation

The Simplicity Indexer will implement `bridge` validation logic in:
- **Repo**: [https://github.com/The-Universal-BRC-20-Extension/](https://github.com/The-Universal-BRC-20-Extension/)
- **Branch**: `main`
- **Examples**: `examples/opi-0/` directory

---

## Test Vectors

### Bridge Lock Transaction (Transaction 1)

**OP_RETURN Payload**:
```json
{"p":"brc-20","op":"bridge","tick":"ORDI","amt":"100"}
```

**Input[0]**: Must contain valid legacy BRC-20 `transfer` inscription with ticker "ORDI" and amount "100".

**Witness[0]**: 32-byte commitment hash (raw bytes, not hex-encoded) - SHA256 of Burn JSON.

**Output[2]**: Vault P2TR address (derived from user pubkey + commitment hash + standardized tweak).

### Bridge Burn Transaction (Transaction 2)

**Witness[1] Payload** (The Preimage):
```json
{"p":"brc-20","op":"bridge","burn":"ORDI","amt":"100"}
```

**Input[0]**: Vault UTXO from Transaction 1.

**Output[0]**: Burn address (e.g., Satoshi address) - Ordinal sent here (burned).

**Bitcoin Script Validation**: Script validates `SHA256(Witness[1]) == commitment_hash` before allowing spend.

### Complete Migration Sequence

**Block N**: Bridge Lock
- User locks 100 ORDI in vault
- Vault address: `bc1p...` (derived with standardized tweak)
- State: ORDI locked, not yet migrated

**Block N+1**: Bridge Burn & Reveal
- User spends vault UTXO
- Witness[1] contains Burn JSON preimage
- Bitcoin Script validates hash match
- Indexer validates atomic sequence
- State: Universal ORDI deployed and minted (100 tokens to user)

## Security Considerations

### Attack Vectors and Mitigations

| Attack Vector | Mitigation |
|---------------|------------|
| **Replay attack** | Inscription locked in vault, cannot be reused |
| **Double lock** | Inscription hash tracked, prevents duplicates |
| **Inflation attack** | Debit-Before-Credit ensures validation before mint |
| **Cancel without burn** | Bitcoin Script rejects invalid transactions (cryptographically enforced) |
| **Ticker conflict** | Universal ticker must be virgin before migration |
| **Vault address spoofing** | Standardized tweak ensures provable protocol linkage |
| **False migration claim** | Indexer validates atomic sequence (Lock → Burn) |

### Conservation of Mass

- **Lock**: 1 legacy ordinal → locked in vault (1:1 ratio)
- **Burn & Mint**: 1 legacy ordinal destroyed → 1 Universal ticker minted (1:1 ratio)
- **No inflation**: Universal tokens cannot exceed burned Legacy tokens

### Cryptographic Security

- **Hash Preimage Covenant**: Script `OP_SHA256 <commitment_hash> OP_EQUALVERIFY <user_pubkey> OP_CHECKSIG` provides cryptographic atomicity
- **Standardized Tweak**: Vault addresses provably linked to Bridge protocol
- **Deterministic Derivation**: Vault addresses verifiable off-chain

---

## Conclusion

This OPI-0 establishes the **Bridge Standard** as a namespace migration protocol for migrating legacy BRC-20 tickers into Universal BRC-20. Through **atomic burn-to-mint** operations enforced by Hash Preimage Covenant, it creates a trustless, one-way migration path that preserves ticker identity while transitioning execution models.

---

## References

1. Nakamoto, S. (2008). "Bitcoin: A Peer-to-Peer Electronic Cash System"
2. Domo (2023). "BRC-20: An experimental fungible token standard"
3. Wuille, P. et al. (2021). "BIP-341: Taproot: SegWit version 1 spending rules"
4. Wuille, P. et al. (2021). "BIP-342: Validation of Taproot Scripts"
5. Laz1m0v (2025). "OPI-1: The Swap Standard for Automated Market Makers"
6. Laz1m0v (2025). "OPI-2: The Curve Extension for Trustless Programmatic Token Distribution"
7. Laz1m0v, GaloisField2718 (2025). "OPI-3: The W Standard for State-Bound Wrapped Assets"
8. Laz1m0v (2026). "OPI-4: The Vault Standard for Automated Strategy Pools & Elastic Shares"
