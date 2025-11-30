# EarnVaultWallet Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/vaults/modules/EarnVaultWallet.vy)

## Overview

EarnVaultWallet is an internal Vyper module that manages yield positions, deposits, swaps, rewards, and performance fee calculations for [EarnVault](EarnVault.md). It handles all state related to yield farming strategies and manager permissions, providing the core yield management functionality that EarnVault exposes through its external interface.

**Core Responsibilities**:
- **Yield Position Tracking**: Maintains registry of active yield positions across protocols
- **Deposit/Withdrawal Execution**: Handles actual deposits and withdrawals through Lego partners
- **Yield Profit Calculation**: Tracks and calculates realized yield for fee collection
- **Swap Execution**: Enables token swaps for rebalancing strategies
- **Manager Registry**: Tracks authorized AI agent managers
- **Performance Fees**: Accumulates and tracks pending performance fees

This module is imported by EarnVault and its functions are called internally. It should not be deployed as a standalone contract.

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                          EarnVaultWallet Module                          |
|                    (Imported by EarnVault)                              |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Yield Position Registry                        | |
|  |                                                                   | |
|  |  vaultToLegoId     -->  Maps vault token to lego ID              | |
|  |  assets[]          -->  Index-based vault token tracking          | |
|  |  numAssets         -->  Total positions count                     | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Yield Tracking State                           | |
|  |                                                                   | |
|  |  lastUnderlyingBal    -->  Previous underlying balance           | |
|  |  pendingYieldRealized -->  Accumulated yield for fees            | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Manager Registry                               | |
|  |                                                                   | |
|  |  managers[]       -->  Index-based manager tracking               | |
|  |  indexOfManager   -->  Manager to index lookup                    | |
|  |  numManagers      -->  Total managers count                       | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Transient Storage                              | |
|  |                                                                   | |
|  |  deregVaultTokens    -->  Tokens to deregister post-tx           | |
|  |  feeVaultTokens      -->  Tokens with withdrawal fees            | |
|  +-------------------------------------------------------------------+ |
+-------------------------------------------------------------------------+
```

## Data Structures

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

### Yield Tracking
```vyper
lastUnderlyingBal: public(uint256)
# Last known underlying balance for yield calculation

pendingYieldRealized: public(uint256)
# Accumulated realized yield pending fee collection
```

### Yield Position Registry
```vyper
vaultToLegoId: public(HashMap[address, uint256])
# Maps vault token address to lego ID

assets: public(HashMap[uint256, address])
# Index to vault token address mapping

indexOfAsset: public(HashMap[address, uint256])
# Vault token address to index mapping

numAssets: public(uint256)
# Total number of yield positions
```

### Manager Registry
```vyper
managers: public(HashMap[uint256, address])
# Index to manager address mapping

indexOfManager: public(HashMap[address, uint256])
# Manager address to index mapping

numManagers: public(uint256)
# Total number of managers
```

### Transient Storage (Reset per Transaction)
```vyper
deregVaultTokens: transient(HashMap[uint256, address])
# Vault tokens to deregister after transaction

deregVaultTokenToId: transient(HashMap[address, uint256])
# Mapping for deregistration tracking

feeVaultTokens: transient(HashMap[uint256, address])
# Vault tokens with withdrawal fees

feeVaultTokenToId: transient(HashMap[address, uint256])
# Mapping for fee tracking
```

### Constants
```vyper
HUNDRED_PERCENT: constant(uint256) = 100_00  # 10,000 basis points
MAX_SWAP_INSTRUCTIONS: constant(uint256) = 5
MAX_TOKEN_PATH: constant(uint256) = 5
MAX_LEGOS: constant(uint256) = 10
MAX_PROOFS: constant(uint256) = 25
LEGO_BOOK_ID: constant(uint256) = 3
SWITCHBOARD_ID: constant(uint256) = 4
VAULT_REGISTRY_ID: constant(uint256) = 10
```

## Core Functions

### `depositForYield`

Deposits vault assets into a yield protocol through a Lego partner.

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
| `_legoId` | `uint256` | Lego partner ID for yield protocol |
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

#### Behavior

1. Validates caller is manager or Switchboard
2. Validates vault token is approved in VaultRegistry
3. Calculates any new yield from existing positions
4. Executes deposit through Lego partner
5. Registers new vault token if not already tracked
6. Updates `lastUnderlyingBal`
7. Emits `EarnVaultDeposit` event

### `withdrawFromYield`

Withdraws from a yield position through a Lego partner.

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

#### Behavior

1. Validates caller is manager or Switchboard
2. Calculates new yield before withdrawal
3. Executes withdrawal through Lego partner
4. Marks vault token for deregistration if fully withdrawn
5. Updates `lastUnderlyingBal`
6. Emits `EarnVaultWithdrawal` event

### `swapTokens`

Executes token swaps through DEX Lego partners for rebalancing.

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
| `_instructions` | `DynArray[SwapInstruction, 5]` | Array of swap instructions |

#### Returns

| Type | Description |
|------|-------------|
| `address` | First input token address |
| `uint256` | Total input amount |
| `address` | Final output token address |
| `uint256` | Total output amount |
| `uint256` | USD value of transaction |

#### Behavior

1. Validates caller is manager or Switchboard
2. Executes each swap instruction sequentially
3. Tracks cumulative input/output amounts
4. Emits `EarnVaultSwap` event

### `claimIncentives`

Claims protocol rewards/incentives from yield protocols.

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

#### Behavior

1. Validates caller is manager or Switchboard
2. Executes claim through Lego partner
3. Emits `EarnVaultRewardsClaim` event

## Yield Position Management Functions

### `updateYieldPosition`

Updates yield position tracking after external changes (e.g., manual intervention).

```vyper
@external
def updateYieldPosition(_vaultToken: address):
```

#### Access

Switchboard only

### `getClaimablePerformanceFees`

Returns accumulated performance fees pending collection.

```vyper
@view
@external
def getClaimablePerformanceFees() -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of fees claimable |

### `claimPerformanceFees`

Claims accumulated performance fees to governance.

```vyper
@external
@nonreentrant
def claimPerformanceFees() -> uint256:
```

#### Access

Switchboard only

#### Behavior

1. Calculates final pending fees
2. Resets `pendingYieldRealized` to zero
3. Transfers fees to governance address
4. Emits `PerformanceFeesClaimed` event

## Manager Registry Functions

### `addManager`

Registers a new AI agent as a manager.

```vyper
@external
def addManager(_manager: address):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_manager` | `address` | Address to register as manager |

#### Access

Switchboard only

#### Behavior

1. Validates manager is not already registered
2. Adds to index-based registry
3. Increments `numManagers`

### `removeManager`

Removes a manager from the registry.

```vyper
@external
def removeManager(_manager: address):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_manager` | `address` | Address to remove |

#### Access

Switchboard only

#### Behavior

1. Uses swap-and-pop pattern for efficient removal
2. Decrements `numManagers`

## Internal Functions

### `_onReceiveVaultFunds`

Called when vault receives new deposits, optionally auto-deposits to yield.

```vyper
@internal
def _onReceiveVaultFunds(
    _targetVaultToken: address,
    _depositor: address,
    _vaultRegistry: address
) -> uint256:
```

#### Behavior

1. Checks if auto-deposit is enabled in VaultRegistry
2. If enabled, deposits to default target vault token
3. Returns amount auto-deposited

### `_prepareRedemption`

Prepares for user redemption by withdrawing from yield positions if needed.

```vyper
@internal
def _prepareRedemption(
    _requestedAmount: uint256,
    _vaultRegistry: address,
    _legoBook: address
) -> uint256:
```

#### Behavior

1. Checks idle balance vs requested amount
2. If insufficient, withdraws from yield positions
3. Respects `redemptionBuffer` setting
4. Respects `minYieldWithdrawAmount` to avoid dust
5. Returns actual amount prepared

### `_calcNewYieldAndGetUnderlying`

Calculates new yield realized and returns underlying balances.

```vyper
@internal
def _calcNewYieldAndGetUnderlying(
    _vaultRegistry: address,
    _legoBook: address,
    _appraiser: address
) -> (uint256, uint256, address):
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Maximum total underlying assets |
| `uint256` | Safe total underlying assets |
| `address` | Vault token with max balance |

#### Behavior

1. Iterates through all yield positions
2. Calculates yield profit for each position
3. Applies performance fee to yield
4. Accumulates in `pendingYieldRealized`
5. Updates `lastUnderlyingBal`

### `_getUnderlyingYieldBalances`

Returns underlying balances across all yield positions.

```vyper
@view
@internal
def _getUnderlyingYieldBalances() -> (uint256, uint256, address):
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Maximum total underlying |
| `uint256` | Safe total underlying |
| `address` | Vault token with maximum balance |

## Events

```vyper
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

### Yield Position Registry

The module maintains an efficient index-based registry of yield positions:

1. **Registration**: New vault tokens are registered on first deposit
2. **Tracking**: `vaultToLegoId` maps tokens to their Lego partner
3. **Deregistration**: Fully withdrawn positions are marked for removal
4. **Efficient Lookup**: Bidirectional index mapping for O(1) access

### Yield Profit Calculation

For each yield position:

1. **Non-Rebasing Assets**: Compares current price-per-share to last recorded
2. **Rebasing Assets**: Compares current balance to expected balance
3. **Max Yield Cap**: Applies `maxYieldIncrease` limit to prevent manipulation
4. **Fee Application**: Calculates performance fee on realized yield
5. **Accumulation**: Adds to `pendingYieldRealized` for later collection

### Transient Storage Usage

Uses Vyper's transient storage for within-transaction state:

- **deregVaultTokens**: Tracks positions to remove after transaction
- **feeVaultTokens**: Tracks positions with withdrawal fees for accounting

This ensures clean state between transactions while enabling complex multi-step operations.

### Manager Authorization

Managers are authorized to:
- Execute yield deposits/withdrawals
- Perform token swaps
- Claim protocol incentives

They cannot:
- Add/remove other managers
- Claim performance fees
- Modify vault configuration

## Security Considerations

### Access Control
- All yield operations require manager or Switchboard authorization
- Manager registry changes require Switchboard only
- Performance fee claims require Switchboard only

### Yield Manipulation Protection
- `maxYieldIncrease` cap prevents flash loan attacks
- Safe conversion uses weighted average to prevent short-term manipulation
- VaultRegistry approval required for all vault tokens

### Reentrancy Protection
- All external functions use `@nonreentrant` modifier
- State updates before external calls

### Position Tracking Integrity
- Index-based registry with bidirectional mapping
- Deregistration uses swap-and-pop for efficiency
- Transient storage for within-transaction cleanup
