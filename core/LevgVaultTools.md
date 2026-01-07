# LevgVaultTools Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/helpers/LevgVaultTools.vy)

## Overview

LevgVaultTools is a convenience contract providing view functions for frontend and web application usage. It aggregates data from multiple sources (LevgVault, Ripe Protocol, Lego partners) to provide comprehensive leverage vault metrics without requiring multiple separate calls.

**Core Features**:
- **Underlying Amount Calculations**: Convert vault tokens to underlying assets across wallet and Ripe positions
- **Ripe Protocol Integration**: Query collateral balances, debt amounts, and borrow limits
- **GREEN/sGREEN Tracking**: Track GREEN token balances across all positions
- **Ratio Calculations**: Compute debt-to-deposit, debt utilization, and LTV ratios
- **Borrow Limits**: Calculate true max borrow amounts considering both vault and Ripe constraints
- **Batch Queries**: Get multiple related values in single calls

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                           LevgVaultTools                                 |
|                    (Frontend Convenience Helper)                         |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Data Aggregation                               | |
|  |                                                                   | |
|  |  LevgVault        -->  Vault token data, total assets             | |
|  |  Ripe Protocol    -->  Collateral, debt, borrow limits            | |
|  |  Yield Legos      -->  Underlying amount conversions              | |
|  |  Price Desk       -->  USD valuations                             | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Query Categories                               | |
|  |                                                                   | |
|  |  Underlying Amounts  -->  Total underlying for positions          | |
|  |  Ripe Collateral     -->  Collateral in Ripe vaults               | |
|  |  GREEN Helpers       -->  GREEN/sGREEN balance tracking           | |
|  |  Borrow Helpers      -->  Debt, rates, max borrow amounts         | |
|  |  Ratio Calculations  -->  Debt ratios, utilization metrics        | |
|  |  Combination Queries -->  Batch data retrieval                    | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    External Dependencies                          | |
|  |                                                                   | |
|  |  UndyHq / LegoBook    -->  Lego partner addresses                 | |
|  |  Ripe Registry        -->  Ripe contract addresses                | |
|  |  Ripe Credit Engine   -->  Debt and borrow calculations           | |
|  |  Ripe Price Desk      -->  USD price conversions                  | |
|  |  Ripe Vault Book      -->  Deposit vault addresses                | |
|  +-------------------------------------------------------------------+ |
+-------------------------------------------------------------------------+
```

## Data Structures

### VaultToken

Tracks yield position metadata:
```vyper
struct VaultToken:
    legoId: uint256         # Lego partner ID
    underlyingAsset: address # Asset deposited in vault
    decimals: uint256       # Token decimals
    isRebasing: bool        # Whether token rebases
```

### RipeAsset

Represents a Ripe Protocol position:
```vyper
struct RipeAsset:
    vaultToken: address   # Vault token address
    ripeVaultId: uint256  # Ripe vault ID
```

## Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `RIPE_MISSION_CONTROL_ID` | 5 | Ripe registry ID for MissionControl |
| `RIPE_PRICE_DESK_ID` | 7 | Ripe registry ID for PriceDesk |
| `RIPE_VAULT_BOOK_ID` | 8 | Ripe registry ID for VaultBook |
| `RIPE_CREDIT_ENGINE_ID` | 13 | Ripe registry ID for CreditEngine |
| `RIPE_ENDAOMENT_PSM_ID` | 22 | Ripe registry ID for Endaoment PSM |
| `STAB_POOL_ID` | 1 | Ripe stability pool vault ID |
| `HUNDRED_PERCENT` | 10000 | 100.00% in basis points |

## State Variables

| Name | Type | Description |
|------|------|-------------|
| `RIPE_REGISTRY` | `address` (immutable) | Ripe Protocol registry address |
| `GREEN_TOKEN` | `address` (immutable) | GREEN token address |
| `SAVINGS_GREEN` | `address` (immutable) | Savings GREEN (sGREEN) token address |
| `USDC` | `address` (immutable) | USDC token address |

## Constructor

### `__init__`

```vyper
@deploy
def __init__(_undyHq: address, _ripeRegistry: address, _usdc: address):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyHq` | `address` | UndyHq registry address |
| `_ripeRegistry` | `address` | Ripe Protocol registry address |
| `_usdc` | `address` | USDC token address |

---

## Total Underlying Functions

### `getTotalUnderlyingAmount`

Returns the total underlying asset amount for a position (collateral or leverage), combining wallet balance, naked Ripe collateral, and vault token positions.

```vyper
@view
@external
def getTotalUnderlyingAmount(
    _levgVault: address,
    _isCollateralAsset: bool,
    _shouldGetMax: bool,
    _legoBook: address = empty(address),
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgVault` | `address` | Leverage vault address |
| `_isCollateralAsset` | `bool` | True for collateral position, false for leverage position |
| `_shouldGetMax` | `bool` | True for optimistic valuation, false for conservative |
| `_legoBook` | `address` | Optional LegoBook address override |
| `_ripeVaultBook` | `address` | Optional Ripe VaultBook address override |
| `_ripeMissionControl` | `address` | Optional Ripe MissionControl address override |
| `_ripeHq` | `address` | Optional Ripe registry address override |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Total underlying amount |

#### Notes

- For raw asset collateral vaults, returns wallet + Ripe collateral directly
- For vault token positions, converts vault tokens to underlying via lego

---

## Asset Amount Functions

### `getAmountForAsset`

Returns total amount of a specific asset held by the vault (wallet balance + Ripe collateral).

```vyper
@view
@external
def getAmountForAsset(
    _levgVault: address,
    _asset: address,
    _ripeVaultId: uint256 = 0,
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgVault` | `address` | Leverage vault address |
| `_asset` | `address` | Asset to query |
| `_ripeVaultId` | `uint256` | Optional Ripe vault ID (auto-detected if 0) |
| `_ripeVaultBook` | `address` | Optional Ripe VaultBook address override |
| `_ripeMissionControl` | `address` | Optional Ripe MissionControl address override |
| `_ripeHq` | `address` | Optional Ripe registry address override |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Total asset amount (wallet + Ripe) |

---

## Vault Token Conversion Functions

### `getUnderlyingAmountForVaultToken`

Converts vault token holdings to underlying asset amount.

```vyper
@view
@external
def getUnderlyingAmountForVaultToken(
    _levgVault: address,
    _isCollateralAsset: bool,
    _shouldGetMax: bool,
    _legoBook: address = empty(address),
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgVault` | `address` | Leverage vault address |
| `_isCollateralAsset` | `bool` | True for collateral position, false for leverage position |
| `_shouldGetMax` | `bool` | True for optimistic conversion, false for conservative |
| `_legoBook` | `address` | Optional LegoBook address override |
| `_ripeVaultBook` | `address` | Optional Ripe VaultBook address override |
| `_ripeMissionControl` | `address` | Optional Ripe MissionControl address override |
| `_ripeHq` | `address` | Optional Ripe registry address override |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Underlying amount from vault token positions |

---

## Ripe Collateral Functions

### `getRipeCollateralBalance`

Returns the amount of an asset deposited as collateral in Ripe Protocol.

```vyper
@view
@external
def getRipeCollateralBalance(
    _levgVault: address,
    _asset: address,
    _ripeVaultId: uint256 = 0,
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgVault` | `address` | Leverage vault address |
| `_asset` | `address` | Asset to query |
| `_ripeVaultId` | `uint256` | Optional Ripe vault ID (auto-detected if 0) |
| `_ripeVaultBook` | `address` | Optional Ripe VaultBook address override |
| `_ripeMissionControl` | `address` | Optional Ripe MissionControl address override |
| `_ripeHq` | `address` | Optional Ripe registry address override |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Collateral amount in Ripe |

---

## GREEN Helper Functions

### `getUnderlyingGreenAmount`

Returns total GREEN amount including raw GREEN in wallet plus GREEN redeemable from sGREEN positions.

```vyper
@view
@external
def getUnderlyingGreenAmount(
    _levgVault: address,
    _green: address = empty(address),
    _savingsGreen: address = empty(address),
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgVault` | `address` | Leverage vault address |
| `_green` | `address` | Optional GREEN token address override |
| `_savingsGreen` | `address` | Optional sGREEN token address override |
| `_ripeVaultBook` | `address` | Optional Ripe VaultBook address override |
| `_ripeMissionControl` | `address` | Optional Ripe MissionControl address override |
| `_ripeHq` | `address` | Optional Ripe registry address override |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Total GREEN amount (18 decimals) |

### `getSavingsGreenBalances`

Returns sGREEN distribution across stability pool and external holdings.

```vyper
@view
@external
def getSavingsGreenBalances(_ripeHq: address = empty(address)) -> (uint256, uint256, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_ripeHq` | `address` | Optional Ripe registry address override |

#### Returns

| Index | Type | Description |
|-------|------|-------------|
| 0 | `uint256` | sGREEN in stability pool |
| 1 | `uint256` | GREEN equivalent in stability pool |
| 2 | `uint256` | sGREEN not in stability pool |
| 3 | `uint256` | GREEN equivalent not in stability pool |

---

## Swap Helper Functions

### `getSwappableUsdcAmount`

Returns USDC amount available for swapping after accounting for debt obligations.

```vyper
@view
@external
def getSwappableUsdcAmount(
    _levgVault: address,
    _leverageVaultToken: address = empty(address),
    _leverageVaultTokenRipeVaultId: uint256 = 0,
    _usdc: address = empty(address),
    _green: address = empty(address),
    _savingsGreen: address = empty(address),
    _legoBook: address = empty(address),
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgVault` | `address` | Leverage vault address |
| `_leverageVaultToken` | `address` | Optional leverage vault token override |
| `_leverageVaultTokenRipeVaultId` | `uint256` | Optional Ripe vault ID override |
| `_usdc` | `address` | Optional USDC address override |
| `_green` | `address` | Optional GREEN address override |
| `_savingsGreen` | `address` | Optional sGREEN address override |
| `_legoBook` | `address` | Optional LegoBook address override |
| `_ripeVaultBook` | `address` | Optional Ripe VaultBook address override |
| `_ripeMissionControl` | `address` | Optional Ripe MissionControl address override |
| `_ripeHq` | `address` | Optional Ripe registry address override |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | USDC amount available for swapping |

#### Notes

- Only applicable for non-USDC collateral vaults
- Accounts for GREEN surplus that can offset debt
- Returns 0 if remaining debt exceeds USDC value

---

## Borrow Helper Functions

### `getBorrowRate`

Returns the current borrow rate for a leverage vault.

```vyper
@view
@external
def getBorrowRate(_levgVault: address, _ripeHq: address = empty(address)) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Current borrow rate |

### `getDebtAmount`

Returns the current debt amount for a leverage vault.

```vyper
@view
@external
def getDebtAmount(_levgVault: address, _ripeHq: address = empty(address)) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Debt amount in GREEN (18 decimals) |

### `getAvailableUsdcFromEndaomentPsm`

Returns available USDC from the Ripe Endaoment PSM.

```vyper
@view
@external
def getAvailableUsdcFromEndaomentPsm(_ripeHq: address = empty(address)) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Available USDC amount |

### `getTrueMaxBorrowAmount`

Returns the true maximum borrow amount, taking the minimum of max debt ratio limit and Ripe LTV limit.

```vyper
@view
@external
def getTrueMaxBorrowAmount(_levgVault: address, _ripeHq: address = empty(address)) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | True max borrow amount (18 decimals) |

### `getMaxBorrowAmountByMaxDebtRatio`

Returns maximum borrow amount based on the vault's max debt ratio setting.

```vyper
@view
@external
def getMaxBorrowAmountByMaxDebtRatio(
    _levgVault: address,
    _underlyingAsset: address = empty(address),
    _collateralVaultToken: address = empty(address),
    _collateralVaultTokenRipeVaultId: uint256 = 0,
    _leverageVaultToken: address = empty(address),
    _totalAssets: uint256 = 0,
    _maxDebtRatio: uint256 = 0,
    _legoBook: address = empty(address),
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgVault` | `address` | Leverage vault address |
| `_underlyingAsset` | `address` | Optional underlying asset override |
| `_collateralVaultToken` | `address` | Optional collateral vault token override |
| `_collateralVaultTokenRipeVaultId` | `uint256` | Optional Ripe vault ID override |
| `_leverageVaultToken` | `address` | Optional leverage vault token override |
| `_totalAssets` | `uint256` | Optional total assets override |
| `_maxDebtRatio` | `uint256` | Optional max debt ratio override |
| `_legoBook` | `address` | Optional LegoBook address override |
| `_ripeVaultBook` | `address` | Optional Ripe VaultBook address override |
| `_ripeMissionControl` | `address` | Optional Ripe MissionControl address override |
| `_ripeHq` | `address` | Optional Ripe registry address override |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Max borrow amount based on debt ratio (18 decimals) |

#### Notes

- Returns `max_value(uint256)` if max debt ratio is 0 (unlimited)
- Accounts for existing debt and GREEN surplus

### `getMaxBorrowAmountByRipeLtv`

Returns maximum borrow amount based on Ripe Protocol's LTV constraints.

```vyper
@view
@external
def getMaxBorrowAmountByRipeLtv(
    _levgVault: address,
    _creditEngine: address = empty(address),
    _ripeHq: address = empty(address),
) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Max borrow amount based on Ripe LTV (18 decimals) |

### `getNetUserDebt`

Returns net user debt after accounting for GREEN surplus.

```vyper
@view
@external
def getNetUserDebt(
    _levgVault: address,
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Net debt (debt - GREEN surplus), 0 if GREEN covers all debt |

---

## Ratio Calculation Functions

### `getDebtToDepositRatio`

Returns the debt-to-deposit ratio in basis points.

```vyper
@view
@external
def getDebtToDepositRatio(
    _levgVault: address,
    _underlyingAsset: address = empty(address),
    _collateralVaultToken: address = empty(address),
    _collateralVaultTokenRipeVaultId: uint256 = 0,
    _leverageVaultToken: address = empty(address),
    _totalAssets: uint256 = 0,
    _legoBook: address = empty(address),
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Debt-to-deposit ratio in basis points (e.g., 5000 = 50%) |

#### Calculation

```
ratio = (netDebt * HUNDRED_PERCENT) / depositUsdValue
```

### `getDebtUtilization`

Returns how much of the max debt ratio is currently being used.

```vyper
@view
@external
def getDebtUtilization(
    _levgVault: address,
    _underlyingAsset: address = empty(address),
    _collateralVaultToken: address = empty(address),
    _collateralVaultTokenRipeVaultId: uint256 = 0,
    _leverageVaultToken: address = empty(address),
    _totalAssets: uint256 = 0,
    _maxDebtRatio: uint256 = 0,
    _legoBook: address = empty(address),
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Utilization in basis points (e.g., 8000 = 80% of max debt ratio used) |

#### Calculation

```
utilization = (debtToDepositRatio * HUNDRED_PERCENT) / maxDebtRatio
```

### `getDebtToRipeCollateralRatio`

Returns the debt-to-Ripe-collateral ratio in basis points.

```vyper
@view
@external
def getDebtToRipeCollateralRatio(
    _levgVault: address,
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Debt-to-Ripe-collateral ratio in basis points |

---

## Combination Query Functions

### `getVaultTokenAmounts`

Returns vault token balances split by location (wallet vs Ripe).

```vyper
@view
@external
def getVaultTokenAmounts(
    _levgVault: address,
    _isCollateralAsset: bool,
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> (uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgVault` | `address` | Leverage vault address |
| `_isCollateralAsset` | `bool` | True for collateral, false for leverage |
| `_ripeVaultBook` | `address` | Optional Ripe VaultBook address override |
| `_ripeMissionControl` | `address` | Optional Ripe MissionControl address override |
| `_ripeHq` | `address` | Optional Ripe registry address override |

#### Returns

| Index | Type | Description |
|-------|------|-------------|
| 0 | `uint256` | Vault tokens in wallet |
| 1 | `uint256` | Vault tokens in Ripe collateral |

### `getUnderlyingAmounts`

Returns detailed breakdown of underlying asset positions.

```vyper
@view
@external
def getUnderlyingAmounts(
    _levgVault: address,
    _isCollateralAsset: bool,
    _shouldGetMax: bool,
    _legoBook: address = empty(address),
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> (uint256, uint256, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgVault` | `address` | Leverage vault address |
| `_isCollateralAsset` | `bool` | True for collateral, false for leverage |
| `_shouldGetMax` | `bool` | True for optimistic, false for conservative |
| `_legoBook` | `address` | Optional LegoBook address override |
| `_ripeVaultBook` | `address` | Optional Ripe VaultBook address override |
| `_ripeMissionControl` | `address` | Optional Ripe MissionControl address override |
| `_ripeHq` | `address` | Optional Ripe registry address override |

#### Returns

| Index | Type | Description |
|-------|------|-------------|
| 0 | `uint256` | Underlying asset in wallet (raw) |
| 1 | `uint256` | Underlying from vault tokens in wallet |
| 2 | `uint256` | Underlying asset in Ripe (raw) |
| 3 | `uint256` | Underlying from vault tokens in Ripe |

### `getGreenAmounts`

Returns detailed breakdown of GREEN/sGREEN positions.

```vyper
@view
@external
def getGreenAmounts(
    _levgVault: address,
    _green: address = empty(address),
    _savingsGreen: address = empty(address),
    _ripeVaultBook: address = empty(address),
    _ripeMissionControl: address = empty(address),
    _ripeHq: address = empty(address),
) -> (uint256, uint256, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgVault` | `address` | Leverage vault address |
| `_green` | `address` | Optional GREEN token address override |
| `_savingsGreen` | `address` | Optional sGREEN token address override |
| `_ripeVaultBook` | `address` | Optional Ripe VaultBook address override |
| `_ripeMissionControl` | `address` | Optional Ripe MissionControl address override |
| `_ripeHq` | `address` | Optional Ripe registry address override |

#### Returns

| Index | Type | Description |
|-------|------|-------------|
| 0 | `uint256` | User debt in GREEN (18 decimals) |
| 1 | `uint256` | GREEN in wallet |
| 2 | `uint256` | GREEN equivalent from sGREEN in wallet |
| 3 | `uint256` | GREEN equivalent from sGREEN in Ripe |

---

## Common Integration Patterns

### Fetching Vault Health Metrics

```python
# Get comprehensive vault health data
tools = LevgVaultTools(tools_address)

# Debt and utilization
debt_ratio = tools.getDebtToDepositRatio(levg_vault)
utilization = tools.getDebtUtilization(levg_vault)
ripe_ltv = tools.getDebtToRipeCollateralRatio(levg_vault)

# Borrow capacity
max_borrow = tools.getTrueMaxBorrowAmount(levg_vault)
current_debt = tools.getDebtAmount(levg_vault)
net_debt = tools.getNetUserDebt(levg_vault)

print(f"Debt Ratio: {debt_ratio / 100}%")
print(f"Utilization: {utilization / 100}%")
print(f"Available to Borrow: {max_borrow - current_debt}")
```

### Fetching Position Breakdown

```python
# Get detailed position data
tools = LevgVaultTools(tools_address)

# Collateral position breakdown
underlying_wallet, underlying_vault_wallet, underlying_ripe, underlying_vault_ripe = \
    tools.getUnderlyingAmounts(levg_vault, True, False)  # collateral, conservative

total_collateral = underlying_wallet + underlying_vault_wallet + underlying_ripe + underlying_vault_ripe

# GREEN position breakdown
debt, green_wallet, sgreen_wallet, sgreen_ripe = \
    tools.getGreenAmounts(levg_vault)

total_green = green_wallet + sgreen_wallet + sgreen_ripe
net_debt = max(0, debt - total_green)

print(f"Total Collateral Underlying: {total_collateral}")
print(f"Total GREEN Available: {total_green}")
print(f"Net Debt: {net_debt}")
```

### Checking Swap Availability

```python
# Check how much USDC can be swapped
tools = LevgVaultTools(tools_address)

swappable_usdc = tools.getSwappableUsdcAmount(levg_vault)

if swappable_usdc > 0:
    print(f"Can swap up to {swappable_usdc} USDC")
else:
    print("No USDC available for swapping (debt obligations)")
```

### Pre-Borrow Validation

```python
# Validate borrow amount before transaction
tools = LevgVaultTools(tools_address)

desired_borrow = 1000 * 10**18  # 1000 GREEN

# Check both limits
max_by_debt_ratio = tools.getMaxBorrowAmountByMaxDebtRatio(levg_vault)
max_by_ripe_ltv = tools.getMaxBorrowAmountByRipeLtv(levg_vault)
true_max = tools.getTrueMaxBorrowAmount(levg_vault)

if desired_borrow > true_max:
    print(f"Cannot borrow {desired_borrow}. Max: {true_max}")
    print(f"  - Debt ratio limit: {max_by_debt_ratio}")
    print(f"  - Ripe LTV limit: {max_by_ripe_ltv}")
else:
    print("Borrow amount is valid")
```

---

## Security Considerations

### View-Only Functions
- All functions are `@view` - no state modifications
- Safe to call from any context without gas concerns for state changes

### Address Overrides
- Optional address parameters allow testing with different contracts
- Production usage should typically use defaults (empty addresses)
- Overrides useful for simulations and what-if scenarios

### Decimal Handling
- GREEN and debt amounts use 18 decimals
- USDC amounts use 6 decimals
- Ratios returned in basis points (10000 = 100%)
- USD values from Ripe PriceDesk use 18 decimals

### Ripe Protocol Dependencies
- Relies on Ripe Protocol contracts for debt/collateral data
- If Ripe contracts are unavailable, queries may fail
- Consider graceful error handling in frontend integrations
