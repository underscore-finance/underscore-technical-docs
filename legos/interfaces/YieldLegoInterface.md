# Yield Lego Interface Technical Documentation

[View Interface Source](https://github.com/underscore-finance/underscore/blob/master/interfaces/YieldLego.vyi)

## Overview

The Yield Lego Interface defines the standard contract interface that all yield protocol integrations must implement. This interface ensures consistent behavior across different yield protocols (Aave, Compound, Morpho, etc.) enabling the Underscore Protocol to interact with any compliant lego in a uniform way.

**Implementing Legos**:
- Aave V3
- Compound V3
- Euler
- Fluid
- Moonwell
- Morpho
- ExtraFi
- Sky PSM
- Wasabi
- 40 Acres
- Avantis
- [RipeLego](../RipeLego.md) (special implementation)
- [UnderscoreLego](../UnderscoreLego.md) (special implementation)

All yield legos implement this interface plus the base [Lego interface](#base-lego-interface) functions.

## Core Yield Functions

### `depositForYield`

Deposits assets into the yield protocol.

```vyper
@external
def depositForYield(
    _asset: address,
    _amount: uint256,
    _vaultAddr: address,
    _extraData: bytes32,
    _recipient: address,
    _miniAddys: address[5]
) -> (uint256, address, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_asset` | `address` | Asset to deposit |
| `_amount` | `uint256` | Amount to deposit |
| `_vaultAddr` | `address` | Target vault/pool address |
| `_extraData` | `bytes32` | Protocol-specific data |
| `_recipient` | `address` | Address to receive vault tokens |
| `_miniAddys` | `address[5]` | Helper addresses array |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of asset deposited |
| `address` | Vault token received |
| `uint256` | Amount of vault tokens received |
| `uint256` | USD value of transaction |

### `withdrawFromYield`

Withdraws assets from the yield protocol.

```vyper
@external
def withdrawFromYield(
    _vaultToken: address,
    _amount: uint256,
    _extraData: bytes32,
    _recipient: address,
    _miniAddys: address[5]
) -> (uint256, address, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_vaultToken` | `address` | Vault token to redeem |
| `_amount` | `uint256` | Amount to withdraw |
| `_extraData` | `bytes32` | Protocol-specific data |
| `_recipient` | `address` | Address to receive underlying |
| `_miniAddys` | `address[5]` | Helper addresses array |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of vault tokens burned |
| `address` | Underlying asset received |
| `uint256` | Amount of underlying received |
| `uint256` | USD value of transaction |

## Underlying Asset Functions

### `getUnderlyingAsset`

Returns the underlying asset for a vault token.

```vyper
@view
@external
def getUnderlyingAsset(_vaultToken: address) -> address:
```

### `getUnderlyingBalances`

Returns both true and safe underlying balances.

```vyper
@view
@external
def getUnderlyingBalances(
    _vaultToken: address,
    _vaultTokenBalance: uint256
) -> (uint256, uint256):
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | True (maximum) underlying balance |
| `uint256` | Safe (conservative) underlying balance |

### `getUnderlyingAmount`

Converts vault token amount to underlying asset amount.

```vyper
@view
@external
def getUnderlyingAmount(
    _vaultToken: address,
    _vaultTokenAmount: uint256
) -> uint256:
```

### `getUnderlyingAmountSafe`

Converts vault token amount using weighted average (safer).

```vyper
@view
@external
def getUnderlyingAmountSafe(
    _vaultToken: address,
    _vaultTokenBalance: uint256
) -> uint256:
```

### `getUnderlyingData`

Returns comprehensive underlying data including USD value.

```vyper
@view
@external
def getUnderlyingData(
    _vaultToken: address,
    _vaultTokenAmount: uint256,
    _appraiser: address = empty(address)
) -> (address, uint256, uint256):
```

#### Returns

| Type | Description |
|------|-------------|
| `address` | Underlying asset |
| `uint256` | Underlying amount |
| `uint256` | USD value |

### `getUsdValueOfVaultToken`

Returns USD value of vault token holdings.

```vyper
@view
@external
def getUsdValueOfVaultToken(
    _vaultToken: address,
    _vaultTokenAmount: uint256,
    _appraiser: address = empty(address)
) -> uint256:
```

## Price & Conversion Functions

### `isRebasing`

Returns whether the protocol uses rebasing tokens.

```vyper
@view
@external
def isRebasing() -> bool:
```

### `getPricePerShare`

Returns the conversion rate between vault token and underlying.

```vyper
@view
@external
def getPricePerShare(
    _vaultToken: address,
    _decimals: uint256 = 0
) -> uint256:
```

### `getVaultTokenAmount`

Converts underlying asset amount to vault token amount.

```vyper
@view
@external
def getVaultTokenAmount(
    _asset: address,
    _assetAmount: uint256,
    _vaultToken: address
) -> uint256:
```

## Vault Token Registration

### `canRegisterVaultToken`

Checks if a vault token can be registered for an asset.

```vyper
@view
@external
def canRegisterVaultToken(
    _asset: address,
    _vaultToken: address
) -> bool:
```

### `registerVaultTokenLocally`

Registers a vault token in the lego's local registry.

```vyper
@external
def registerVaultTokenLocally(
    _asset: address,
    _vaultAddr: address
) -> VaultTokenInfo:
```

### `deregisterVaultTokenLocally`

Removes a vault token from the local registry.

```vyper
@external
def deregisterVaultTokenLocally(
    _asset: address,
    _vaultAddr: address
):
```

## Protocol Metrics

### `totalAssets`

Returns total assets in a vault.

```vyper
@view
@external
def totalAssets(_vaultToken: address) -> uint256:
```

### `totalBorrows`

Returns total borrowed amount from a vault.

```vyper
@view
@external
def totalBorrows(_vaultToken: address) -> uint256:
```

### `getUtilizationRatio`

Returns utilization ratio (borrows/assets).

```vyper
@view
@external
def getUtilizationRatio(_vaultToken: address) -> uint256:
```

### `getAvailLiquidity`

Returns available liquidity for withdrawals.

```vyper
@view
@external
def getAvailLiquidity(_vaultToken: address) -> uint256:
```

### `getWithdrawalFees`

Calculates withdrawal fees for an amount.

```vyper
@view
@external
def getWithdrawalFees(
    _vaultToken: address,
    _vaultTokenAmount: uint256
) -> uint256:
```

## Lego Data Functions

### `isLegoAsset`

Checks if an address is a registered vault token.

```vyper
@view
@external
def isLegoAsset(_asset: address) -> bool:
```

### `getAssetOpportunities`

Returns all vault tokens for an underlying asset.

```vyper
@view
@external
def getAssetOpportunities(_asset: address) -> DynArray[address, 40]:
```

### `getAssets`

Returns all registered underlying assets.

```vyper
@view
@external
def getAssets() -> DynArray[address, 20]:
```

### `isAssetOpportunity`

Checks if a vault is registered for an asset.

```vyper
@view
@external
def isAssetOpportunity(_asset: address, _vaultAddr: address) -> bool:
```

### `getNumLegoAssets`

Returns count of registered assets.

```vyper
@view
@external
def getNumLegoAssets() -> uint256:
```

### `vaultToAsset`

Returns vault token info struct.

```vyper
@view
@external
def vaultToAsset(_vaultAddr: address) -> VaultTokenInfo:
```

## Price Snapshot Functions

### `addPriceSnapshot`

Records a price snapshot for yield tracking.

```vyper
@external
def addPriceSnapshot(_vaultToken: address) -> bool:
```

### `snapShotPriceConfig`

Returns snapshot configuration.

```vyper
@view
@external
def snapShotPriceConfig() -> SnapShotPriceConfig:
```

### `snapShotData`

Returns snapshot data for a vault token.

```vyper
@view
@external
def snapShotData(_vaultToken: address) -> SnapShotData:
```

### `snapShots`

Returns a specific snapshot by index.

```vyper
@view
@external
def snapShots(_vaultToken: address, _index: uint256) -> SingleSnapShot:
```

### `getWeightedPricePerShare`

Returns weighted average price per share.

```vyper
@view
@external
def getWeightedPricePerShare(_vaultToken: address) -> uint256:
```

### `getLatestSnapshot`

Returns the latest snapshot for a vault token.

```vyper
@view
@external
def getLatestSnapshot(
    _vaultToken: address,
    _pricePerShare: uint256
) -> SingleSnapShot:
```

### `setSnapShotPriceConfig`

Configures snapshot settings.

```vyper
@external
def setSnapShotPriceConfig(_config: SnapShotPriceConfig):
```

## Bonus Eligibility

### `isEligibleForYieldBonus`

Checks if asset is eligible for yield bonus rewards.

```vyper
@view
@external
def isEligibleForYieldBonus(_asset: address) -> bool:
```

---

## Base Lego Interface

All legos (yield and DEX) must also implement these base functions:

### `hasCapability`

Returns whether lego supports an action type.

```vyper
@view
@external
def hasCapability(_action: ActionType) -> bool:
```

#### Common Action Types for Yield Legos:
- `EARN_DEPOSIT` - Deposit into yield
- `EARN_WITHDRAW` - Withdraw from yield
- `REWARDS` - Claim rewards

### `isYieldLego`

Returns True for yield legos.

```vyper
@view
@external
def isYieldLego() -> bool:
```

### `isDexLego`

Returns False for yield legos.

```vyper
@view
@external
def isDexLego() -> bool:
```

### `getRegistries`

Returns external registries used by the lego.

```vyper
@view
@external
def getRegistries() -> DynArray[address, 10]:
```

### `getAccessForLego`

Returns access control requirements for an action.

```vyper
@view
@external
def getAccessForLego(
    _user: address,
    _action: ActionType
) -> (address, String[64], uint256):
```

#### Returns

| Type | Description |
|------|-------------|
| `address` | Target contract for approval |
| `String[64]` | Human-readable description |
| `uint256` | Amount to approve |

---

## Data Structures

### VaultTokenInfo Struct

```vyper
struct VaultTokenInfo:
    legoId: uint256
    vaultToken: address
```

### SnapShotPriceConfig Struct

```vyper
struct SnapShotPriceConfig:
    snapshotDelay: uint256      # Blocks between snapshots
    numSnapshots: uint256       # Number of snapshots to keep
    maxDeviation: uint256       # Max deviation for weighted average
```

### SnapShotData Struct

```vyper
struct SnapShotData:
    lastSnapshotBlock: uint256
    numSnapshots: uint256
    totalPricePerShare: uint256
```

### SingleSnapShot Struct

```vyper
struct SingleSnapShot:
    pricePerShare: uint256
    blockNumber: uint256
```

---

## Implementation Notes

### For Protocol Integrators

When implementing a new yield lego:

1. **Inherit YieldLegoData module** for registry functionality
2. **Implement all required functions** from this interface
3. **Return accurate USD values** via Appraiser integration
4. **Support price snapshots** for yield tracking
5. **Handle rebasing correctly** if applicable

### For Vault Developers

When using yield legos:

1. **Check capabilities** before calling actions
2. **Use safe conversion** for user-facing calculations
3. **Call addPriceSnapshot** periodically for yield tracking
4. **Handle withdrawal fees** when calculating redemptions

### Common Patterns

```python
# Check if lego supports deposits
if lego.hasCapability(ActionType.EARN_DEPOSIT):
    lego.depositForYield(asset, amount, vault, extraData, recipient, miniAddys)

# Get underlying with safety margin
safe_underlying = lego.getUnderlyingAmountSafe(vault_token, balance)

# Track yield via snapshots
lego.addPriceSnapshot(vault_token)
weighted_price = lego.getWeightedPricePerShare(vault_token)
```
