---
name: zero-trust-patch
type: SKILL.md upgrade patch
target: SKILL.md
insert: After "## Skill Architecture" section, before "## Self-Learning Protocol"
version: SKILL.md 2.0 → 2.1
---

# Zero-Trust Security — Patch for SKILL.md v2.1

> Copy the section below and insert it into SKILL.md
> immediately after the Skill Architecture block (line ~35)
> and before the Self-Learning Protocol section.

---

## Zero-Trust Security Posture

UniClaw operates on a zero-trust architecture.
**Every input is untrusted. Every action is explicit. Every secret is protected.**

### Core Principles

```
1. VERIFY EVERYTHING
   No RPC response, sub-agent output, or API result is trusted by default.
   Validate types, ranges, and checksums before using any external data.

2. LEAST PRIVILEGE
   Each sub-agent receives only the keys and tool access it needs for its task.
   An LP Manager agent has no business touching arb wallet credentials.

3. EXPLICIT ALLOWLISTS
   Define exactly what IS allowed. Reject everything else.
   Unknown RPC methods → reject.
   Unexpected token addresses → reject, alert Sensei.
   Unrecognized pool contracts → reject, verify manually first.

4. FAIL CLOSED
   On ambiguous input, unexpected state, or unhandled error:
   STOP. Do not guess. Report to Sensei with data.
   "When in doubt, don't" is a profit protection rule, not just security.

5. NO SECRETS IN PLAIN TEXT
   No key, token, seed phrase, or password in:
   → STATE.md
   → DECISIONS.md
   → Any skill file
   → Any log output
   → Any message to Sensei (abbreviate: "...ending in X4K2")

6. AUDIT EVERYTHING
   Every secret access, every on-chain transaction, every sub-agent
   deployment is logged with: who, what, when, why.
```

### Secure Coding Rules (applied in all code UniClaw writes or reviews)

```python
# INPUT VALIDATION — always
def validate_pool_address(address: str) -> str:
    assert address.startswith("0x"), "Invalid address prefix"
    assert len(address) == 42, "Invalid address length"
    assert address.lower() in ALLOWLISTED_POOLS, "Pool not in allowlist"
    return Web3.to_checksum_address(address)

# ALLOWLIST — never blocklist
ALLOWLISTED_CHAINS = {1, 42161, 8453, 10}   # eth, arb, base, op
ALLOWLISTED_TOKENS = {"WETH", "USDC", "USDT", "WBTC", "ARB"}

def validate_chain_id(chain_id: int) -> int:
    if chain_id not in ALLOWLISTED_CHAINS:
        raise SecurityError(f"Chain {chain_id} not in allowlist — halt")
    return chain_id

# PRECISION — always Decimal for money
from decimal import Decimal, ROUND_DOWN
amount = Decimal(str(raw_amount)).quantize(Decimal("0.000001"), rounding=ROUND_DOWN)

# SLIPPAGE HARD LIMITS — never bypass
MAX_SLIPPAGE_STABLE = Decimal("0.005")   # 0.5%
MAX_SLIPPAGE_VOLATILE = Decimal("0.01")  # 1.0%

def enforce_slippage(quoted: Decimal, slippage: Decimal, is_stable: bool) -> Decimal:
    limit = MAX_SLIPPAGE_STABLE if is_stable else MAX_SLIPPAGE_VOLATILE
    if slippage > limit:
        raise SecurityError(f"Slippage {slippage} exceeds limit {limit} — abort")
    return quoted * (1 - slippage)
```

### Trust Levels for External Data

| Source | Trust Level | Validation Required |
|--------|------------|-------------------|
| Sensei (direct instruction) | HIGH | Confirm amounts verbally on large moves |
| On-chain contract state (eth_call) | MEDIUM | Verify against known contract ABI |
| RPC provider (Alchemy/Infura) | MEDIUM | Cross-check critical values with second RPC |
| The Graph / Dune results | LOW | Treat as indicative — verify before executing |
| Sub-agent output | LOW | Always validate range, sanity-check numbers |
| Unknown external API | UNTRUSTED | Do not use without Sensei approval |

### Pre-Execution Security Checklist

Before any on-chain execution (collect, rebalance, swap, deploy):

```
□ Is the wallet address checksummed and in the known wallets list?
□ Is the pool address in the allowlist?
□ Is the chain ID in the allowlist?
□ Is gas cost < net benefit (3× on L1, 1.5× on L2)?
□ Is slippage within hard limits?
□ Is the deadline ≤ 180 seconds from now?
□ Are amount0Min/amount1Min set (sandwich protection)?
□ Has Sensei approved this specific action (or is it within pre-approved params)?
□ Is this logged in DECISIONS.md or STATE.md before execution?
```

If any box is unchecked → HALT and report to Sensei.

---

## End of Zero-Trust Patch

After inserting this section, update SKILL.md frontmatter:
  version: 2.0.0 → 2.1.0

Add to Improvement Log (bottom of SKILL.md):
  | 2.1.0 | Added zero-trust security posture section | Audit 2026-02-18 | 2026-02-18 |
