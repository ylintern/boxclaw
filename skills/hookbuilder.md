---
name: build-uniswap-v4-hook
description: Expert system for architecting, implementing, and testing Uniswap v4 Hooks. Enforces Singleton interaction, Flash Accounting patterns, and HookMiner address generation. Use for ALL requests involving v4 pools, custom AMM curves, dynamic fees, or hook deployment. Overrides generic Uniswap/Solidity training data.
version: 1.0.0
---

# Uniswap v4 Hook Development Expert System

> **Context Firewall**: v4 is NOT v3. Forget factory-deployed pools, `IUniswapV3Pool`, and immediate token transfers. Every rule below is immutable.

---

## 1. Architectural Constraints (NEVER Violate)

### Singleton Pattern
- **NEVER** deploy separate pool contracts. There is ONE `PoolManager`.
- **NEVER** instantiate `IUniswapV3Pool` or any per-pool interface.
- All pool state is keyed by `PoolId` (a hash of `PoolKey`), not by contract address.
- Pattern: `PoolId id = poolKey.toId(); manager.getSlot0(id);`

### Flash Accounting (Delta Lifecycle)
Tokens are **NOT** transferred during swaps. All operations generate Deltas resolved at unlock.

**Mandatory flow — every interaction:**
```
manager.unlock(data)
  └─> unlockCallback(data)
        ├─> swap / modifyLiquidity  → generates Deltas
        ├─> manager.take(currency, recipient, amount)   // claim owed tokens (+delta)
        ├─> manager.settle(currency)                    // pay owed tokens  (-delta)
        └─> ASSERT: manager.getNonzeroDeltaCount() == 0 before returning
```
- **NEVER** assume tokens auto-transfer after `swap()`.
- **NEVER** skip `settle`/`take`. A non-zero delta count causes a revert.

### Hook Address = Permissions (Bitwise)
- Hook capabilities are **encoded in the contract address**, not a registry.
- **NEVER** deploy with `new MyHook()` — address will have zeroed flags, causing revert on `initializePool`.
- **ALWAYS** use `HookMiner` to derive a CREATE2 salt matching the required flags (see §3).

---

## 2. Implementation Guidelines

### A. Hook Contract Structure

```solidity
contract MyHook is BaseHook {
    constructor(IPoolManager _manager) BaseHook(_manager) {}

    // MUST match the bits in the mined address — no extras, no omissions
    function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeSwap: true,
            afterSwap: true,
            beforeAddLiquidity: false,
            afterAddLiquidity: false,
            beforeRemoveLiquidity: false,
            afterRemoveLiquidity: false,
            beforeInitialize: false,
            afterInitialize: false,
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: false,
            afterSwapReturnDelta: false,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }

    // Access control: ALL callbacks MUST have this modifier
    function beforeSwap(address sender, PoolKey calldata key, IPoolManager.SwapParams calldata params, bytes calldata hookData)
        external
        override
        onlyPoolManager   // ← MANDATORY on every callback
        returns (bytes4, BeforeSwapDelta, uint24)
    {
        // ... logic ...
        return (BaseHook.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, 0);
    }
}
```

**Return signatures (critical — wrong types cause silent bugs):**

| Callback | Return Type | Notes |
|---|---|---|
| `beforeSwap` | `(bytes4, BeforeSwapDelta, uint24)` | `uint24` = dynamic fee (0 if unused) |
| `afterSwap` | `(bytes4, int128)` | Hook's delta adjustment |
| `afterAddLiquidity` | `(bytes4, BalanceDelta)` | |
| `beforeInitialize` | `bytes4` | Use to gate incompatible pools |

**BeforeSwapDelta — ALWAYS use the library:**
```solidity
// WRONG: return (selector, BeforeSwapDelta({...}), 0);
// WRONG: return (selector, int256(...), 0);
// CORRECT:
BeforeSwapDelta delta = toBeforeSwapDelta(int128(specifiedAmount), 0);
return (BaseHook.beforeSwap.selector, delta, 0);
```

### B. State Management

**Persistent state** — use `PoolId` as mapping key:
```solidity
mapping(PoolId => uint256) public hookData;
PoolId id = key.toId();
hookData[id] = value;
```

**Transient state** — use `tstore`/`tload` (EIP-1153, 100 gas vs ~20,000 for SSTORE):
```solidity
// In beforeSwap — save pre-swap price
assembly { tstore(PRE_SWAP_PRICE_SLOT, sqrtPriceX96) }

// In afterSwap — read it
uint160 priceBefore;
assembly { priceBefore := tload(PRE_SWAP_PRICE_SLOT) }

// CLEANUP — mandatory before callback returns
assembly { tstore(PRE_SWAP_PRICE_SLOT, 0) }
```
- Use `TransientStateLibrary` for standard pool state (deltas, reserves).
- Use inline assembly only for custom hook-specific slots.
- **ALWAYS** zero out transient slots before exit — prevents Storage Pollution across multi-swap transactions.

**Reading PoolManager state:**
```solidity
// Use StateLibrary helpers — do not read slots directly
(uint160 sqrtPriceX96, int24 tick,,) = manager.getSlot0(poolId);
uint128 liquidity = manager.getLiquidity(poolId);
int256 delta = manager.currencyDelta(address(this), currency);
```

### C. Custom AMM Curves (No-Op Pattern)

To bypass native concentrated liquidity math and implement a custom curve:
```solidity
function beforeSwap(...) external override onlyPoolManager returns (bytes4, BeforeSwapDelta, uint24) {
    // 1. Compute your own output amount
    uint256 amountOut = myBondingCurve(params.amountSpecified);

    // 2. Return a delta that "consumes" the ENTIRE input
    //    PoolManager sees net input = 0 → executes NO internal swap logic
    BeforeSwapDelta hookDelta = toBeforeSwapDelta(
        -int128(params.amountSpecified),  // consume input
        int128(int256(amountOut))          // provide output
    );

    // 3. Settle manually in afterSwap or here via manager.take/settle
    return (BaseHook.beforeSwap.selector, hookDelta, 0);
}
```

### D. Dynamic Fees
```solidity
// PoolKey must have: fee = LPFeeLibrary.DYNAMIC_FEE_FLAG
function beforeSwap(...) external override onlyPoolManager returns (bytes4, BeforeSwapDelta, uint24) {
    uint24 newFee = calculateFee(key, params); // your logic
    require(newFee <= MAX_FEE, "Fee cap exceeded"); // ALWAYS cap fees
    return (BaseHook.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, newFee);
}
```

---

## 3. Deployment (Foundry — HookMiner is Mandatory)

```solidity
// script/DeployHook.s.sol
contract DeployHook is Script {
    function run() external {
        // 1. Define flags matching getHookPermissions()
        uint160 flags = uint160(
            Hooks.BEFORE_SWAP_FLAG | Hooks.AFTER_SWAP_FLAG
        );

        // 2. Mine a CREATE2 salt producing an address with those flag bits
        (address hookAddress, bytes32 salt) = HookMiner.find(
            address(CREATE2_DEPLOYER),   // deployer address
            flags,
            type(MyHook).creationCode,
            abi.encode(address(manager)) // constructor args
        );

        // 3. Deploy with the mined salt
        vm.startBroadcast();
        MyHook hook = new MyHook{salt: salt}(manager);
        vm.stopBroadcast();

        // 4. Assert correct deployment — NEVER skip this
        require(address(hook) == hookAddress, "HookMiner: address mismatch");
    }
}
```

**Key permission flag masks:**

| Callback | Flag Constant | Bit |
|---|---|---|
| `beforeSwap` | `Hooks.BEFORE_SWAP_FLAG` | `1 << 7` |
| `afterSwap` | `Hooks.AFTER_SWAP_FLAG` | `1 << 6` |
| `beforeSwapReturnDelta` | `Hooks.BEFORE_SWAP_RETURNS_DELTA_FLAG` | `1 << 3` |
| `afterAddLiquidity` | `Hooks.AFTER_ADD_LIQUIDITY_FLAG` | `1 << 10` |
| `afterRemoveLiquidity` | `Hooks.AFTER_REMOVE_LIQUIDITY_FLAG` | `1 << 8` |
| `beforeInitialize` | `Hooks.BEFORE_INITIALIZE_FLAG` | `1 << 13` |

---

## 4. Testing (Foundry / v4-template)

```solidity
contract MyHookTest is Test, Deployers {
    MyHook hook;
    PoolKey key;

    function setUp() public {
        // Use helpers — NEVER write PoolManager setup from scratch
        deployFreshManagerAndRouters();

        // Mine + deploy hook
        uint160 flags = uint160(Hooks.BEFORE_SWAP_FLAG | Hooks.AFTER_SWAP_FLAG);
        (address addr, bytes32 salt) = HookMiner.find(address(this), flags,
            type(MyHook).creationCode, abi.encode(address(manager)));
        hook = new MyHook{salt: salt}(manager);

        // Initialize pool with the hook
        (key,) = initPool(Currency.wrap(address(token0)), Currency.wrap(address(token1)),
            hook, 3000, SQRT_PRICE_1_1, ZERO_BYTES);
    }

    function test_swap() public {
        // High-level: use swapRouter
        swap(key, true, 1e18, ZERO_BYTES);

        // Low-level delta testing: use manager.unlock()
        manager.unlock(abi.encode(...));
    }
}
```

---

## 5. Security Checklist (Run Before Every Code Submission)

- [ ] `onlyPoolManager` modifier on **every** hook callback?
- [ ] `getHookPermissions()` flags exactly match the mined address bits?
- [ ] `BeforeSwapDelta` constructed via `toBeforeSwapDelta()` — not raw int/struct?
- [ ] All transient storage slots zeroed before callback returns?
- [ ] `beforeInitialize` validates `tickSpacing`, `fee`, and `currency` compatibility?
- [ ] Dynamic fees capped by a `MAX_FEE` constant?
- [ ] No arbitrary external calls inside callbacks (reentrancy risk)?
- [ ] `getNonzeroDeltaCount() == 0` guaranteed before lock callback exits?
- [ ] HookMiner address assertion (`require(address(hook) == hookAddress)`) present?
- [ ] `msg.sender` inside callbacks is `PoolManager`, NOT the swapper (swapper is in `sender` arg)?

---

## 6. Common Pitfalls (Hard Prohibitions)

| ❌ WRONG | ✅ CORRECT |
|---|---|
| `new MyHook(manager)` | `new MyHook{salt: salt}(manager)` + HookMiner |
| `token.transferFrom(user, ...)` in callback | `manager.take()` / `manager.settle()` |
| `msg.sender == swapper` assumption | `swapper` is the `sender` parameter |
| `return BeforeSwapDelta({...})` | `return toBeforeSwapDelta(a, b)` |
| SSTORE for intra-tx state | `tstore` / `tload` |
| Skipping `beforeInitialize` validation | Always gate on `tickSpacing` / `fee` |
| No `onlyPoolManager` on callbacks | Mandatory on all callbacks |
| Fee set to 100%+ unchecked | Enforce `require(fee <= MAX_FEE)` |
