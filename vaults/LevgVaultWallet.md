# LevgVaultWallet Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore/blob/master/contracts/vaults/modules/LevgVaultWallet.vy)

## Overview

LevgVaultWallet is an internal Vyper module that manages leveraged vault state, yield positions, debt management, and swaps for [LevgVault](LevgVault.md). It handles the complex state management required for leveraged positions including collateral tracking, debt operations via Ripe Protocol, and slippage validation for GREEN/USDC swaps.

**Core Responsibilities**:
- **Dual Position Management**: Tracks collateral and leverage vault positions separately
- **Debt Operations**: Executes addCollateral, removeCollateral, borrow, repayDebt through Ripe
- **Swap Validation**: Enforces slippage limits on GREEN/USDC swaps
- **Manager Registry**: Tracks authorized AI agent managers
- **Capital Tracking**: Maintains `netUserCapital` for risk management

This module is imported by LevgVault and its functions are called internally.

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                        LevgVaultWallet Module                            |
|                      (Imported by LevgVault)                            |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Position State                                 | |
|  |                                                                   | |
|  |  collateralAsset  -->  RipeAsset (vaultToken, ripeVaultId)       | |
|  |  leverageAsset    -->  RipeAsset (vaultToken, ripeVaultId)       | |
|  |  vaultToLegoId    -->  Maps vault token to lego ID               | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Risk Parameters                                | |
|  |                                                                   | |
|  |  maxDebtRatio         -->  Max debt as % of capital              | |
|  |  usdcSlippageAllowed  -->  Max USDC swap slippage                | |
|  |  greenSlippageAllowed -->  Max GREEN swap slippage               | |
|  |  netUserCapital       -->  Deposits - withdrawals tracking       | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Helper Integration                             | |
|  |                                                                   | |
|  |  levgVaultHelper  -->  Calculation contract for:                 | |
|  |    - Total assets calculation                                     | |
|  |    - Max borrow amount                                            | |
|  |    - Post-swap validation                                         | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Manager Registry                               | |
|  |                                                                   | |
|  |  managers[]       -->  Index-based manager tracking               | |
|  |  numManagers      -->  Total managers count                       | |
|  +-------------------------------------------------------------------+ |
+-------------------------------------------------------------------------+
```

## Data Structures

### RipeAsset Struct
Tracks Ripe Protocol position information:
```vyper
struct RipeAsset:
    vaultToken: address   # Vault token address
    ripeVaultId: uint256  # Ripe vault ID for this position
```

### VaultActionData Struct
Contains context for vault operations:
```vyper
struct VaultActionData:
    ledger: address
    missionControl: address
    legoBook: address
    appraiser: address
    vaultRegistry: address
    vaultAsset: address
    signer: address
    legoId: uint256
    legoAddr: address
```

## State Variables

### Position Tracking
```vyper
vaultToLegoId: public(HashMap[address, uint256])
# Maps vault token to lego ID

collateralAsset: public(RipeAsset)
# Collateral vault token and Ripe vault ID

leverageAsset: public(RipeAsset)
# Leverage vault token and Ripe vault ID
```

### Risk Parameters
```vyper
maxDebtRatio: public(uint256)
# Maximum debt as % of capital (basis points, max 30000 = 300%)

usdcSlippageAllowed: public(uint256)
# Maximum slippage for USDC swaps (basis points)

greenSlippageAllowed: public(uint256)
# Maximum slippage for GREEN swaps (basis points)

netUserCapital: public(uint256)
# Cumulative deposits minus withdrawals
```

### Helper Contract
```vyper
levgVaultHelper: public(address)
# Address of LevgVaultHelper for calculations
```

### Manager Registry
```vyper
managers: public(HashMap[uint256, address])
# Index to manager mapping

indexOfManager: public(HashMap[address, uint256])
# Manager to index mapping

numManagers: public(uint256)
# Total managers count
```

### Constants
```vyper
HUNDRED_PERCENT: constant(uint256) = 100_00
RIPE_LEGO_ID: constant(uint256) = 1
LEGO_BOOK_ID: constant(uint256) = 3
SWITCHBOARD_ID: constant(uint256) = 4
VAULT_REGISTRY_ID: constant(uint256) = 10
```

## Yield Management Functions

### `depositForYield`

Deposits into yield protocols (collateral or leverage vault).

```vyper
@external
@nonreentrant
def depositForYield(
    _legoId: uint256,
    _asset: address,
    _vaultAddr: address,
    _amount: uint256,
    _extraData: bytes32
) -> (uint256, address, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_legoId` | `uint256` | Lego partner ID |
| `_asset` | `address` | Asset to deposit |
| `_vaultAddr` | `address` | Target vault address |
| `_amount` | `uint256` | Amount to deposit |
| `_extraData` | `bytes32` | Protocol-specific data |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount deposited |
| `address` | Vault token received |
| `uint256` | Vault tokens received |
| `uint256` | USD value |

#### Access

Managers or Switchboard only

### `withdrawFromYield`

Withdraws from yield positions.

```vyper
@external
@nonreentrant
def withdrawFromYield(
    _legoId: uint256,
    _vaultToken: address,
    _amount: uint256,
    _extraData: bytes32,
    _isSpecialTx: bool
) -> (uint256, address, uint256, uint256):
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Vault tokens burned |
| `address` | Underlying asset |
| `uint256` | Underlying received |
| `uint256` | USD value |

## Swap Functions

### `swapTokens`

Executes token swaps with slippage validation for GREEN/USDC pairs.

```vyper
@external
@nonreentrant
def swapTokens(
    _instructions: DynArray[SwapInstruction, MAX_SWAP_INSTRUCTIONS]
) -> (address, uint256, address, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_instructions` | `DynArray[SwapInstruction, 5]` | Swap instructions |

#### Returns

| Type | Description |
|------|-------------|
| `address` | Input token |
| `uint256` | Input amount |
| `address` | Output token |
| `uint256` | Output amount |
| `uint256` | USD value |

#### Slippage Validation

For GREEN <-> USDC swaps, the module:
1. Calls `levgVaultHelper.performPostSwapValidation()`
2. Validates against `usdcSlippageAllowed` and `greenSlippageAllowed`
3. Reverts if slippage exceeds limits

## Debt Management Functions

### `addCollateral`

Adds collateral to Ripe Protocol.

```vyper
@external
@nonreentrant
def addCollateral(
    _legoId: uint256,
    _asset: address,
    _amount: uint256,
    _extraData: bytes32
) -> (uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_legoId` | `uint256` | Must be RIPE_LEGO_ID |
| `_asset` | `address` | Collateral asset |
| `_amount` | `uint256` | Amount to add |
| `_extraData` | `bytes32` | Ripe-specific data |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount added |
| `uint256` | USD value |

#### Events Emitted

- `LevgVaultAction(op=ADD_COLLATERAL)`

### `removeCollateral`

Removes collateral from Ripe Protocol.

```vyper
@external
@nonreentrant
def removeCollateral(
    _legoId: uint256,
    _asset: address,
    _amount: uint256,
    _extraData: bytes32
) -> (uint256, uint256):
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount removed |
| `uint256` | USD value |

#### Events Emitted

- `LevgVaultAction(op=REMOVE_COLLATERAL)`

### `borrow`

Borrows GREEN from Ripe Protocol against collateral.

```vyper
@external
@nonreentrant
def borrow(
    _legoId: uint256,
    _borrowAsset: address,
    _amount: uint256,
    _extraData: bytes32
) -> (uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_legoId` | `uint256` | Must be RIPE_LEGO_ID |
| `_borrowAsset` | `address` | GREEN or SavingsGREEN |
| `_amount` | `uint256` | Amount to borrow |
| `_extraData` | `bytes32` | Ripe-specific data |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount borrowed |
| `uint256` | USD value |

#### Debt Ratio Validation

Before borrowing:
1. Calls `levgVaultHelper.getMaxBorrowAmount()`
2. Validates borrow amount <= max allowed
3. Reverts if would exceed `maxDebtRatio`

#### Events Emitted

- `LevgVaultAction(op=BORROW)`

### `repayDebt`

Repays GREEN debt to Ripe Protocol.

```vyper
@external
@nonreentrant
def repayDebt(
    _legoId: uint256,
    _paymentAsset: address,
    _paymentAmount: uint256,
    _extraData: bytes32
) -> (uint256, uint256):
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount repaid |
| `uint256` | USD value |

#### Events Emitted

- `LevgVaultAction(op=REPAY_DEBT)`

## Reward Functions

### `claimIncentives`

Claims protocol rewards/incentives.

```vyper
@external
@nonreentrant
def claimIncentives(
    _legoId: uint256,
    _rewardToken: address,
    _rewardAmount: uint256,
    _proofs: DynArray[bytes32, MAX_PROOFS]
) -> (uint256, uint256):
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount claimed |
| `uint256` | USD value |

## Configuration Functions

### `setCollateralVault`

Sets the collateral vault token configuration.

```vyper
@external
def setCollateralVault(
    _vaultToken: address,
    _legoId: uint256,
    _ripeVaultId: uint256,
    _shouldMaxWithdraw: bool
):
```

#### Behavior

1. Validates vault token via LevgVaultHelper
2. If `_shouldMaxWithdraw`, withdraws all from previous vault
3. Updates `collateralAsset` struct
4. Updates `vaultToLegoId` mapping
5. Emits `CollateralVaultTokenSet`

### `setLeverageVault`

Sets the leverage vault token configuration.

```vyper
@external
def setLeverageVault(
    _vaultToken: address,
    _legoId: uint256,
    _ripeVaultId: uint256,
    _shouldMaxWithdraw: bool
):
```

#### Behavior

Similar to `setCollateralVault` but for leverage position.

### `setSlippagesAllowed`

Updates slippage tolerance for swaps.

```vyper
@external
def setSlippagesAllowed(_usdcSlippage: uint256, _greenSlippage: uint256):
```

### `setLevgVaultHelper`

Updates the helper contract address.

```vyper
@external
def setLevgVaultHelper(_levgVaultHelper: address):
```

### `setMaxDebtRatio`

Sets maximum debt ratio limit.

```vyper
@external
def setMaxDebtRatio(_ratio: uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_ratio` | `uint256` | Max ratio in basis points (max 30000 = 300%) |

## Manager Functions

### `addManager` / `removeManager`

```vyper
@external
def addManager(_manager: address):

@external
def removeManager(_manager: address):
```

## Internal Functions

### `_onReceiveVaultFunds`

Called when vault receives deposits, auto-deposits to collateral vault.

```vyper
@internal
def _onReceiveVaultFunds(
    _depositor: address,
    _vaultRegistry: address
) -> uint256:
```

#### Behavior

1. Gets deposit amount
2. If auto-deposit enabled, deposits to collateral vault
3. Returns amount deposited

### `_prepareRedemption`

Prepares for user withdrawal, may involve deleveraging.

```vyper
@internal
def _prepareRedemption(
    _requestedAmount: uint256,
    _vaultRegistry: address,
    _legoBook: address
) -> uint256:
```

#### Behavior

For leveraged vaults, redemption may require:
1. Withdrawing from leverage vault
2. Swapping to GREEN
3. Repaying debt
4. Removing collateral
5. Withdrawing from collateral vault

### `_getTotalAssets`

Calculates total assets for the vault.

```vyper
@view
@internal
def _getTotalAssets(_shouldGetMax: bool) -> uint256:
```

#### Behavior

Routes to LevgVaultHelper based on vault type:
- USDC vaults: `getTotalAssetsForUsdcVault()`
- Non-USDC vaults: `getTotalAssetsForNonUsdcVault()`

## Events

```vyper
event LevgVaultAction:
    op: indexed(uint256)
    asset1: indexed(address)
    asset2: address
    amount1: uint256
    amount2: uint256
    usdValue: uint256
    legoId: uint256
    signer: indexed(address)

event CollateralVaultTokenSet:
    collateralVaultToken: indexed(address)
    legoId: uint256
    ripeVaultId: uint256

event LeverageVaultTokenSet:
    leverageVaultToken: indexed(address)
    legoId: uint256
    ripeVaultId: uint256

event SlippagesSet:
    usdcSlippage: uint256
    greenSlippage: uint256

event LevgVaultHelperSet:
    levgVaultHelper: indexed(address)

event MaxDebtRatioSet:
    maxDebtRatio: uint256
```

## Internal Mechanisms

### Net User Capital Tracking

The module maintains `netUserCapital` to track actual user deposits:

1. **On Deposit**: `netUserCapital += depositAmount`
2. **On Withdrawal**: `netUserCapital -= withdrawAmount`

This is used by LevgVaultHelper to calculate safe debt ratios, preventing manipulation through cyclical deposits/withdrawals.

### Slippage Validation Flow

For GREEN/USDC swaps:

```
1. Manager calls swapTokens()
2. Module executes swap through Lego
3. Module calls levgVaultHelper.performPostSwapValidation()
4. Helper validates:
   - If GREEN->USDC: Check against greenSlippageAllowed
   - If USDC->GREEN: Check against usdcSlippageAllowed
5. Reverts if slippage exceeds limits
```

### Debt Ratio Enforcement

Before any borrow:

```
1. Module calls levgVaultHelper.getMaxBorrowAmount()
2. Helper calculates:
   - Current collateral value
   - Current debt
   - netUserCapital
   - maxDebtRatio limit
3. Returns maximum safe borrow amount
4. Module validates requested amount <= max
5. Reverts if would exceed ratio
```

## Security Considerations

### Access Control
- All yield/debt operations require manager or Switchboard
- Configuration changes require Switchboard only
- Manager registry changes require Switchboard only

### Debt Safety
- `maxDebtRatio` enforced on every borrow
- Net capital tracking prevents manipulation
- Slippage validation on GREEN/USDC swaps

### Position Integrity
- Collateral and leverage positions tracked separately
- Ripe vault IDs prevent cross-position confusion
- Vault token validation on configuration changes
