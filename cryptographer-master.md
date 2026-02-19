---
name: cryptographer-master
version: 1.0.0
role: Cryptographic operations, key management, signature schemes, MPC, and secure protocol design
requires: SKILL.md, skills/api-key-vault.md
thinking: high
classification: SECURITY — deployed for key ops, signature verification, protocol audits
---

# 🔑 Cryptographer Master — Role Skill

> Cryptography is the foundation UniClaw is built on.
> Every wallet, every signature, every secret ultimately rests here.
> This role handles the math correctly or it doesn't handle it at all.

---

## Role Objective

Answer: **"Is this cryptographic operation correct, secure, and safe to execute?"**

Deployed for:
- Generating and verifying transaction signatures
- Designing or auditing key management schemes (HD wallets, MPC setups)
- Reviewing smart contract cryptography (signature verification, hash functions)
- Implementing or verifying zero-knowledge proofs in DeFi context
- Advising on secure multi-party computation setups for Sensei's operations

---

## Elliptic Curves Reference

### secp256k1 — Bitcoin & Ethereum standard

```
Equation:    y² = x³ + 7  (over Fp)
Prime p:     2²⁵⁶ − 2³² − 977
Order n:     FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
Generator G: (Gx, Gy) — 64-byte compressed point
Key size:    256-bit private key → 512-bit public key (uncompressed), 264-bit (compressed)

Used by:     Bitcoin, Ethereum, most EVM chains
Ops:         ECDSA signing, ECDH key exchange
Why:         Efficient scalar multiplication; no patent issues at time of adoption

Caution:
  - k MUST be unique and random per signature — reuse leaks private key
  - RFC 6979 deterministic k generation eliminates this risk — always use it
  - Curve has no cofactor (h=1) — no cofactor attacks
```

### secp256r1 (P-256, prime256v1) — NIST standard

```
Equation:    y² = x³ − 3x + b  (over Fp)
Prime p:     2²⁵⁶ − 2²²⁴ + 2¹⁹² + 2⁹⁶ − 1
Key size:    256-bit

Used by:     TLS/HTTPS, Apple Secure Enclave, Passkeys (WebAuthn), FIDO2
Why UniClaw: Hardware security keys (YubiKey, iPhone Secure Element) use P-256
             Account Abstraction (ERC-4337) increasingly supports P-256 verification

Caution:
  - NIST involvement raises theoretical backdoor concerns (Dual EC DRBG precedent)
  - No practical attack known; theoretical only
  - For hardware wallet / passkey integration: required curve
```

### Curve25519 / Ed25519 — Modern standard

```
Curve25519 (Montgomery form):   y² = x³ + 486662x² + x  (mod 2²⁵⁵ − 19)
Ed25519 (twisted Edwards form): −x² + y² = 1 − (121665/121666)x²y²

Key size:    256-bit private → 256-bit public
Signature:  EdDSA (Ed25519) — deterministic, no random k needed
Speed:      ~2× faster than secp256k1 for signing
Security:   Constant-time implementations easier → side-channel resistant

Used by:     Solana (ed25519), Cardano, Polkadot, SSH, TLS 1.3, Signal Protocol
             Cosmos / IBC chains, Near Protocol, Aptos, Sui

UniClaw note:
  - Solana positions require Ed25519 signing
  - Cross-chain operations may need both secp256k1 (EVM) and Ed25519 (Solana)
  - Ed25519 is FIPS 186-5 approved (2023) — regulatory safe

Cofactor h=8: small subgroup attacks possible if not validated
  → Always validate point is on prime-order subgroup before use
```

### BN254 (alt-bn128) — ZK-proof standard

```
Pairing-friendly curve used in zk-SNARKs
Used by:    Ethereum precompiles (ecAdd, ecMul, ecPairing at 0x06-0x08)
            Groth16, PLONK proof systems
            Aztec, zkSync Era, Polygon zkEVM

Key property: Efficient bilinear pairings → enables zkSNARK verification on-chain
Security:    ~100-bit (weaker than 128-bit targets) — fine for current use, watch post-quantum

UniClaw note: When auditing ZK-LP or privacy protocols, this is the curve.
```

### BLS12-381 — Ethereum 2.0 / consensus standard

```
Used by:    Ethereum validator signatures (BLS aggregation)
            Filecoin, Zcash Sapling, EIP-2537

Key property: BLS signature aggregation — N signatures → 1 signature
              Critical for Ethereum beacon chain efficiency

UniClaw note: Relevant when monitoring validator performance or
              MEV/PBS (Proposer-Builder Separation) dynamics
```

---

## Signature Schemes

### ECDSA — Ethereum Transactions

```python
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives import hashes
import secrets

# CORRECT: RFC 6979 deterministic k (use a library, never roll your own k)
def sign_transaction_hash(private_key_bytes: bytes, tx_hash: bytes) -> tuple[int, int, int]:
    """
    Sign a 32-byte Ethereum transaction hash.
    Returns (v, r, s) — Ethereum signature components.

    NEVER implement k generation yourself. Use eth_account or web3.py.
    """
    from eth_account._utils.signing import sign_message_hash
    from eth_keys import keys

    private_key = keys.PrivateKey(private_key_bytes)
    signature = private_key.sign_msg_hash(tx_hash)
    return signature.v, int.from_bytes(signature.r, 'big'), int.from_bytes(signature.s, 'big')

def verify_ecdsa(public_key_bytes: bytes, message_hash: bytes, r: int, s: int) -> bool:
    """Verify an ECDSA signature. Returns True if valid."""
    from eth_keys import keys
    pub_key = keys.PublicKey(public_key_bytes)
    signature = keys.Signature(vrs=(0, r, s))
    try:
        return pub_key == signature.recover_public_key_from_msg_hash(message_hash)
    except Exception:
        return False

# SIGNATURE MALLEABILITY: Ethereum enforces s <= n/2 (EIP-2)
# Always check: if s > n//2: s = n - s, v = 1 - v
N = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
def normalize_signature(v: int, r: int, s: int) -> tuple[int, int, int]:
    if s > N // 2:
        s = N - s
        v = 1 - v
    return v, r, s
```

### EdDSA (Ed25519) — Solana / Cross-chain

```python
from cryptography.hazmat.primitives.asymmetric.ed25519 import (
    Ed25519PrivateKey, Ed25519PublicKey
)

def sign_ed25519(private_key_bytes: bytes, message: bytes) -> bytes:
    """Sign arbitrary message with Ed25519. Deterministic — no random k."""
    private_key = Ed25519PrivateKey.from_private_bytes(private_key_bytes)
    return private_key.sign(message)

def verify_ed25519(public_key_bytes: bytes, message: bytes, signature: bytes) -> bool:
    """Verify Ed25519 signature."""
    public_key = Ed25519PublicKey.from_public_bytes(public_key_bytes)
    try:
        public_key.verify(signature, message)
        return True
    except Exception:
        return False

# Ed25519 security notes:
# - Deterministic: same key + message = same signature (unlike ECDSA with bad k)
# - Batch verification available: verify N signatures in ~1.7× single sig time
# - No cofactor issue in standard Ed25519 (uses clamping)
```

### EIP-712 — Structured Data Signing (DeFi standard)

```python
from eth_abi import encode
import eth_account

def build_eip712_domain(name: str, version: str, chain_id: int, contract: str) -> dict:
    return {
        "name": name,
        "version": version,
        "chainId": chain_id,
        "verifyingContract": contract
    }

def sign_typed_data(private_key: str, domain: dict, types: dict, message: dict) -> str:
    """
    EIP-712 structured signing — used by Uniswap permits, 1inch, most DeFi protocols.
    Returns hex signature.
    """
    structured_data = {
        "types": types,
        "domain": domain,
        "primaryType": list(types.keys())[-1],
        "message": message
    }
    signed = eth_account.sign_typed_data(private_key, full_message=structured_data)
    return signed.signature.hex()

# UniClaw uses EIP-712 for:
# - ERC-2612 permit() — gasless token approvals (no approve tx needed)
# - Uniswap V3 position manager signed messages
# - 1inch fusion orders
```

---

## HD Wallet Derivation (BIP-32/39/44)

```python
from bip_utils import (
    Bip39MnemonicGenerator, Bip39SeedGenerator,
    Bip44, Bip44Coins, Bip44Changes
)

# BIP-44 path: m / purpose' / coin_type' / account' / change / address_index
DERIVATION_PATHS = {
    "ethereum":  "m/44'/60'/0'/0/",    # coin_type 60 = ETH
    "solana":    "m/44'/501'/0'/0/",   # coin_type 501 = SOL
    "bitcoin":   "m/44'/0'/0'/0/",     # coin_type 0 = BTC
    "polygon":   "m/44'/60'/0'/0/",    # same as ETH (EVM)
}

# UniClaw wallet hierarchy
UNICLAW_ACCOUNTS = {
    0: "sensei_primary",     # read-only monitoring
    1: "operations",         # LP, fees, rebalancing
    2: "arb",                # isolated arbitrage risk
    3: "test",               # testnet operations
}

def derive_address(mnemonic: str, account: int, index: int, coin: str = "ethereum") -> dict:
    """
    Derive a wallet address from mnemonic.
    Returns: {address, private_key, public_key} — handle with api-key-vault.
    """
    seed = Bip39SeedGenerator(mnemonic).Generate()
    bip44_ctx = Bip44.FromSeed(seed, Bip44Coins.ETHEREUM)
    account_ctx = bip44_ctx.Purpose().Coin().Account(account).Change(Bip44Changes.CHAIN_EXT)
    address_ctx = account_ctx.AddressIndex(index)

    return {
        "address": address_ctx.PublicKey().ToAddress(),
        "public_key": address_ctx.PublicKey().RawCompressed().ToHex(),
        # private_key: never return this — store in vault immediately
    }
```

---

## Multi-Party Computation (MPC)

MPC allows multiple parties to jointly compute a function (e.g. sign a transaction)
without any single party learning the other parties' inputs.

### Why MPC for UniClaw

```
Traditional: Single private key → single point of failure
             Stolen key = total loss

MPC-TSS (Threshold Signature Scheme):
  - Key is SPLIT into N shares across N parties
  - T-of-N parties must cooperate to sign (e.g. 2-of-3)
  - No single party ever holds the full private key
  - No single compromise = no theft
```

### MPC Schemes UniClaw Should Know

```
GG20 / GG21 (Gennaro-Goldfeder):
  - Most widely deployed for EVM chains
  - 2-of-3 or 3-of-5 threshold ECDSA on secp256k1
  - Used by: Fireblocks, Web3Auth, Privy, Safe{AA}
  - Weakness: complex protocol, implementation bugs are common
  - Audit requirement: use audited library only (tss-lib by bnb-chain)

FROST (Flexible Round-Optimized Schnorr Threshold):
  - Threshold Schnorr signatures
  - Simpler than GG20, fewer rounds (2 vs 3+)
  - Used by: newer Cosmos chains, Bitcoin Taproot MPC setups
  - Ed25519 variant: FROST-Ed25519

Shamir Secret Sharing (SSS) — simpler but weaker:
  - Key reconstruction requires T-of-N shares in one place
  - Reconstructed key is exposed at signing time → not true MPC
  - Use only for backup/recovery, never for live signing
```

### MPC Operational Rules for UniClaw

```
Rule 1: Live signing uses GG20 2-of-3 minimum
  Parties: [UniClaw agent] + [Sensei hardware key] + [cold backup]

Rule 2: Key generation ceremony
  □ All parties online simultaneously
  □ Verify all commitment values before proceeding
  □ Record key generation timestamp and party identities
  □ Never regenerate shares without all parties present

Rule 3: Share storage
  Party 1 (UniClaw): encrypted in api-key-vault (class CRITICAL)
  Party 2 (Sensei):  hardware security key (YubiKey or Ledger)
  Party 3 (backup):  encrypted cold storage, offline

Rule 4: Signing threshold
  Routine ops (< $10,000):    UniClaw + Sensei (2-of-3)
  Large moves (> $10,000):    UniClaw + Sensei + verbal confirmation
  Emergency recovery:          All 3 parties required
```

---

## Hash Functions

```python
# Ethereum uses keccak256 — NOT SHA3-256 (different padding)
from eth_hash.auto import keccak

def keccak256(data: bytes) -> bytes:
    return keccak(data)

def keccak256_hex(data: bytes) -> str:
    return "0x" + keccak(data).hex()

# Function selector (first 4 bytes of keccak256 of function signature)
def function_selector(signature: str) -> bytes:
    # e.g. "transfer(address,uint256)"
    return keccak(signature.encode())[:4]

# Common hash usage in DeFi
HASH_USES = {
    "keccak256":  "EVM storage, event topics, CREATE2 addresses, EIP-712",
    "sha256":     "Bitcoin, IPFS CIDs, Merkle tree leaves (some protocols)",
    "poseidon":   "ZK-SNARK friendly hash — Aztec, Tornado Cash, zkSync",
    "pedersen":   "StarkNet commitments, older ZK circuits",
    "blake2b":    "Substrate/Polkadot, Zcash, performance-critical paths",
    "blake3":     "Modern high-performance: 10× faster than SHA3",
}
```

---

## Symmetric Encryption (for secrets at rest)

```python
# ChaCha20-Poly1305 — ZeroClaw's enc2 scheme — preferred
from cryptography.hazmat.primitives.ciphers.aead import ChaCha20Poly1305
import secrets

def encrypt_secret(key: bytes, plaintext: bytes) -> bytes:
    """
    Encrypt with ChaCha20-Poly1305.
    key: 32 bytes
    Returns: nonce (12 bytes) + ciphertext + tag (16 bytes)
    """
    assert len(key) == 32, "ChaCha20 requires 32-byte key"
    chacha = ChaCha20Poly1305(key)
    nonce = secrets.token_bytes(12)
    ciphertext = chacha.encrypt(nonce, plaintext, None)
    return nonce + ciphertext

def decrypt_secret(key: bytes, ciphertext_with_nonce: bytes) -> bytes:
    chacha = ChaCha20Poly1305(key)
    nonce = ciphertext_with_nonce[:12]
    ciphertext = ciphertext_with_nonce[12:]
    return chacha.decrypt(nonce, ciphertext, None)

# AES-256-GCM — alternative for hardware-accelerated environments
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

def encrypt_aes_gcm(key: bytes, plaintext: bytes) -> bytes:
    """AES-256-GCM encryption. key must be 32 bytes."""
    assert len(key) == 32
    aesgcm = AESGCM(key)
    nonce = secrets.token_bytes(12)
    return nonce + aesgcm.encrypt(nonce, plaintext, None)

# When to use which:
# ChaCha20-Poly1305: software environments, mobile, ZeroClaw secrets
# AES-256-GCM:       hardware with AES-NI acceleration (cloud servers, x86)
# NEVER: AES-CBC without MAC, AES-ECB, RC4, DES, 3DES
```

---

## Key Derivation Functions

```python
from cryptography.hazmat.primitives.kdf.scrypt import Scrypt
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from cryptography.hazmat.primitives import hashes
import secrets

# scrypt — Ethereum keystore v3 standard
def derive_key_scrypt(password: str, salt: bytes = None) -> tuple[bytes, bytes]:
    """Derives 32-byte encryption key from password. Used for keystore files."""
    if salt is None:
        salt = secrets.token_bytes(32)
    kdf = Scrypt(salt=salt, length=32, n=2**18, r=8, p=1)
    key = kdf.derive(password.encode())
    return key, salt
    # n=2^18 (high security) — use 2^14 for faster test environments

# HKDF — derive multiple keys from one master secret
def derive_subkey(master_secret: bytes, info: bytes, length: int = 32) -> bytes:
    """
    Derive a purpose-specific subkey from master secret.
    info: context string e.g. b"uniclaw-dune-api-key"
    """
    hkdf = HKDF(
        algorithm=hashes.SHA256(),
        length=length,
        salt=None,
        info=info
    )
    return hkdf.derive(master_secret)

# Argon2id — modern password hashing (better than bcrypt for GPU resistance)
# Use for: wallet password hashing, user authentication
# Library: argon2-cffi
from argon2 import PasswordHasher
ph = PasswordHasher(time_cost=3, memory_cost=65536, parallelism=4)
```

---

## Smart Contract Cryptography Audit Checklist

When reviewing Solidity code for cryptographic issues:

```
SIGNATURE VERIFICATION
  □ Is ecrecover return value checked for address(0)?
     (ecrecover returns 0 on invalid signature — must revert if 0)
  □ Is signature replay prevented? (nonce, chainId, contract address in signed data)
  □ Is EIP-712 used for structured data? (prevents cross-function replay)
  □ Is SignatureChecker.isValidSignatureNow() used? (handles ERC-1271 too)
  □ Is ECDSA.recover() from OpenZeppelin used? (not raw ecrecover)

RANDOMNESS
  □ Is block.timestamp or block.number used as randomness? → INSECURE
  □ Is blockhash used? → INSECURE (miners/validators can manipulate)
  □ Is Chainlink VRF used for on-chain randomness? → CORRECT

HASH FUNCTIONS
  □ Is keccak256 used correctly? (not sha256 where keccak is expected)
  □ Are hash preimage attacks possible? (hash of user-controlled input)
  □ Is abi.encodePacked with multiple variable-length args used?
     → Use abi.encode instead to prevent hash collision attacks

MERKLE PROOFS
  □ Is OpenZeppelin MerkleProof used? (audited, handles edge cases)
  □ Is leaf double-hashing used? (prevents second-preimage attack)
  □ Is proof length validated? (DoS via enormous proof)
```

---

## Commitment Schemes

```
HASH COMMITMENTS (Pedersen / keccak)
  commit(value, nonce) = keccak256(abi.encode(value, nonce))
  Used in: Dutch auctions, sealed bid systems, price oracles
  Security: computationally binding, perfectly hiding (with random nonce)

PEDERSEN COMMITMENTS (homomorphic — ZK-friendly)
  C = r*G + v*H  (G, H are independent generators)
  Property: C(a) + C(b) = C(a+b) — add commitments without revealing values
  Used in: Bulletproofs, confidential transactions, Aztec

KZG COMMITMENTS (polynomial — EIP-4844 / danksharding)
  Used in:  Ethereum blob transactions, zkRollup data availability
  Property: O(1) proof size regardless of polynomial degree
```

---

## Post-Quantum Awareness

```
Current EVM chains:    secp256k1 ECDSA — vulnerable to quantum (Shor's algorithm)
Timeline estimate:     5–15 years for cryptographically relevant quantum computers
Ethereum mitigation:   EIP-7642 and related proposals for PQ migration paths
                       Account Abstraction (ERC-4337) enables quantum-safe wallets today

UniClaw watchlist:
  □ CRYSTALS-Kyber (ML-KEM) — NIST PQC standard for key encapsulation
  □ CRYSTALS-Dilithium (ML-DSA) — NIST PQC standard for signatures
  □ SPHINCS+ (SLH-DSA) — hash-based, conservative choice

Action now: Keep private keys in cold storage — long-term offline = less quantum exposure.
```

---

## What to Report to UniClaw

```
CRYPTOGRAPHER REPORT
═════════════════════
Operation:    [Sign | Verify | Audit | Key Gen | MPC Setup]
Scheme:       [ECDSA-secp256k1 | Ed25519 | EIP-712 | BLS | etc.]
Status:       [PASS | FAIL | REVIEW NEEDED]

Findings:
  [One finding per line — specific, not generic]
  Example:
    ✅ Signature normalized (s <= n/2) — EIP-2 compliant
    ✅ RFC 6979 deterministic k used — no k reuse risk
    ⚠️  ecrecover zero-address check missing in contract line 142
    ❌ block.timestamp used as randomness source — insecure

Recommendation:
  [One clear sentence on what action is needed]

Risk Level:   [LOW | MEDIUM | HIGH | CRITICAL]
Awaiting:     [Sensei approval | No action needed]
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial — full cryptography role for UniClaw | Sensei request 2026-02-18 | 2026-02-18 |
