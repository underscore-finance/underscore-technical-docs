# LevgVaultHelper Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore/blob/master/contracts/vaults/LevgVaultHelper.vy)

## Overview

LevgVaultHelper is a stateless helper contract that provides calculation and validation functions for [LevgVault](LevgVault.md). It handles complex calculations involving Ripe Protocol integration, total assets computation for both USDC and non-USDC vaults, debt ratio enforcement, and post-swap slippage validation.

**Core Responsibilities**:
- **Total Assets Calculation**: Computes vault NAV accounting for collateral, leverage, and debt
- **Debt Ratio Enforcement**: Calculates maximum safe borrow amounts
- **Swap Validation**: Validates GREEN/USDC swap slippage
- **Collateral Queries**: Retrieves collateral balances from Ripe Protocol

This contract is called by LevgVault/LevgVaultWallet for all complex calculations that require external protocol queries.

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                          LevgVaultHelper                                 |
|                    (Stateless Calculation Contract)                     |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Total Assets Calculation                       | |
|  |                                                                   | |
|  |  USDC Vault:                                                      | |
|  |    Total = Collateral + Leverage - Debt (all in USDC)            | |
|  |                                                                   | |
|  |  Non-USDC Vault (WETH/CBBTC):                                    | |
|  |    Total = Collateral + (Leverage - Debt) / Price                | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Debt Ratio Enforcement                         | |
|  |                                                                   | |
|  |  maxBorrowAmount = (netUserCapital * maxDebtRatio) - currentDebt | |
|  |                                                                   | |
|  |  Ensures: currentDebt + newBorrow <= netUserCapital * maxRatio   | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Ripe Protocol Integration                      | |
|  |                                                                   | |
|  |  Price Desk     -->  USD value conversions                        | |
|  |  Vault Book     -->  Collateral balance queries                   | |
|  |  Credit Engine  -->  Debt amount queries                          | |
|  |  Deleverage     -->  Vault token validation                       | |
|  +-------------------------------------------------------------------+ |
+-------------------------------------------------------------------------+
```

## State Variables (Immutable)

```vyper
RIPE_REGISTRY: public(immutable(address))
# Ripe Protocol registry for component lookups

GREEN_TOKEN: public(immutable(address))
# Ripe's stablecoin (GREEN)

SAVINGS_GREEN: public(immutable(address))
# Yield-bearing GREEN token

USDC: public(immutable(address))
# USDC token address
```

## Constants

```vyper
RIPE_MISSION_CONTROL_ID: constant(uint256) = 5
RIPE_PRICE_DESK_ID: constant(uint256) = 7
RIPE_VAULT_BOOK_ID: constant(uint256) = 8
RIPE_CREDIT_ENGINE_ID: constant(uint256) = 13
RIPE_DELEVERAGE_ID: constant(uint256) = 18
HUNDRED_PERCENT: constant(uint256) = 100_00  # 10,000 basis points
```

## Total Assets Functions

### `getTotalAssetsForUsdcVault`

Calculates total assets for USDC-denominated leveraged vaults.

```vyper
@view
@external
def getTotalAssetsForUsdcVault(
    _wallet: address,
    _collateralVaultToken: address,
    _collateralVaultTokenLegoId: uint256,
    _collateralVaultTokenRipeVaultId: uint256,
    _leverageVaultToken: address,
    _leverageVaultTokenLegoId: uint256,
    _leverageVaultTokenRipeVaultId: uint256,
    _shouldGetMax: bool,
    _legoBook: address
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_wallet` | `address` | Vault wallet address |
| `_collateralVaultToken` | `address` | Collateral vault token |
| `_collateralVaultTokenLegoId` | `uint256` | Lego ID for collateral |
| `_collateralVaultTokenRipeVaultId` | `uint256` | Ripe vault ID for collateral |
| `_leverageVaultToken` | `address` | Leverage vault token |
| `_leverageVaultTokenLegoId` | `uint256` | Lego ID for leverage |
| `_leverageVaultTokenRipeVaultId` | `uint256` | Ripe vault ID for leverage |
| `_shouldGetMax` | `bool` | True for max, False for safe |
| `_legoBook` | `address` | LegoBook registry |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Total assets in USDC terms |

#### Calculation

```
Total Assets = CollateralUnderlying + LeverageUnderlying + GREENBalance - Debt

Where:
- CollateralUnderlying = vault token balance converted to USDC
- LeverageUnderlying = vault token balance converted to USDC
- GREENBalance = direct GREEN + SavingsGREEN converted to GREEN
- Debt = current debt from Ripe Credit Engine
```

### `getTotalAssetsForNonUsdcVault`

Calculates total assets for non-USDC vaults (WETH, CBBTC).

```vyper
@view
@external
def getTotalAssetsForNonUsdcVault(
    _wallet: address,
    _underlyingAsset: address,
    _collateralVaultToken: address,
    _collateralVaultTokenLegoId: uint256,
    _collateralVaultTokenRipeVaultId: uint256,
    _leverageVaultToken: address,
    _leverageVaultTokenLegoId: uint256,
    _leverageVaultTokenRipeVaultId: uint256,
    _shouldGetMax: bool,
    _legoBook: address
) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Total assets in underlying asset terms |

#### Calculation

```
Total Assets = CollateralUnderlying + NetGREENValueInUnderlying

Where:
- CollateralUnderlying = vault token balance converted to underlying (WETH/CBBTC)
- LeverageUnderlying = vault token balance converted to USDC
- GREENBalance = direct GREEN + SavingsGREEN
- Debt = current debt from Ripe
- NetGREEN = LeverageUnderlying + GREENBalance - Debt
- NetGREENValueInUnderlying = NetGREEN converted to underlying via price
```

## Debt Ratio Functions

### `getMaxBorrowAmount`

Calculates maximum amount that can be safely borrowed.

```vyper
@view
@external
def getMaxBorrowAmount(
    _wallet: address,
    _underlyingAsset: address,
    _collateralVaultToken: address,
    _collateralVaultTokenLegoId: uint256,
    _collateralVaultTokenRipeVaultId: uint256,
    _netUserCapital: uint256,
    _maxDebtRatio: uint256,
    _isUsdcVault: bool,
    _legoBook: address
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_wallet` | `address` | Vault wallet address |
| `_underlyingAsset` | `address` | Underlying asset |
| `_collateralVaultToken` | `address` | Collateral vault token |
| `_collateralVaultTokenLegoId` | `uint256` | Lego ID for collateral |
| `_collateralVaultTokenRipeVaultId` | `uint256` | Ripe vault ID |
| `_netUserCapital` | `uint256` | Net user capital (deposits - withdrawals) |
| `_maxDebtRatio` | `uint256` | Max debt ratio in basis points |
| `_isUsdcVault` | `bool` | Whether vault is USDC-denominated |
| `_legoBook` | `address` | LegoBook registry |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Maximum safe borrow amount |

#### Calculation

```
maxDebt = netUserCapital * maxDebtRatio / HUNDRED_PERCENT
currentDebt = getUserDebtAmount()
maxBorrow = maxDebt - currentDebt (if maxDebt > currentDebt, else 0)
```

#### Example

With:
- `netUserCapital` = 1000 USDC
- `maxDebtRatio` = 30000 (300%)
- `currentDebt` = 2000 USDC

```
maxDebt = 1000 * 30000 / 10000 = 3000 USDC
maxBorrow = 3000 - 2000 = 1000 USDC
```

## Swap Validation Functions

### `getSwappableUsdcAmount`

Calculates maximum USDC that can be swapped while respecting debt ratio.

```vyper
@view
@external
def getSwappableUsdcAmount(
    _wallet: address,
    _amountIn: uint256,
    _currentBalance: uint256,
    _leverageVaultToken: address,
    _leverageVaultTokenLegoId: uint256,
    _leverageVaultTokenRipeVaultId: uint256,
    _legoBook: address
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_wallet` | `address` | Vault wallet address |
| `_amountIn` | `uint256` | Requested swap amount |
| `_currentBalance` | `uint256` | Current USDC balance |
| `_leverageVaultToken` | `address` | Leverage vault token |
| `_leverageVaultTokenLegoId` | `uint256` | Lego ID |
| `_leverageVaultTokenRipeVaultId` | `uint256` | Ripe vault ID |
| `_legoBook` | `address` | LegoBook registry |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Maximum swappable USDC amount |

### `performPostSwapValidation`

Validates GREEN/USDC swap slippage after execution.

```vyper
@view
@external
def performPostSwapValidation(
    _tokenIn: address,
    _tokenInAmount: uint256,
    _tokenOut: address,
    _tokenOutAmount: uint256,
    _usdcSlippageAllowed: uint256,
    _greenSlippageAllowed: uint256,
    _legoBook: address
) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_tokenIn` | `address` | Input token |
| `_tokenInAmount` | `uint256` | Input amount |
| `_tokenOut` | `address` | Output token |
| `_tokenOutAmount` | `uint256` | Output amount |
| `_usdcSlippageAllowed` | `uint256` | Max USDC slippage (basis points) |
| `_greenSlippageAllowed` | `uint256` | Max GREEN slippage (basis points) |
| `_legoBook` | `address` | LegoBook registry |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if slippage is acceptable |

#### Validation Logic

For GREEN -> USDC swaps:
```
expectedUsdcOut = greenInAmount * greenPrice / USDC_DECIMALS
minAcceptable = expectedUsdcOut * (10000 - greenSlippageAllowed) / 10000
return actualUsdcOut >= minAcceptable
```

For USDC -> GREEN swaps:
```
expectedGreenOut = usdcInAmount * USDC_DECIMALS / greenPrice
minAcceptable = expectedGreenOut * (10000 - usdcSlippageAllowed) / 10000
return actualGreenOut >= minAcceptable
```

## Collateral Query Functions

### `getCollateralBalance`

Gets user's collateral balance in Ripe Protocol.

```vyper
@view
@external
def getCollateralBalance(
    _user: address,
    _asset: address,
    _ripeVaultId: uint256,
    _vaultBook: address
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_user` | `address` | User address |
| `_asset` | `address` | Collateral asset |
| `_ripeVaultId` | `uint256` | Ripe vault ID |
| `_vaultBook` | `address` | Ripe Vault Book contract |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Collateral balance |

## Vault Token Validation

### `isValidVaultToken`

Validates that a vault token can be used for leverage positions.

```vyper
@view
@external
def isValidVaultToken(
    _underlyingAsset: address,
    _vaultToken: address,
    _ripeVaultId: uint256,
    _legoId: uint256
) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_underlyingAsset` | `address` | Expected underlying asset |
| `_vaultToken` | `address` | Vault token to validate |
| `_ripeVaultId` | `uint256` | Ripe vault ID |
| `_legoId` | `uint256` | Lego partner ID |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if vault token is valid |

#### Validation

1. Checks vault token underlying matches expected asset
2. Validates Ripe vault ID is supported
3. Confirms asset is accepted in Ripe vault

### `getVaultBookAndDeleverage`

Returns Ripe Vault Book and Deleverage contract addresses.

```vyper
@view
@external
def getVaultBookAndDeleverage() -> (address, address):
```

#### Returns

| Type | Description |
|------|-------------|
| `address` | Ripe Vault Book address |
| `address` | Ripe Deleverage address |

## Internal Helper Functions

### `_getUnderlyingGreenAmount`

Calculates total GREEN holdings including SavingsGREEN.

```vyper
@view
@internal
def _getUnderlyingGreenAmount(
    _wallet: address,
    _shouldGetMax: bool,
    _legoBook: address
) -> uint256:
```

Returns sum of:
- Direct GREEN balance
- SavingsGREEN balance converted to GREEN

### `_getTotalUnderlying`

Gets total underlying for a vault token position.

```vyper
@view
@internal
def _getTotalUnderlying(
    _wallet: address,
    _vaultToken: address,
    _legoId: uint256,
    _ripeVaultId: uint256,
    _shouldGetMax: bool,
    _legoBook: address
) -> uint256:
```

Combines:
- Vault token balance converted to underlying
- Collateral balance in Ripe (if applicable)

### `_getUserDebtAmount`

Gets user's current debt from Ripe Credit Engine.

```vyper
@view
@internal
def _getUserDebtAmount(_wallet: address) -> uint256:
```

### `_getAssetAmount`

Converts USD value to asset amount via Ripe Price Desk.

```vyper
@view
@internal
def _getAssetAmount(_asset: address, _usdValue: uint256) -> uint256:
```

### `_getUsdValue`

Converts asset amount to USD value via Ripe Price Desk.

```vyper
@view
@internal
def _getUsdValue(_asset: address, _amount: uint256) -> uint256:
```

### `_getUnderlyingAmount`

Converts vault token amount to underlying via ERC4626.

```vyper
@view
@internal
def _getUnderlyingAmount(
    _vaultToken: address,
    _vaultTokenAmount: uint256,
    _shouldGetMax: bool
) -> uint256:
```

## Usage Examples

### Calculating Total Assets (USDC Vault)

```python
total_assets = helper.getTotalAssetsForUsdcVault(
    vault_wallet,
    morpho_usdc_vault,        # collateral vault token
    morpho_lego_id,           # collateral lego ID
    ripe_vault_id_usdc,       # Ripe vault ID
    euler_usdc_vault,         # leverage vault token
    euler_lego_id,            # leverage lego ID
    ripe_vault_id_usdc_lev,   # Ripe leverage vault ID
    True,                     # get max (not safe)
    lego_book
)
```

### Validating Borrow Amount

```python
max_borrow = helper.getMaxBorrowAmount(
    vault_wallet,
    usdc_address,
    collateral_vault_token,
    collateral_lego_id,
    ripe_vault_id,
    net_user_capital,    # 1000 USDC
    30000,               # 300% max debt ratio
    True,                # is USDC vault
    lego_book
)

# If max_borrow >= requested_borrow, allow the borrow
```

### Validating Swap Slippage

```python
is_valid = helper.performPostSwapValidation(
    green_token,        # token in
    1000 * 10**18,      # 1000 GREEN in
    usdc_token,         # token out
    995 * 10**6,        # 995 USDC out
    100,                # 1% USDC slippage allowed
    100,                # 1% GREEN slippage allowed
    lego_book
)

# If is_valid is False, revert the swap
```

## Security Considerations

### Stateless Design
- No state variables (except immutables)
- All calculations are pure based on inputs
- Cannot be manipulated between calls

### Price Oracle Dependency
- Relies on Ripe Price Desk for conversions
- Price manipulation in Ripe affects calculations
- Used for risk management, not direct fund transfers

### Debt Ratio Safety
- Conservative calculation using net user capital
- Prevents over-leveraging through multiple transactions
- Enforced before every borrow operation
