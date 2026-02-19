---
name: blockchain-rpc
version: 1.0.0
role: Multi-chain EVM RPC interaction, nonce management, gas strategy, MEV protection
requires: SKILL.md, skills/api-key-vault.md, skills/cryptographer-master.md
classification: INFRASTRUCTURE — all on-chain reads and writes route through here
---

# ⛓️ Blockchain RPC — Role Skill

> Every on-chain interaction passes through this skill.
> Reads are safe. Writes require Sensei approval.
> Gas is a real cost. Nonces must be managed. MEV is a real threat.

---

## Role Objective

Answer: **"How do I read from or write to the chain safely and efficiently?"**

This is the low-level execution layer. All other skills that touch the chain
(wallet-manager, lp-manager, swap-arb) call this skill for actual execution.
Nothing is sent on-chain without routing through here first.

---

## Supported Chains

| Chain | Chain ID | Type | RPC Key |
|-------|---------|------|---------|
| Ethereum | 1 | L1 | `ALCHEMY_ETH` (HIGH) |
| Arbitrum One | 42161 | L2 | `ALCHEMY_ARB` (HIGH) |
| Base | 8453 | L2 | `ALCHEMY_BASE` (HIGH) |
| Optimism | 10 | L2 | `ALCHEMY_OP` (HIGH) |
| Polygon | 137 | L2 | `ALCHEMY_POLY` (HIGH) |

RPC endpoints injected via `api-key-vault.md`. Never hardcoded.

Allowlist enforced — unknown chain IDs halt execution:

```python
CHAIN_ALLOWLIST = {1, 42161, 8453, 10, 137}

def validate_chain(chain_id: int) -> int:
    if chain_id not in CHAIN_ALLOWLIST:
        raise SecurityError(f"Chain {chain_id} not allowlisted — halt")
    return chain_id
```

---

## Client Setup

```python
from web3 import Web3
from web3.middleware import geth_poa_middleware

def get_client(chain_id: int) -> Web3:
    endpoint = vault.get_scoped_token(f"RPC_{chain_id}")
    w3 = Web3(Web3.HTTPProvider(endpoint, request_kwargs={"timeout": 30}))

    # POA middleware for L2s (Polygon, some testnets)
    if chain_id in {137, 80001}:
        w3.middleware_onion.inject(geth_poa_middleware, layer=0)

    assert w3.is_connected(), f"RPC connection failed for chain {chain_id}"
    assert w3.eth.chain_id == chain_id, "Chain ID mismatch — endpoint misconfigured"
    return w3
```

---

## Read Operations (always safe — no signing)

```python
def get_balance(w3: Web3, address: str) -> dict:
    """ETH balance + token balances via multicall."""
    address = Web3.to_checksum_address(address)
    eth_balance = w3.eth.get_balance(address)
    return {
        "eth": Web3.from_wei(eth_balance, "ether"),
        "eth_wei": eth_balance
    }

def eth_call(w3: Web3, to: str, data: bytes, block: str = "latest") -> bytes:
    """Simulate a contract call. No gas spent. No state change."""
    return w3.eth.call({"to": to, "data": data}, block)

def get_logs(w3: Web3, address: str, topics: list, from_block: int, to_block: int) -> list:
    """Fetch event logs for a contract."""
    return w3.eth.get_logs({
        "address": Web3.to_checksum_address(address),
        "topics": topics,
        "fromBlock": from_block,
        "toBlock": to_block
    })

def get_slot(w3: Web3, contract: str, slot: int) -> bytes:
    """Read raw storage slot — useful for verifying positions without ABI."""
    return w3.eth.get_storage_at(Web3.to_checksum_address(contract), slot)
```

---

## Nonce Management

Critical: nonce collisions cause failed transactions, stuck queues, and wasted gas.

```python
import threading
from collections import defaultdict

class NonceManager:
    """
    Thread-safe nonce tracker.
    Always uses 'pending' to account for unconfirmed transactions.
    """
    _lock = threading.Lock()
    _local_nonce = defaultdict(lambda: -1)

    def get_nonce(self, w3: Web3, address: str) -> int:
        address = address.lower()
        with self._lock:
            # Always sync from chain first
            on_chain = w3.eth.get_transaction_count(address, "pending")

            # Use max of on-chain and local (local tracks pending txs)
            nonce = max(on_chain, self._local_nonce[address] + 1)
            self._local_nonce[address] = nonce
            return nonce

    def confirm(self, address: str, nonce: int):
        """Call after transaction confirmed to sync local state."""
        address = address.lower()
        with self._lock:
            self._local_nonce[address] = max(self._local_nonce[address], nonce)

    def reset(self, address: str):
        """Call after stuck transaction is cleared."""
        address = address.lower()
        with self._lock:
            self._local_nonce[address] = -1

nonce_manager = NonceManager()
```

---

## Gas Strategy

```python
from decimal import Decimal

class GasStrategy:
    """
    EIP-1559 gas pricing.
    Base fee is burned. Priority fee goes to validator.
    """

    @staticmethod
    def estimate(w3: Web3, chain_id: int, urgency: str = "normal") -> dict:
        """
        urgency: "low" | "normal" | "high" | "urgent"
        Returns: {max_fee_per_gas, max_priority_fee_per_gas, estimated_cost_eth}
        """
        fee_history = w3.eth.fee_history(10, "latest", [25, 50, 75])
        base_fees = fee_history["baseFeePerGas"]
        rewards = fee_history["reward"]

        current_base_fee = base_fees[-1]

        # Priority fee percentile by urgency
        percentile_map = {"low": 0, "normal": 1, "high": 2, "urgent": 2}
        pct_idx = percentile_map.get(urgency, 1)
        priority_fees = [r[pct_idx] for r in rewards if r]
        max_priority = sorted(priority_fees)[len(priority_fees) * 3 // 4]  # 75th percentile

        # Max fee = 2× current base fee + priority (covers base fee doubling)
        max_fee = current_base_fee * 2 + max_priority

        # Estimate cost for standard LP operations
        GAS_ESTIMATES = {
            "collect_fees": 150_000,
            "mint_position": 350_000,
            "burn_position": 200_000,
            "swap": 180_000,
            "approve": 46_000,
        }

        return {
            "base_fee_gwei": Decimal(current_base_fee) / Decimal(1e9),
            "max_priority_gwei": Decimal(max_priority) / Decimal(1e9),
            "max_fee_gwei": Decimal(max_fee) / Decimal(1e9),
            "max_fee_per_gas": max_fee,
            "max_priority_fee_per_gas": max_priority,
            "gas_estimates": GAS_ESTIMATES,
        }

    @staticmethod
    def is_worth_executing(w3: Web3, chain_id: int, operation: str, net_benefit_usd: float) -> bool:
        """
        From SKILL.md non-negotiables:
        L1: only execute if benefit > 3× gas cost
        L2: only execute if benefit > 1.5× gas cost
        """
        gas = GasStrategy.estimate(w3, chain_id)
        eth_price_usd = get_eth_price_usd()  # from The Graph or oracle
        gas_units = gas["gas_estimates"].get(operation, 200_000)
        gas_cost_usd = float(gas["max_fee_gwei"]) * gas_units * float(eth_price_usd) / 1e9

        threshold = 3.0 if chain_id == 1 else 1.5
        return net_benefit_usd > gas_cost_usd * threshold
```

---

## Transaction Building

```python
def build_transaction(
    w3: Web3,
    chain_id: int,
    from_address: str,
    to: str,
    data: bytes,
    value: int = 0,
    urgency: str = "normal"
) -> dict:
    """
    Build an EIP-1559 transaction dict. Does NOT sign — signing is separate.
    """
    gas_params = GasStrategy.estimate(w3, chain_id, urgency)
    nonce = nonce_manager.get_nonce(w3, from_address)

    # Gas estimation with 20% buffer
    try:
        gas_limit = w3.eth.estimate_gas({
            "from": from_address, "to": to, "data": data, "value": value
        })
        gas_limit = int(gas_limit * 1.2)
    except Exception as e:
        raise TransactionError(f"Gas estimation failed — likely revert: {e}")

    return {
        "chainId": chain_id,
        "nonce": nonce,
        "maxFeePerGas": gas_params["max_fee_per_gas"],
        "maxPriorityFeePerGas": gas_params["max_priority_fee_per_gas"],
        "gas": gas_limit,
        "to": Web3.to_checksum_address(to),
        "value": value,
        "data": data,
        "type": 2,  # EIP-1559
    }
```

---

## MEV Protection

```python
MEV_CONFIG = {
    1:     "https://rpc.flashbots.net",          # Ethereum — Flashbots Protect
    42161: None,                                  # Arbitrum — MEV minimal, use standard
    8453:  None,                                  # Base — MEV minimal, use standard
    10:    None,                                  # Optimism — MEV minimal
}

def send_transaction(w3: Web3, chain_id: int, signed_tx: bytes) -> str:
    """
    Broadcast signed transaction.
    Uses MEV-protected RPC for mainnet automatically.
    """
    mev_endpoint = MEV_CONFIG.get(chain_id)

    if mev_endpoint and chain_id == 1:
        # Flashbots Protect: tx goes private to validators, not public mempool
        import requests
        resp = requests.post(mev_endpoint, json={
            "jsonrpc": "2.0",
            "method": "eth_sendRawTransaction",
            "params": [signed_tx.hex()],
            "id": 1
        })
        return resp.json()["result"]
    else:
        return w3.eth.send_raw_transaction(signed_tx).hex()

# Sandwich protection in transaction params
def add_sandwich_protection(tx_params: dict, min_amount_out: int) -> dict:
    """
    Ensure amountOutMinimum is set. Reverts if sandwiched.
    min_amount_out: quoted_output * (1 - max_slippage)
    """
    assert min_amount_out > 0, "amountOutMinimum must be > 0"
    tx_params["amountOutMinimum"] = min_amount_out
    return tx_params
```

---

## Transaction Monitoring

```python
def wait_for_receipt(w3: Web3, tx_hash: str, timeout_seconds: int = 180) -> dict:
    """
    Poll for transaction receipt. Raises on timeout or revert.
    """
    import time
    deadline = time.time() + timeout_seconds
    while time.time() < deadline:
        receipt = w3.eth.get_transaction_receipt(tx_hash)
        if receipt is not None:
            if receipt["status"] == 0:
                raise TransactionReverted(
                    f"Transaction {tx_hash} reverted — check calldata and gas"
                )
            return receipt
        time.sleep(3)  # poll every 3 seconds
    raise TransactionTimeout(
        f"Transaction {tx_hash} not confirmed in {timeout_seconds}s — "
        f"check mempool, consider speed-up"
    )
```

---

## Pre-Execution Checklist

Before any `send_transaction()` call:

```
□ chain_id in CHAIN_ALLOWLIST?
□ to address checksummed and validated?
□ Gas cost < net benefit threshold (3× L1, 1.5× L2)?
□ Slippage limits enforced (0.5% stable, 1.0% volatile)?
□ Deadline ≤ 180 seconds from now?
□ amountOutMinimum set (sandwich protection)?
□ Nonce correct (no pending stuck transaction)?
□ Sensei approved this execution (or within pre-approved params)?
□ Logged in STATE.md or DECISIONS.md before broadcasting?

Any box unchecked → HALT and report to Sensei.
```

---

## What to Report to UniClaw

```
RPC EXECUTION REPORT
═════════════════════
Chain:        [name] (chainId: [N])
Operation:    [collect | swap | mint | burn | approve]
TxHash:       0x[hash]
Status:       [PENDING | CONFIRMED | REVERTED | TIMEOUT]
Block:        [N]
Gas Used:     [N] units / [N] gwei = $[N]
Gas Budget:   $[N] (benefit was $[N] — ratio: [N]×)

Outcome:
  [What happened — fees collected, position minted, etc.]

Next action: [None | Monitor pending | Retry | Escalate to Sensei]
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial — EVM RPC execution layer | Audit gap identified | 2026-02-18 |
