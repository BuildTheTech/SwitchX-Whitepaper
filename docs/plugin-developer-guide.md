# SwitchX V4 Plugin Developer Guide

Every advanced feature in SwitchX — adaptive fees, MEV protection, oracle tracking, farming integration, ALM rebalancing, and limit orders — is implemented as a plugin. This same system is open to third-party developers, enabling anyone to build custom pool behaviors without modifying the core AMM engine.

This guide covers everything needed to build, deploy, and attach custom plugins to SwitchX V4 pools.

---

## Table of Contents

1. [Hook Model & Bitmap Configuration](#1-hook-model--bitmap-configuration)
2. [The IV4Plugin Interface](#2-the-iv4plugin-interface)
3. [Plugin Attachment](#3-plugin-attachment)
4. [Abstract Plugin SDK](#4-abstract-plugin-sdk)
5. [Creating a Custom Plugin](#5-creating-a-custom-plugin)
6. [Custom Pool Deployment Path](#6-custom-pool-deployment-path)
7. [Composing Sub-Plugins (Mixin Pattern)](#7-composing-sub-plugins-mixin-pattern)
8. [Development Constraints](#8-development-constraints)
9. [SDK Package Reference](#9-sdk-package-reference)

---

## 1. Hook Model & Bitmap Configuration

Every V4 pool can have **one plugin** attached to it. The plugin receives lifecycle callbacks (hooks) from the pool at well-defined execution points. Which hooks are active is controlled by a **bitmap configuration** (`pluginConfig`, a `uint8`) stored in the pool's global state — each bit enables or disables a specific hook.

**Hook Bitmap Flags:**

| Bit | Flag | Value | Hook Enabled |
|-----|------|-------|-------------|
| 0 | `BEFORE_SWAP_FLAG` | 1 | `beforeSwap` — called before every swap |
| 1 | `AFTER_SWAP_FLAG` | 2 | `afterSwap` — called after every swap |
| 2 | `BEFORE_POSITION_MODIFY_FLAG` | 4 | `beforeModifyPosition` — called before mint/burn |
| 3 | `AFTER_POSITION_MODIFY_FLAG` | 8 | `afterModifyPosition` — called after mint/burn |
| 4 | `BEFORE_FLASH_FLAG` | 16 | `beforeFlash` — called before flash loans |
| 5 | `AFTER_FLASH_FLAG` | 32 | `afterFlash` — called after flash loans |
| 6 | `AFTER_INIT_FLAG` | 64 | `afterInitialize` — called after pool initialization |
| 7 | `DYNAMIC_FEE` | 128 | Enables fee override and plugin fee returns |

The `beforeInitialize` hook is **always called** if a plugin is attached, regardless of the bitmap. This allows plugins to perform one-time setup during pool initialization.

**Hook execution flow (swap example):**

```
User calls pool.swap()
  -> Pool checks pluginConfig bit 0 (BEFORE_SWAP_FLAG)
  -> If set: pool calls plugin.beforeSwap(sender, recipient, zeroToOne, amount, limitPrice, payInAdvance, data)
  -> Plugin returns (selector, feeOverride, pluginFee)
  -> Pool validates returned selector matches IV4Plugin.beforeSwap.selector
  -> Pool executes swap with overridden fee (if DYNAMIC_FEE flag is set)
  -> Pool checks pluginConfig bit 1 (AFTER_SWAP_FLAG)
  -> If set: pool calls plugin.afterSwap(sender, recipient, zeroToOne, amount, limitPrice, amount0, amount1, data)
  -> Plugin returns selector
  -> Pool validates returned selector
```

Every hook must return its own function selector as a safety check. If the returned selector does not match the expected value, the pool reverts with `invalidHookResponse`.

---

## 2. The IV4Plugin Interface

All plugins implement the `IV4Plugin` interface (`@switchx/v4-core`), which defines ten methods:

| Method | Returns | Purpose |
|--------|---------|---------|
| `defaultPluginConfig()` | `uint8` | Declares which hooks the plugin requires |
| `handlePluginFee(amount0, amount1)` | `bytes4` | Receives plugin fee tokens from the pool |
| `beforeInitialize(sender, sqrtPriceX96)` | `bytes4` | Pre-initialization setup |
| `afterInitialize(sender, sqrtPriceX96, tick)` | `bytes4` | Post-initialization setup |
| `beforeModifyPosition(sender, recipient, bottomTick, topTick, liquidityDelta, data)` | `(bytes4, uint24)` | Pre-mint/burn logic; can return a plugin fee |
| `afterModifyPosition(sender, recipient, bottomTick, topTick, liquidityDelta, amount0, amount1, data)` | `bytes4` | Post-mint/burn logic |
| `beforeSwap(sender, recipient, zeroToOne, amountRequired, limitSqrtPrice, withPaymentInAdvance, data)` | `(bytes4, uint24, uint24)` | Pre-swap logic; returns fee override and plugin fee |
| `afterSwap(sender, recipient, zeroToOne, amountRequired, limitSqrtPrice, amount0, amount1, data)` | `bytes4` | Post-swap logic (MEV recapture, limit orders, ALM triggers) |
| `beforeFlash(sender, recipient, amount0, amount1, data)` | `bytes4` | Pre-flash loan logic |
| `afterFlash(sender, recipient, amount0, amount1, paid0, paid1, data)` | `bytes4` | Post-flash loan logic |

**Fee returns in `beforeSwap`:**

The `beforeSwap` hook returns three values: `(selector, feeOverride, pluginFee)`. The `feeOverride` replaces the pool's base fee for this swap. The `pluginFee` is an additional fee accrued to the plugin contract itself (collected via `handlePluginFee`). Both require the `DYNAMIC_FEE` flag (bit 7) to be set — if it is not set and a nonzero value is returned, the pool reverts.

Plugins that calculate dynamic fees should also implement `IV4DynamicFeePlugin`, which exposes `getCurrentFee()` and `getCurrentFeeDirectional(zeroToOne)` as view functions. This allows the Quoter and frontend UIs to read the current fee without executing a swap.

---

## 3. Plugin Attachment

Plugins can be attached to pools through two mechanisms:

**Automatic attachment at creation time:**

When `V4Factory.createPool()` is called, the factory's `defaultPluginFactory` is invoked to deploy and attach a plugin in a single atomic transaction:

```
V4Factory.createPool(token0, token1, data)
  -> Calls defaultPluginFactory.beforeCreatePoolHook(pool, creator, deployer, token0, token1, data)
  -> Plugin factory deploys new plugin contract, returns its address
  -> Pool is deployed via CREATE2 with the plugin address in its constructor
  -> Calls defaultPluginFactory.afterCreatePoolHook(plugin, pool, deployer)
```

**Manual attachment post-creation:**

Pool administrators can call `pool.setPlugin(newPluginAddress)` to replace the current plugin. This flushes any pending plugin fees to the old plugin and resets `pluginConfig` to `0`. The new plugin must then set its desired configuration via `pool.setPluginConfig(newConfig)` (callable by the plugin itself or a pool administrator).

---

## 4. Abstract Plugin SDK

SwitchX provides a published SDK (`@switchx/abstract-plugin`) with base contracts that simplify custom plugin development:

**`AbstractPlugin`** — The foundational non-upgradeable base. Provides:
- `onlyPool` modifier for hook access control
- Default no-op implementations of all ten hooks (returning correct selectors)
- `_updatePluginConfigInPool(uint8)` — writes new config to pool
- `_enablePluginFlags(uint8)` / `_disablePluginFlags(uint8)` — bitwise helpers for toggling individual hooks
- `_getPoolState()` — reads current price, tick, fee, and plugin config
- `collectPluginFee(token, amount, recipient)` — authorized withdrawal of accumulated plugin fees
- `activeModules` tracking for introspection

**`BaseAbstractPlugin`** — Extends `AbstractPlugin` with factory role-based authorization. The `_authorize()` function checks that the caller holds the `SWITCHX_BASE_PLUGIN_MANAGER` role (granted through `V4Factory`).

**`UpgradeableAbstractPlugin`** — A beacon-proxy-compatible variant using OpenZeppelin's `Initializable` pattern. Reads the pool address from proxy bytecode, enabling a single implementation contract to serve multiple pools through lightweight proxies.

---

## 5. Creating a Custom Plugin

### Step 1: Implement the plugin contract

Inherit from `AbstractPlugin` and override only the hooks your logic requires:

```solidity
import '@switchx/abstract-plugin/contracts/AbstractPlugin.sol';

contract MyFeePlugin is AbstractPlugin {
    uint24 public customFee;

    constructor(address _pool, address _factory)
        AbstractPlugin(_pool, _factory) {}

    function _authorize() internal view override {
        require(msg.sender == pluginFactory, "Unauthorized");
    }

    function defaultPluginConfig() external pure override returns (uint8) {
        return uint8(
            Plugins.BEFORE_SWAP_FLAG |   // Enable beforeSwap hook
            Plugins.DYNAMIC_FEE          // Enable fee override
        );
    }

    function beforeSwap(
        address, address, bool, int256, uint160, bool, bytes calldata
    ) external override onlyPool returns (bytes4, uint24, uint24) {
        return (IV4Plugin.beforeSwap.selector, customFee, 0);
    }
}
```

Key rules:
- Every overridden hook **must** return its own function selector (e.g., `IV4Plugin.beforeSwap.selector`)
- Set the `DYNAMIC_FEE` flag (bit 7) if your plugin returns nonzero `feeOverride` or `pluginFee`
- Use the `onlyPool` modifier on all hook functions to prevent unauthorized calls
- Call `_updatePluginConfigInPool()` from your `afterInitialize` hook to sync the config bitmap

### Step 2: Create a plugin factory

Implement `IV4PluginFactory` to deploy your plugin when new pools are created:

```solidity
import '@switchx/v4-core/contracts/interfaces/plugin/IV4PluginFactory.sol';

contract MyPluginFactory is IV4PluginFactory {
    address public immutable v4Factory;
    mapping(address => address) public pluginByPool;

    constructor(address _v4Factory) {
        v4Factory = _v4Factory;
    }

    function beforeCreatePoolHook(
        address pool, address, address, address, address, bytes calldata
    ) external override returns (address) {
        require(msg.sender == v4Factory, "Only V4Factory");
        address plugin = address(new MyFeePlugin(pool, address(this)));
        pluginByPool[pool] = plugin;
        return plugin;
    }

    function afterCreatePoolHook(address, address, address) external view override {
        require(msg.sender == v4Factory, "Only V4Factory");
    }
}
```

### Step 3: Register the factory

The V4Factory owner calls `setDefaultPluginFactory(myPluginFactory)` to make it the default for all new pools. Alternatively, use the custom pool path (Section 6) to deploy pools with your plugin without changing the global default.

### Annotated Example Contracts

The repository includes fully compilable, step-by-step annotated example contracts in `src/abstract-plugin/contracts/examples/`:

| Contract | Description |
|----------|-------------|
| `ExampleDynamicFeePlugin.sol` | Complete plugin implementation demonstrating hook overrides, fee logic, config bitmap management, and the `IV4DynamicFeePlugin` view interface |
| `ExamplePluginFactory.sol` | Matching factory showing both automatic attachment (via `defaultPluginFactory`) and retroactive attachment for existing pools |

These are designed as copy-and-modify templates.

---

## 6. Custom Pool Deployment Path

For developers who want to deploy pools with custom plugins without changing the protocol-wide default, SwitchX provides `V4CustomPoolEntryPoint` and `AbstractCustomPluginFactory`:

```
Developer's CustomFactory.createCustomPool(creator, tokenA, tokenB, data)
  -> CustomFactory calls V4CustomPoolEntryPoint.createCustomPool(deployer=self, ...)
  -> EntryPoint calls V4Factory.createCustomPool(deployer, ...)
  -> V4Factory calls EntryPoint.beforeCreatePoolHook(pool, ...)
  -> EntryPoint forwards to CustomFactory.beforeCreatePoolHook(pool, ...)
  -> CustomFactory deploys plugin and returns address
  -> Pool is created with custom plugin attached
```

Inherit from `AbstractCustomPluginFactory` and implement `_createPlugin(pool)`:

```solidity
import '@switchx/abstract-plugin/contracts/AbstractCustomPluginFactory.sol';

contract MyCustomFactory is AbstractCustomPluginFactory {
    constructor(address _entryPoint) AbstractCustomPluginFactory(_entryPoint) {}

    function _createPlugin(address pool) internal override returns (address) {
        return address(new MyFeePlugin(pool, address(this)));
    }
}
```

The `V4CustomPoolEntryPoint` must hold the `CUSTOM_POOL_DEPLOYER` role on V4Factory. It also provides permissioned pool admin methods (`setTickSpacing`, `setPlugin`, `setPluginConfig`, `setFee`) that only the deploying factory can call for its pools.

---

## 7. Composing Sub-Plugins (Mixin Pattern)

SwitchX's production plugin (`SwitchXBasePlugin`) demonstrates a composition pattern where multiple independent sub-plugins are combined into a single contract via Solidity multiple inheritance. Each sub-plugin provides internal helper methods, and the final contract orchestrates them in each hook:

```
SwitchXBasePlugin
  +-- DynamicFeePlugin         -> _getCurrentFee()
  +-- VolatilityOraclePlugin   -> _writeTimepoint(), _getLastTick()
  +-- BackrunFeePlugin         -> _applyBackrunFee()
  +-- CrossDexOracleFeePlugin  -> _applyCrossDexOracleFee()
  +-- FeeDiscountPlugin        -> _applyFeeDiscount()
  +-- FarmingProxyPlugin       -> _updateVirtualPoolTick()
  +-- LimitOrderPlugin         -> _updateLimitOrderManagerState()
  +-- SecurityPlugin           -> _checkStatus()
  +-- AlmPlugin                -> _obtainTWAPAndRebalance()
  +-- MevBackrunPlugin         -> _isMevInternalSwap(), _tryMevAfterSwap()
```

To build a composite plugin:

1. Create abstract sub-plugin contracts, each providing internal methods prefixed with `_`
2. Your final plugin inherits all sub-plugins
3. Override hooks to call the sub-plugin methods in the desired execution order
4. Set `defaultPluginConfig` as a constant combining all required flags

For complex sub-plugin logic requiring separate implementation contracts, the SDK provides the `BaseConnector` contract for delegatecall-based composition. This allows a plugin to delegate specific hook logic to external implementation contracts while sharing the plugin's storage context.

---

## 8. Development Constraints

- **One plugin per pool**: Each pool supports exactly one attached plugin. Complex behavior must be composed within a single plugin contract.
- **Gas sensitivity**: Hooks execute inline with swaps and other pool operations. Expensive hook logic directly impacts user gas costs. Keep hooks lean.
- **Selector validation**: The pool validates every hook's return selector. A wrong or missing selector reverts the entire pool operation.
- **Dynamic fee gate**: Returning nonzero `feeOverride` or `pluginFee` without the `DYNAMIC_FEE` flag set causes a hard revert.
- **Plugin fee collection**: Plugin fees accumulate in the pool contract until `handlePluginFee` is called. Use `collectPluginFee()` (from `AbstractPlugin`) to withdraw.
- **Config self-management**: Plugins can call `pool.setPluginConfig()` on themselves to dynamically enable or disable hooks at runtime. Use `_enablePluginFlags()` and `_disablePluginFlags()` for safe bitwise updates.

---

## 9. SDK Package Reference

| Package | Contents |
|---------|----------|
| `@switchx/abstract-plugin` | `AbstractPlugin`, `BaseAbstractPlugin`, `UpgradeableAbstractPlugin`, `AbstractCustomPluginFactory`, `BaseConnector` |
| `@switchx/v4-core` | `IV4Plugin`, `IV4DynamicFeePlugin`, `IV4PluginFactory`, `Plugins` library |
| `@switchx/v4-periphery` | `V4CustomPoolEntryPoint`, `IV4CustomPoolEntryPoint` |

### Source Contracts

| Contract | Path |
|----------|------|
| AbstractPlugin | `src/abstract-plugin/contracts/AbstractPlugin.sol` |
| BaseAbstractPlugin | `src/abstract-plugin/contracts/BaseAbstractPlugin.sol` |
| UpgradeableAbstractPlugin | `src/abstract-plugin/contracts/UpgradableAbstractPlugin.sol` |
| AbstractCustomPluginFactory | `src/abstract-plugin/contracts/AbstractCustomPluginFactory.sol` |
| BaseConnector | `src/abstract-plugin/contracts/BaseConnector.sol` |
| IV4Plugin | `src/core/contracts/interfaces/plugin/IV4Plugin.sol` |
| IV4DynamicFeePlugin | `src/core/contracts/interfaces/plugin/IV4DynamicFeePlugin.sol` |
| IV4PluginFactory | `src/core/contracts/interfaces/plugin/IV4PluginFactory.sol` |
| Plugins | `src/core/contracts/libraries/Plugins.sol` |
| V4CustomPoolEntryPoint | `src/periphery/contracts/V4CustomPoolEntryPoint.sol` |
| ExampleDynamicFeePlugin | `src/abstract-plugin/contracts/examples/ExampleDynamicFeePlugin.sol` |
| ExamplePluginFactory | `src/abstract-plugin/contracts/examples/ExamplePluginFactory.sol` |
