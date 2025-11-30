# Appraiser Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/core/Appraiser.vy)

## Overview

Appraiser is the price oracle and valuation engine for the Underscore Protocol. As a Department contract, it provides accurate USD value calculations for all assets including normal tokens and yield-bearing vaults, handles yield profit calculations with performance fee support, and integrates with RipePriceDesk for reliable price data.

**Core Features**:
- **RipePriceDesk Integration**: Delegates all pricing to Ripe Protocol's price oracle
- **Yield Asset Support**: Specialized handling for vault tokens with underlying conversion
- **Profit Calculation**: Tracks yield profits for both rebasing and non-rebasing assets
- **VaultRegistry Integration**: Adds price snapshots for earn vault assets
- **USD Valuation**: Comprehensive USD value calculations for portfolio management

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                               Appraiser                                 |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                       Price Flow Architecture                      | |
|  |                                                                   | |
|  |  Normal Assets:                                                   | |
|  |    1. Query RipePriceDesk.getUsdValue()                          | |
|  |    2. Return USD value                                            | |
|  |                                                                   | |
|  |  Yield Assets:                                                     | |
|  |    1. Get underlying amount via YieldLego.getUnderlyingAmount()  | |
|  |    2. Query RipePriceDesk with underlying asset                   | |
|  |    3. Return USD value                                            | |
|  |                                                                   | |
|  |  Earn Vault Assets:                                               | |
|  |    1. Get USD value as above                                      | |
|  |    2. Add price snapshot to RipePriceDesk                        | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                     Yield Profit Calculation                       | |
|  |                                                                   | |
|  |  Rebasing Assets:                                                  | |
|  |    • Profit = Current Balance - Last Balance                      | |
|  |    • Apply max yield increase cap if configured                   | |
|  |    • Return profit amount and performance fee                     | |
|  |                                                                   | |
|  |  Non-Rebasing Vaults:                                              | |
|  |    • Track price per share changes                                | |
|  |    • Calculate profit on tracked balance                          | |
|  |    • Convert to vault tokens for withdrawal                       | |
|  |    • Apply max yield cap and performance fee                      | |
|  +-------------------------------------------------------------------+ |
+-------------------------------------------------------------------------+
```

## Module Integration

Appraiser implements the Department interface and integrates:
- `Addys` module for address registry management
- `DeptBasics` module for pause functionality

## Data Structures

### VaultToken (from Ledger)

```vyper
struct VaultToken:
    legoId: uint256           # ID of lego managing this token
    underlyingAsset: address  # Base asset (if yield token)
    decimals: uint256         # Token decimals
    isRebasing: bool          # Whether token is rebasing
```

### AssetUsdValueConfig (from MissionControl)

```vyper
struct AssetUsdValueConfig:
    legoId: uint256           # Lego ID from Ledger
    legoAddr: address         # Resolved lego address
    isYieldAsset: bool        # True if has underlying
    underlyingAsset: address  # Underlying asset address
```

### ProfitCalcConfig (from MissionControl)

```vyper
struct ProfitCalcConfig:
    legoId: uint256           # Lego ID from Ledger
    legoAddr: address         # Resolved lego address
    isYieldAsset: bool        # True if has underlying
    underlyingAsset: address  # Underlying asset address
    maxYieldIncrease: uint256 # Max yield increase threshold
    performanceFee: uint256   # Performance fee (basis points)
    isRebasing: bool          # Whether token is rebasing
    decimals: uint256         # Token decimals
```

## Constants

```vyper
HUNDRED_PERCENT: constant(uint256) = 100_00  # 100.00%
RIPE_PRICE_DESK_ID: constant(uint256) = 7    # Registry ID for RipePriceDesk
```

## Immutable Variables

```vyper
RIPE_HQ: immutable(address)  # Ripe protocol registry address
```

## Constructor

### `__init__`

Initializes Appraiser with registry addresses.

```vyper
@deploy
def __init__(
    _undyHq: address,
    _ripeHq: address,
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyHq` | `address` | UndyHq registry contract address |
| `_ripeHq` | `address` | RipeHq registry contract address |

## Yield Profit Functions

### `calculateYieldProfits`

Calculates yield profits for a position.

```vyper
@external
def calculateYieldProfits(
    _asset: address,
    _currentBalance: uint256,
    _lastBalance: uint256,
    _lastPricePerShare: uint256,
    _missionControl: address,
    _legoBook: address,
) -> (uint256, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_asset` | `address` | Yield asset address |
| `_currentBalance` | `uint256` | Current balance of asset |
| `_lastBalance` | `uint256` | Previous balance of asset |
| `_lastPricePerShare` | `uint256` | Previous price per share |
| `_missionControl` | `address` | MissionControl address (optional) |
| `_legoBook` | `address` | LegoBook address (optional) |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | New price per share (0 for rebasing) |
| `uint256` | Profit amount in asset tokens |
| `uint256` | Performance fee percentage |

#### Access

Can only be called by user wallets.

#### Behavior

1. Gets ProfitCalcConfig from MissionControl
2. Returns (0, 0, 0) for non-yield assets
3. For rebasing: calculates profit as balance increase
4. For non-rebasing: calculates profit based on price per share change
5. Applies max yield increase cap if configured

### `calculateYieldProfitsNoUpdate`

View function version of profit calculation.

```vyper
@view
@external
def calculateYieldProfitsNoUpdate(
    _legoId: uint256,
    _asset: address,
    _underlyingAsset: address,
    _currentBalance: uint256,
    _lastBalance: uint256,
    _lastPricePerShare: uint256,
) -> (uint256, uint256, uint256):
```

### `lastPricePerShare`

Gets current price per share for an asset.

```vyper
@view
@external
def lastPricePerShare(_asset: address) -> uint256:
```

#### Behavior

1. Gets VaultToken data from Ledger
2. Returns 0 if not a yield asset
3. Gets lego address from LegoBook
4. Returns price per share from YieldLego

## USD Value Functions

### `getUsdValue`

Gets USD value for an asset amount.

```vyper
@view
@external
def getUsdValue(
    _asset: address,
    _amount: uint256,
    _missionControl: address = empty(address),
    _legoBook: address = empty(address),
    _ledger: address = empty(address),
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_asset` | `address` | Asset address |
| `_amount` | `uint256` | Amount of asset |
| `_missionControl` | `address` | Optional MissionControl |
| `_legoBook` | `address` | Optional LegoBook |
| `_ledger` | `address` | Optional Ledger |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | USD value of the amount |

### `getUnderlyingUsdValue`

Gets USD value for an underlying asset amount directly from Ripe.

```vyper
@view
@external
def getUnderlyingUsdValue(_asset: address, _amount: uint256) -> uint256:
```

### `updatePriceAndGetUsdValue`

Updates price snapshot and returns USD value.

```vyper
@external
def updatePriceAndGetUsdValue(
    _asset: address,
    _amount: uint256,
    _missionControl: address = empty(address),
    _legoBook: address = empty(address),
) -> uint256:
```

#### Access

- User wallets
- Valid Undy addresses
- Registered backpack items

#### Behavior

1. Gets USD value
2. If asset is an earn vault (not leverage vault), adds price snapshot to RipePriceDesk
3. Returns USD value

### `updatePriceAndGetUsdValueAndIsYieldAsset`

Updates price and returns USD value with yield asset flag.

```vyper
@external
def updatePriceAndGetUsdValueAndIsYieldAsset(
    _asset: address,
    _amount: uint256,
    _missionControl: address = empty(address),
    _legoBook: address = empty(address),
) -> (uint256, bool):
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | USD value |
| `bool` | Whether asset is yield-bearing |

## Ripe Integration Functions

### `getRipePrice`

Gets price from RipePriceDesk.

```vyper
@view
@external
def getRipePrice(_asset: address) -> uint256:
```

### `getAssetAmountFromRipe`

Gets asset amount for a given USD value.

```vyper
@view
@external
def getAssetAmountFromRipe(_asset: address, _usdValue: uint256) -> uint256:
```

Used by LootDistributor to convert USD values to RIPE token amounts for bonuses.

## Internal Functions

### `_getUsdValueAndIsYieldAsset`

Core internal function for USD value calculation.

```vyper
@view
@internal
def _getUsdValueAndIsYieldAsset(
    _asset: address,
    _amount: uint256,
    _ledger: address,
    _missionControl: address,
    _legoBook: address,
) -> (uint256, bool, address):
```

#### Behavior

1. Gets AssetUsdValueConfig from MissionControl
2. If yield asset with underlying:
   - Converts vault tokens to underlying via YieldLego
   - Gets USD value of underlying from RipePriceDesk
3. Otherwise:
   - Gets USD value directly from RipePriceDesk
4. Returns (usdValue, isYieldAsset, ripePriceDesk)

### `_handleRebaseYieldAsset`

Calculates profits for rebasing yield assets.

```vyper
@view
@internal
def _handleRebaseYieldAsset(
    _currentBalance: uint256,
    _lastBalance: uint256,
    _maxYieldIncrease: uint256,
    _performanceFee: uint256,
) -> (uint256, uint256, uint256):
```

#### Formula

```
Profit = Current Balance - Last Balance
If max cap: Profit = min(Profit, Last Balance × maxYieldIncrease / 10000)
Return (0, actualProfit, performanceFee)
```

### `_handleNormalYieldAsset`

Calculates profits for non-rebasing vault assets.

```vyper
@view
@internal
def _handleNormalYieldAsset(
    _currentBalance: uint256,
    _lastBalance: uint256,
    _lastPricePerShare: uint256,
    _currentPricePerShare: uint256,
    _config: ProfitCalcConfig,
) -> (uint256, uint256, uint256):
```

#### Formula

```
Tracked Balance = min(currentBalance, lastBalance)
Prev Underlying = trackedBalance × lastPPS / decimals
Current Underlying = trackedBalance × currentPPS / decimals
Profit (underlying) = currentUnderlying - prevUnderlying
If max cap: Profit = min(Profit, prevUnderlying × maxYieldIncrease / 10000)
Profit (vault tokens) = profitInUnderlying × decimals / currentPPS
Return (currentPPS, profitInVaultTokens, performanceFee)
```

### `_updateRipeSnapshot`

Adds price snapshot to RipePriceDesk.

```vyper
@internal
def _updateRipeSnapshot(_asset: address):
```

Called when updating prices for earn vault assets.

## Security Considerations

### Access Control
- User wallet validation for state-changing functions
- Backpack item permissions for pricing updates
- Valid Undy address checks

### Price Security
- All prices delegated to RipePriceDesk
- VaultRegistry check for earn vault snapshot updates
- Graceful failure on pause

### Yield Security
- Maximum yield increase caps
- Performance fee deduction
- Tracked balance for dilution protection (uses min of current/last balance)

## Common Integration Patterns

### Basic Price Queries
```python
# Get USD value
usd_value = appraiser.getUsdValue(
    token.address,
    amount
)

# Get Ripe price
price = appraiser.getRipePrice(token.address)
```

### Yield Profit Tracking
```python
# Track yield profits
new_pps, profit, fee = appraiser.calculateYieldProfits(
    vault.address,
    current_balance,
    last_balance,
    last_pps,
    mission_control,
    lego_book
)

# Calculate fee amount
fee_amount = profit * fee // 10000
```

### Price Updates for Valuation
```python
# Update and get USD value (adds snapshot for earn vaults)
usd_value = appraiser.updatePriceAndGetUsdValue(
    asset.address,
    balance
)

# Check if yield asset
usd_value, is_yield = appraiser.updatePriceAndGetUsdValueAndIsYieldAsset(
    asset.address,
    balance
)
```

### RIPE Token Conversion
```python
# Convert USD value to RIPE amount (used by LootDistributor)
ripe_amount = appraiser.getAssetAmountFromRipe(
    ripe_token.address,
    usd_value
)
```
