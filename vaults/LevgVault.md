# LevgVault Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore/blob/master/contracts/vaults/LevgVault.vy)

## Overview

LevgVault is an ERC4626-compliant leveraged vault that enables users to amplify their yield returns through borrowing. It integrates with the Ripe Protocol for lending/borrowing operations and manages two distinct asset positions: a collateral vault and a leverage vault. Users deposit assets and receive shares, while AI agent managers execute leveraged strategies within defined risk parameters.

**Core Features**:
- **ERC4626 Compliance**: Standard vault interface for seamless integration
- **Leveraged Yield**: Borrows against collateral to amplify returns
- **Dual Position System**: Manages collateral and leverage vault tokens separately
- **Ripe Protocol Integration**: Uses Ripe for lending, borrowing, and collateral management
- **Debt Ratio Enforcement**: Maximum debt ratio limits (up to 300%)
- **Slippage Controls**: Configurable slippage limits for GREEN/USDC swaps
- **Net Capital Tracking**: Tracks user deposits minus withdrawals for risk management

The vault uses [LevgVaultWallet](LevgVaultWallet.md) for state management, [LevgVaultHelper](LevgVaultHelper.md) for calculations, and [VaultErc20Token](VaultErc20Token.md) for share tokens.

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                              LevgVault                                   |
|                   (ERC4626 + Leveraged Positions)                       |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Dual Position System                           | |
|  |                                                                   | |
|  |  Collateral Vault Token  -->  User's base capital position        | |
|  |  Leverage Vault Token    -->  Borrowed capital position           | |
|  |                                                                   | |
|  |  Total Assets = Collateral + Leverage - Debt                      | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Risk Parameters                                | |
|  |                                                                   | |
|  |  maxDebtRatio         -->  Maximum debt as % of capital (300%)   | |
|  |  usdcSlippageAllowed  -->  Max slippage on USDC swaps            | |
|  |  greenSlippageAllowed -->  Max slippage on GREEN swaps           | |
|  |  netUserCapital       -->  Tracks deposits - withdrawals         | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Debt Management                                | |
|  |                     (via Ripe Protocol)                           | |
|  |                                                                   | |
|  |  addCollateral()    -->  Deposit collateral to Ripe              | |
|  |  removeCollateral() -->  Withdraw collateral from Ripe           | |
|  |  borrow()          -->  Borrow GREEN from Ripe                   | |
|  |  repayDebt()       -->  Repay GREEN debt                         | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    External Dependencies                          | |
|  |                                                                   | |
|  |  Ripe Protocol     -->  Lending/borrowing infrastructure         | |
|  |  LevgVaultHelper   -->  Calculation helper contract              | |
|  |  VaultRegistry     -->  Configuration and approvals              | |
|  |  LegoBook          -->  Lego partner registry                    | |
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
    ledger: address         # Ledger contract
    missionControl: address # MissionControl contract
    legoBook: address       # LegoBook registry
    appraiser: address      # Appraiser for USD values
    vaultRegistry: address  # VaultRegistry config
    vaultAsset: address     # Vault's underlying asset
    signer: address         # Transaction signer
    legoId: uint256         # Lego ID for operation
    legoAddr: address       # Lego contract address
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
# Maximum debt as % of capital (basis points, max 300% = 30000)

usdcSlippageAllowed: public(uint256)
# Maximum slippage for USDC swaps (basis points)

greenSlippageAllowed: public(uint256)
# Maximum slippage for GREEN swaps (basis points)

netUserCapital: public(uint256)
# Cumulative deposits minus withdrawals (tracks user capital)
```

### Helper Contract
```vyper
levgVaultHelper: public(address)
# Address of LevgVaultHelper contract for calculations
```

### Manager Registry
```vyper
managers: public(HashMap[uint256, address])
indexOfManager: public(HashMap[address, uint256])
numManagers: public(uint256)
```

### Constants
```vyper
HUNDRED_PERCENT: constant(uint256) = 100_00
RIPE_LEGO_ID: constant(uint256) = 1
LEGO_BOOK_ID: constant(uint256) = 3
SWITCHBOARD_ID: constant(uint256) = 4
VAULT_REGISTRY_ID: constant(uint256) = 10
```

## ERC4626 Deposit Functions

### `deposit`

Deposits exact amount of assets and receives proportional shares.

```vyper
@external
@nonreentrant
def deposit(_assets: uint256, _receiver: address) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_assets` | `uint256` | Amount of underlying asset to deposit |
| `_receiver` | `address` | Address to receive shares |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of shares minted |

#### Behavior

1. Pulls assets from depositor
2. Calculates shares based on total assets
3. Auto-deposits to collateral vault if configured
4. Increases `netUserCapital`
5. Mints shares to receiver

#### Events Emitted

- `Deposit(sender, owner, assets, shares)`

### `depositWithMinAmountOut`

Deposits with slippage protection on minimum shares.

```vyper
@external
@nonreentrant
def depositWithMinAmountOut(
    _assets: uint256,
    _minAmountOut: uint256,
    _receiver: address
) -> uint256:
```

### `mint`

Mints exact shares, pulling required assets.

```vyper
@external
@nonreentrant
def mint(_shares: uint256, _receiver: address) -> uint256:
```

### `maxDeposit` / `maxMint`

Returns maximum deposit/mint amounts.

```vyper
@view
@external
def maxDeposit(_receiver: address) -> uint256:

@view
@external
def maxMint(_receiver: address) -> uint256:
```

## ERC4626 Withdrawal Functions

### `withdraw`

Withdraws exact amount of assets, burning required shares.

```vyper
@external
@nonreentrant
def withdraw(_assets: uint256, _receiver: address, _owner: address) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_assets` | `uint256` | Amount of underlying asset to withdraw |
| `_receiver` | `address` | Address to receive assets |
| `_owner` | `address` | Owner of shares to burn |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of shares burned |

#### Behavior

1. Calculates shares needed for assets
2. May trigger deleveraging if insufficient liquidity
3. Decreases `netUserCapital`
4. Burns shares and transfers assets

### `redeem`

Redeems exact shares for proportional assets.

```vyper
@external
@nonreentrant
def redeem(_shares: uint256, _receiver: address, _owner: address) -> uint256:
```

### `redeemWithMinAmountOut`

Redeems with slippage protection.

```vyper
@external
@nonreentrant
def redeemWithMinAmountOut(
    _shares: uint256,
    _minAmountOut: uint256,
    _receiver: address,
    _owner: address
) -> uint256:
```

### `maxWithdraw` / `maxRedeem`

Returns maximum withdrawal/redemption amounts.

```vyper
@view
@external
def maxWithdraw(_owner: address) -> uint256:

@view
@external
def maxRedeem(_owner: address) -> uint256:
```

## Share Conversion Functions

### `convertToShares` / `convertToSharesSafe`

Converts asset amount to share amount.

```vyper
@view
@external
def convertToShares(_assets: uint256) -> uint256:

@view
@external
def convertToSharesSafe(_assets: uint256) -> uint256:
```

### `convertToAssets` / `convertToAssetsSafe`

Converts share amount to asset amount.

```vyper
@view
@external
def convertToAssets(_shares: uint256) -> uint256:

@view
@external
def convertToAssetsSafe(_shares: uint256) -> uint256:
```

## Asset Functions

### `asset`

Returns the vault's underlying asset.

```vyper
@view
@external
def asset() -> address:
```

### `totalAssets` / `getTotalAssets`

Returns total assets under management.

```vyper
@view
@external
def totalAssets() -> uint256:

@view
@external
def getTotalAssets(_shouldGetMax: bool) -> uint256:
```

**Total Assets Calculation**:
- For USDC vaults: `Collateral + Leverage - Debt` (all in USDC terms)
- For non-USDC vaults (WETH/CBBTC): `Collateral + (Leverage - Debt) converted to underlying`

### `isLeveragedVault`

Returns True indicating this is a leveraged vault.

```vyper
@view
@external
def isLeveragedVault() -> bool:
```

## Debt Management Functions (Manager Only)

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
| `_legoId` | `uint256` | Must be RIPE_LEGO_ID (1) |
| `_asset` | `address` | Asset to deposit as collateral |
| `_amount` | `uint256` | Amount to deposit |
| `_extraData` | `bytes32` | Ripe-specific data |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount deposited |
| `uint256` | USD value of transaction |

#### Events Emitted

- `LevgVaultAction(op=ADD_COLLATERAL, ...)`

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
| `uint256` | USD value of transaction |

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
| `_legoId` | `uint256` | Must be RIPE_LEGO_ID (1) |
| `_borrowAsset` | `address` | GREEN or SavingsGREEN |
| `_amount` | `uint256` | Amount to borrow |
| `_extraData` | `bytes32` | Ripe-specific data |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount borrowed |
| `uint256` | USD value of transaction |

#### Constraints

- Must not exceed `maxDebtRatio` after borrowing
- Validated by LevgVaultHelper.getMaxBorrowAmount()

#### Events Emitted

- `LevgVaultAction(op=BORROW, ...)`

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

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_legoId` | `uint256` | Must be RIPE_LEGO_ID (1) |
| `_paymentAsset` | `address` | GREEN or SavingsGREEN |
| `_paymentAmount` | `uint256` | Amount to repay |
| `_extraData` | `bytes32` | Ripe-specific data |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount repaid |
| `uint256` | USD value of transaction |

#### Events Emitted

- `LevgVaultAction(op=REPAY_DEBT, ...)`

## Yield Management Functions (Manager Only)

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

### `swapTokens`

Executes token swaps with slippage validation.

```vyper
@external
@nonreentrant
def swapTokens(
    _instructions: DynArray[SwapInstruction, MAX_SWAP_INSTRUCTIONS]
) -> (address, uint256, address, uint256, uint256):
```

#### Slippage Validation

For GREEN <-> USDC swaps:
- Validates against `usdcSlippageAllowed` and `greenSlippageAllowed`
- Uses LevgVaultHelper.performPostSwapValidation()
- Ensures debt ratio remains within limits

### `claimIncentives`

Claims protocol rewards.

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

## Configuration Functions (Switchboard Only)

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

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_vaultToken` | `address` | Collateral vault token address |
| `_legoId` | `uint256` | Lego ID for the vault |
| `_ripeVaultId` | `uint256` | Ripe vault ID for collateral tracking |
| `_shouldMaxWithdraw` | `bool` | Whether to max withdraw from previous vault |

#### Events Emitted

- `CollateralVaultTokenSet`

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

#### Events Emitted

- `LeverageVaultTokenSet`

### `setSlippagesAllowed`

Updates slippage tolerance for swaps.

```vyper
@external
def setSlippagesAllowed(_usdcSlippage: uint256, _greenSlippage: uint256):
```

#### Events Emitted

- `SlippagesSet`

### `setLevgVaultHelper`

Updates the helper contract address.

```vyper
@external
def setLevgVaultHelper(_levgVaultHelper: address):
```

#### Events Emitted

- `LevgVaultHelperSet`

### `setMaxDebtRatio`

Sets maximum debt ratio limit.

```vyper
@external
def setMaxDebtRatio(_ratio: uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_ratio` | `uint256` | Max debt ratio in basis points (max 30000 = 300%) |

#### Events Emitted

- `MaxDebtRatioSet`

## Manager Functions (Switchboard Only)

### `addManager` / `removeManager`

```vyper
@external
def addManager(_manager: address):

@external
def removeManager(_manager: address):
```

## Administration Functions

### `sweepLeftovers`

Emergency sweep of vault assets to governance.

```vyper
@external
@nonreentrant
def sweepLeftovers() -> uint256:
```

#### Access

Governance only

## Events

```vyper
event LevgVaultAction:
    op: indexed(uint256)       # Action type
    asset1: indexed(address)   # Primary asset
    asset2: address            # Secondary asset
    amount1: uint256           # Primary amount
    amount2: uint256           # Secondary amount
    usdValue: uint256          # USD value
    legoId: uint256            # Lego ID
    signer: indexed(address)   # Signer

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

## Leverage Strategy Example

### Opening a Leveraged Position

1. User deposits 1000 USDC
2. Manager deposits to collateral vault (e.g., Morpho)
3. Manager adds collateral to Ripe
4. Manager borrows GREEN against collateral
5. Manager swaps GREEN to USDC via Endaoment PSM
6. Manager deposits borrowed USDC to leverage vault
7. Repeat steps 3-6 for more leverage

### Leverage Math

With 300% max debt ratio:
- Initial capital: 1000 USDC
- Max borrowable: 3000 USDC worth of GREEN
- Total exposure: 4000 USDC
- Leverage multiple: 4x

### Closing a Leveraged Position

1. User requests withdrawal
2. Manager withdraws from leverage vault
3. Manager swaps USDC to GREEN
4. Manager repays debt
5. Manager removes collateral
6. Manager withdraws from collateral vault
7. Assets sent to user

## Security Considerations

### Debt Ratio Enforcement
- `maxDebtRatio` limits borrowing (default up to 300%)
- Validated on every borrow via LevgVaultHelper
- Cannot borrow if would exceed limit

### Slippage Protection
- GREEN/USDC swaps validated against configured limits
- Post-swap validation ensures acceptable slippage
- Prevents sandwich attacks on large trades

### Net Capital Tracking
- `netUserCapital` tracks user deposits minus withdrawals
- Used for debt ratio calculations
- Prevents manipulation through repeated deposit/withdraw

### Access Control
- Deposit/withdraw: Public (subject to VaultRegistry rules)
- Debt operations: Managers or Switchboard only
- Configuration: Switchboard only
- Emergency functions: Governance only

### Liquidation Risk
- Vault maintains positions in Ripe Protocol
- Subject to Ripe's liquidation mechanics
- Managers responsible for maintaining healthy ratios

## Integration with LevgVaultHelper

The vault relies on [LevgVaultHelper](LevgVaultHelper.md) for:
- `getTotalAssetsForUsdcVault()` - Calculate total assets for USDC vaults
- `getTotalAssetsForNonUsdcVault()` - Calculate for WETH/CBBTC vaults
- `getMaxBorrowAmount()` - Enforce debt ratio limits
- `performPostSwapValidation()` - Validate GREEN/USDC swap slippage
- `getCollateralBalance()` - Query collateral in Ripe
