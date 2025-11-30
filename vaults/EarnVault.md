# EarnVault Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/vaults/EarnVault.vy)

## Overview

EarnVault is an ERC4626-compliant autopilot vault for yield generation, managed by AI agents and enforced by onchain rules. Users deposit assets and receive shares representing their proportional ownership of the vault's total assets. The vault manages multiple yield positions across various DeFi protocols through the Lego Partner system.

**Core Features**:
- **ERC4626 Compliance**: Standard vault interface with deposit/withdraw/mint/redeem functions
- **Multi-Position Yield**: Manages yield across multiple protocols simultaneously
- **AI Agent Management**: Authorized managers (AI agents) execute yield strategies
- **Performance Fees**: Protocol collects fees on realized yield profits
- **Auto-Deposit**: Optionally routes new deposits directly into yield positions
- **Safe Conversions**: Provides both optimistic and conservative share/asset conversions

The vault uses a modular architecture with [EarnVaultWallet](EarnVaultWallet.md) handling yield position management and [VaultErc20Token](VaultErc20Token.md) providing the share token implementation.

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                              EarnVault                                   |
|                        (ERC4626 Compliant)                              |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                       Share Management                            | |
|  |                                                                   | |
|  |  deposit() / mint()     -->  Mint shares, optional auto-deposit   | |
|  |  withdraw() / redeem()  -->  Burn shares, withdraw from yield     | |
|  |  convertToShares/Assets -->  Safe or optimistic conversions       | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Yield Position Management                      | |
|  |                     (via EarnVaultWallet)                         | |
|  |                                                                   | |
|  |  depositForYield()   -->  Deposit into yield protocols            | |
|  |  withdrawFromYield() -->  Withdraw from yield protocols           | |
|  |  swapTokens()        -->  Rebalance between positions             | |
|  |  claimIncentives()   -->  Claim protocol rewards                  | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                      Manager System                               | |
|  |                                                                   | |
|  |  * AI agents registered as managers                               | |
|  |  * Execute yield strategies within rules                          | |
|  |  * Controlled by Switchboard governance                           | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    External Dependencies                          | |
|  |                                                                   | |
|  |  VaultRegistry  -->  Configuration, approved vault tokens         | |
|  |  LegoBook       -->  DeFi protocol integrations                   | |
|  |  Appraiser      -->  USD value calculations                       | |
|  |  Switchboard    -->  Governance and permissions                   | |
|  +-------------------------------------------------------------------+ |
+-------------------------------------------------------------------------+
```

## Data Structures

### VaultToken Struct
Tracks information about yield positions:
```vyper
struct VaultToken:
    legoId: uint256         # Lego partner ID
    underlyingAsset: address # Asset deposited in vault
    decimals: uint256       # Token decimals
    isRebasing: bool        # Whether token rebases
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

### SwapInstruction Struct
Defines parameters for token swaps:
```vyper
struct SwapInstruction:
    legoId: uint256                              # Lego partner to use
    amountIn: uint256                            # Input amount
    minAmountOut: uint256                        # Minimum output required
    tokenPath: DynArray[address, MAX_TOKEN_PATH] # Token swap path
    poolPath: DynArray[address, MAX_TOKEN_PATH]  # Pool identifiers
```

## State Variables

### Yield Tracking
- `lastUnderlyingBal: uint256` - Last known underlying balance for yield calculation
- `pendingYieldRealized: uint256` - Accumulated realized yield pending fee collection

### Yield Position Registry
- `vaultToLegoId: HashMap[address, uint256]` - Maps vault token to lego ID
- `assets: HashMap[uint256, address]` - Index to vault token mapping
- `indexOfAsset: HashMap[address, uint256]` - Vault token to index mapping
- `numAssets: uint256` - Total number of yield positions

### Manager Registry
- `managers: HashMap[uint256, address]` - Index to manager mapping
- `indexOfManager: HashMap[address, uint256]` - Manager to index mapping
- `numManagers: uint256` - Total number of managers

### Constants
- `HUNDRED_PERCENT: uint256 = 100_00` - 100.00% in basis points
- `MAX_SWAP_INSTRUCTIONS: uint256 = 5` - Maximum swap steps
- `MAX_TOKEN_PATH: uint256 = 5` - Maximum tokens in swap path
- `MAX_LEGOS: uint256 = 10` - Maximum lego partners per action
- `MAX_PROOFS: uint256 = 25` - Maximum merkle proofs for claims

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

#### Access

External, non-reentrant. Requires deposits enabled in VaultRegistry.

#### Events Emitted

- `Deposit(sender, owner, assets, shares)`

### `depositWithMinAmountOut`

Deposits with slippage protection on minimum shares received.

```vyper
@external
@nonreentrant
def depositWithMinAmountOut(
    _assets: uint256,
    _minAmountOut: uint256,
    _receiver: address
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_assets` | `uint256` | Amount of underlying asset to deposit |
| `_minAmountOut` | `uint256` | Minimum shares to receive |
| `_receiver` | `address` | Address to receive shares |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of shares minted |

### `mint`

Mints exact amount of shares, pulling required assets.

```vyper
@external
@nonreentrant
def mint(_shares: uint256, _receiver: address) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_shares` | `uint256` | Amount of shares to mint |
| `_receiver` | `address` | Address to receive shares |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of assets deposited |

### `maxDeposit` / `maxMint`

Returns maximum deposit/mint amounts for a receiver.

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

#### Events Emitted

- `Withdraw(sender, receiver, owner, assets, shares)`

### `redeem`

Redeems exact amount of shares for proportional assets.

```vyper
@external
@nonreentrant
def redeem(_shares: uint256, _receiver: address, _owner: address) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_shares` | `uint256` | Amount of shares to redeem |
| `_receiver` | `address` | Address to receive assets |
| `_owner` | `address` | Owner of shares to burn |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of assets received |

### `redeemWithMinAmountOut`

Redeems with slippage protection on minimum assets received.

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

Returns maximum withdrawal/redemption amounts for an owner.

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

- `convertToShares`: Uses maximum underlying balance (optimistic)
- `convertToSharesSafe`: Uses safe/conservative underlying balance

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

- `convertToAssets`: Uses maximum underlying balance (optimistic)
- `convertToAssetsSafe`: Uses safe/conservative underlying balance

## Asset & Yield Functions

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

- `totalAssets`: Returns maximum total assets
- `getTotalAssets`: Parameterized to return max or safe amount

### `getClaimablePerformanceFees`

Returns accrued performance fees pending collection.

```vyper
@view
@external
def getClaimablePerformanceFees() -> uint256:
```

## Yield Management Functions (Manager Only)

### `depositForYield`

Deposits vault assets into a yield protocol.

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
| `_asset` | `address` | Asset to deposit (must be vault's underlying) |
| `_vaultAddr` | `address` | Target yield vault address |
| `_amount` | `uint256` | Amount to deposit |
| `_extraData` | `bytes32` | Protocol-specific data |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of asset deposited |
| `address` | Vault token received |
| `uint256` | Amount of vault tokens received |
| `uint256` | USD value of transaction |

#### Access

Managers or Switchboard only. Vault token must be approved in VaultRegistry.

#### Events Emitted

- `EarnVaultDeposit`

### `withdrawFromYield`

Withdraws from a yield position.

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

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_legoId` | `uint256` | Lego partner ID |
| `_vaultToken` | `address` | Vault token to redeem |
| `_amount` | `uint256` | Amount to withdraw |
| `_extraData` | `bytes32` | Protocol-specific data |
| `_isSpecialTx` | `bool` | Whether this is a special transaction |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of vault tokens burned |
| `address` | Underlying asset received |
| `uint256` | Amount of underlying received |
| `uint256` | USD value of transaction |

#### Events Emitted

- `EarnVaultWithdrawal`

### `swapTokens`

Executes token swaps through DEX protocols.

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
| `address` | Input token address |
| `uint256` | Input amount |
| `address` | Output token address |
| `uint256` | Output amount |
| `uint256` | USD value of transaction |

#### Events Emitted

- `EarnVaultSwap`

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

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_legoId` | `uint256` | Lego partner ID |
| `_rewardToken` | `address` | Expected reward token |
| `_rewardAmount` | `uint256` | Expected amount |
| `_proofs` | `DynArray[bytes32, 25]` | Merkle proofs for claim |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Reward amount claimed |
| `uint256` | USD value of rewards |

#### Events Emitted

- `EarnVaultRewardsClaim`

## Manager & Administration Functions

### `addManager` / `removeManager`

Registers or removes AI agent managers.

```vyper
@external
def addManager(_manager: address):

@external
def removeManager(_manager: address):
```

#### Access

Switchboard only

### `updateYieldPosition`

Updates yield position tracking after external changes.

```vyper
@external
def updateYieldPosition(_vaultToken: address):
```

#### Access

Switchboard only

### `claimPerformanceFees`

Claims accumulated performance fees to governance.

```vyper
@external
@nonreentrant
def claimPerformanceFees() -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of fees claimed |

#### Access

Switchboard only

#### Events Emitted

- `PerformanceFeesClaimed`

### `sweepLeftovers`

Emergency function to sweep vault assets to governance.

```vyper
@external
@nonreentrant
def sweepLeftovers() -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount swept |

#### Access

Governance only

#### Events Emitted

- `LeftoversSwept`

## Events

```vyper
event Deposit:
    sender: indexed(address)
    owner: indexed(address)
    assets: uint256
    shares: uint256

event Withdraw:
    sender: indexed(address)
    receiver: indexed(address)
    owner: indexed(address)
    assets: uint256
    shares: uint256

event LeftoversSwept:
    amount: uint256
    recipient: indexed(address)

event EarnVaultDeposit:
    asset: indexed(address)
    assetAmountDeposited: uint256
    assetAmountAdjusted: uint256
    vaultToken: indexed(address)
    vaultTokenReceived: uint256
    vaultTokenExpected: uint256
    usdValue: uint256
    legoId: uint256
    signer: indexed(address)

event EarnVaultWithdrawal:
    vaultToken: indexed(address)
    vaultTokenBurned: uint256
    underlyingAsset: indexed(address)
    underlyingAmountReceived: uint256
    usdValue: uint256
    legoId: uint256
    signer: indexed(address)

event EarnVaultSwap:
    tokenIn: indexed(address)
    tokenInAmount: uint256
    tokenOut: indexed(address)
    tokenOutAmount: uint256
    usdValue: uint256
    swapFee: uint256
    legoId: uint256
    signer: indexed(address)

event EarnVaultRewardsClaim:
    rewardToken: indexed(address)
    rewardAmount: uint256
    usdValue: uint256
    legoId: uint256
    signer: indexed(address)

event PerformanceFeesClaimed:
    pendingFees: uint256
```

## Internal Mechanisms

### Share/Asset Conversions

The vault provides two conversion modes:
1. **Optimistic (Max)**: Uses maximum underlying balance from all yield positions
2. **Safe (Conservative)**: Uses weighted average price per share to prevent manipulation

### Auto-Deposit on Deposit

When configured in VaultRegistry:
1. User deposits trigger automatic deployment to default yield position
2. Shares are minted based on post-deployment total assets
3. Reduces idle capital in vault

### Redemption Process

When users redeem:
1. Check if sufficient idle balance exists
2. If not, pull from yield positions proportionally
3. Respect `redemptionBuffer` setting (minimum % kept in yield)
4. Respect `minYieldWithdrawAmount` to avoid dust withdrawals

### Performance Fee Collection

1. Yield profits are tracked per position
2. Accumulated in `pendingYieldRealized`
3. Collected by governance via `claimPerformanceFees()`
4. Fee percentage configured per vault in VaultRegistry

## Security Considerations

### Access Control
- Deposit/withdraw: Public (subject to VaultRegistry rules)
- Yield operations: Managers or Switchboard only
- Configuration: Switchboard only
- Emergency functions: Governance only

### Reentrancy Protection
- All state-changing functions use `@nonreentrant`

### Slippage Protection
- `depositWithMinAmountOut` and `redeemWithMinAmountOut` enforce minimums
- Swap instructions include `minAmountOut` parameters

### Vault Token Approval
- Only VaultRegistry-approved tokens can be used for yield
- Prevents unauthorized protocol integrations

### Emergency Controls
- `sweepLeftovers` for emergency asset recovery
- VaultRegistry can freeze vault operations

## Integration with VaultRegistry

The vault relies on [VaultRegistry](../registries/VaultRegistry.md) for:
- `canDeposit` / `canWithdraw` - Enable/disable operations
- `maxDepositAmount` - Deposit caps
- `isVaultOpsFrozen` - Emergency freeze
- `performanceFee` - Fee percentage on yield
- `redemptionBuffer` - Minimum % to keep in yield positions
- `minYieldWithdrawAmount` - Minimum per-position withdrawal
- `shouldAutoDeposit` - Auto-deploy deposits to yield
- `defaultTargetVaultToken` - Default yield position for auto-deposit
- `isApprovedVaultToken` - Whitelist of allowed yield protocols
- `shouldEnforceAllowlist` - User allowlist enforcement
