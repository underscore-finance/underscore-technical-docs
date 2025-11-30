# AgentSenderSpecial Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/core/agent/AgentSenderSpecial.vy)

## Overview

AgentSenderSpecial is a specialized agent sender contract that provides pre-built workflow functions for common multi-step DeFi operations. Unlike AgentSenderGeneric which handles individual actions, this contract bundles related operations into single transaction workflows with automatic output chaining.

**Core Features**:
- **Collateralized Borrowing Workflow**: Add collateral + borrow + swap + yield in one transaction
- **Debt Repayment Workflow**: Deleverage + withdraw + swap + repay + remove collateral
- **Yield Rebalancing Workflow**: Withdraw + swap + deposit/transfer across protocols
- **Rewards Harvesting Workflow**: Claim + swap + deposit into yield or collateral
- **Signature-Based Authentication**: EIP-712 compliant signatures for delegated operations
- **Automatic Output Chaining**: Previous operation outputs automatically feed next operations

## Data Structures

### CollateralAsset

```vyper
struct CollateralAsset:
    vaultId: uint256
    asset: address
    amount: uint256
```

### DeleverageAsset

```vyper
struct DeleverageAsset:
    vaultId: uint256
    asset: address
    targetRepayAmount: uint256
```

### DepositYieldPosition

```vyper
struct DepositYieldPosition:
    legoId: uint256
    asset: address
    amount: uint256
    vaultAddr: address
```

### WithdrawYieldPosition

```vyper
struct WithdrawYieldPosition:
    legoId: uint256
    vaultToken: address
    vaultTokenAmount: uint256
```

### TransferData

```vyper
struct TransferData:
    asset: address
    amount: uint256
    recipient: address
```

### Signature

```vyper
struct Signature:
    signature: Bytes[65]
    nonce: uint256
    expiration: uint256
```

### Action Codes

| Code | Workflow |
|------|----------|
| 100 | Add Collateral and Borrow |
| 101 | Repay and Withdraw |
| 102 | Rebalance Yield Positions with Swap |
| 103 | Claim Incentives and Swap |

## Constants

- `MAX_COLLATERAL_ASSETS: uint256 = 10` - Maximum collateral assets per operation
- `MAX_DELEVERAGE_ASSETS: uint256 = 25` - Maximum deleverage assets
- `MAX_YIELD_POSITIONS: uint256 = 25` - Maximum yield positions
- `MAX_INSTRUCTIONS: uint256 = 15` - Maximum batch instructions
- `MAX_SWAP_INSTRUCTIONS: uint256 = 5` - Maximum swap instructions
- `MAX_PROOFS: uint256 = 25` - Maximum merkle proofs

## State Variables

| Name | Type | Description |
|------|------|-------------|
| `currentNonce` | `HashMap[address, uint256]` | Per-wallet nonce for replay protection |
| `UNDY_HQ` | `address` (immutable) | UndyHq registry address |
| `RIPE_GREEN_TOKEN` | `address` (immutable) | Ripe green token address |
| `RIPE_SAVINGS_GREEN` | `address` (immutable) | Ripe savings green address |

## Constructor

### `__init__`

```vyper
@deploy
def __init__(
    _undyHq: address,
    _owner: address,
    _minTimeLock: uint256,
    _maxTimeLock: uint256,
    _greenToken: address,
    _savingsGreen: address,
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyHq` | `address` | UndyHq registry address |
| `_owner` | `address` | Contract owner |
| `_minTimeLock` | `uint256` | Minimum timelock duration |
| `_maxTimeLock` | `uint256` | Maximum timelock duration |
| `_greenToken` | `address` | Ripe green token address |
| `_savingsGreen` | `address` | Ripe savings green address |

## Specialized Workflow Functions

### `addCollateralAndBorrow`

Multi-step workflow: Add collateral → Borrow → Swap (optional) → Deposit to Yield (optional)

```vyper
@external
def addCollateralAndBorrow(
    _agentWrapper: address,
    _userWallet: address,
    _debtLegoId: uint256,
    _addCollateralAssets: DynArray[CollateralAsset, MAX_COLLATERAL_ASSETS] = [],
    _greenBorrowAmount: uint256 = 0,
    _wantsSavingsGreen: bool = True,
    _shouldEnterStabPool: bool = False,
    _swapInstructions: DynArray[Wallet.SwapInstruction, MAX_SWAP_INSTRUCTIONS] = [],
    _yieldPosition: DepositYieldPosition = empty(DepositYieldPosition),
    _sig: Signature = empty(Signature),
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentWrapper` | `address` | AgentWrapper contract address |
| `_userWallet` | `address` | User wallet to operate on |
| `_debtLegoId` | `uint256` | Debt protocol Lego ID |
| `_addCollateralAssets` | `DynArray[CollateralAsset, 10]` | Collateral assets to add |
| `_greenBorrowAmount` | `uint256` | Amount to borrow |
| `_wantsSavingsGreen` | `bool` | Use savings green vs regular green |
| `_shouldEnterStabPool` | `bool` | Enter stability pool |
| `_swapInstructions` | `DynArray[SwapInstruction, 5]` | Optional swap after borrow |
| `_yieldPosition` | `DepositYieldPosition` | Optional yield deposit |
| `_sig` | `Signature` | Authentication signature |

#### Workflow Steps

1. Authenticate access (action code 100)
2. Add collateral for each asset in `_addCollateralAssets`
3. Borrow green tokens if `_greenBorrowAmount != 0`
4. Execute swap if instructions provided (chains borrow output)
5. Deposit to yield if position specified (chains swap output)

### `repayAndWithdraw`

Multi-step workflow: Deleverage → Withdraw from Yield → Swap → Repay Debt → Remove Collateral

```vyper
@external
def repayAndWithdraw(
    _agentWrapper: address,
    _userWallet: address,
    _debtLegoId: uint256,
    _deleverageAssets: DynArray[DeleverageAsset, MAX_DELEVERAGE_ASSETS] = [],
    _yieldPosition: WithdrawYieldPosition = empty(WithdrawYieldPosition),
    _swapInstructions: DynArray[Wallet.SwapInstruction, MAX_SWAP_INSTRUCTIONS] = [],
    _repayAsset: address = empty(address),
    _repayAmount: uint256 = max_value(uint256),
    _removeCollateralAssets: DynArray[CollateralAsset, MAX_COLLATERAL_ASSETS] = [],
    _sig: Signature = empty(Signature),
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentWrapper` | `address` | AgentWrapper contract address |
| `_userWallet` | `address` | User wallet to operate on |
| `_debtLegoId` | `uint256` | Debt protocol Lego ID |
| `_deleverageAssets` | `DynArray[DeleverageAsset, 25]` | Assets to deleverage |
| `_yieldPosition` | `WithdrawYieldPosition` | Position to withdraw from |
| `_swapInstructions` | `DynArray[SwapInstruction, 5]` | Optional swap instructions |
| `_repayAsset` | `address` | Asset to repay debt with |
| `_repayAmount` | `uint256` | Amount to repay |
| `_removeCollateralAssets` | `DynArray[CollateralAsset, 10]` | Collateral to remove |
| `_sig` | `Signature` | Authentication signature |

#### Workflow Steps

1. Authenticate access (action code 101)
2. Deleverage assets if provided
3. Withdraw from yield position (chains output)
4. Execute swap if instructions provided (chains withdraw output)
5. Repay debt if asset specified (chains swap output)
6. Remove collateral assets

### `rebalanceYieldPositionsWithSwap`

Multi-step workflow: Withdraw from Yield → Swap → Deposit to Yield OR Transfer

```vyper
@external
def rebalanceYieldPositionsWithSwap(
    _agentWrapper: address,
    _userWallet: address,
    _withdrawFrom: DynArray[WithdrawYieldPosition, MAX_YIELD_POSITIONS] = [],
    _swapInstructions: DynArray[Wallet.SwapInstruction, MAX_SWAP_INSTRUCTIONS] = [],
    _depositTo: DynArray[DepositYieldPosition, MAX_YIELD_POSITIONS] = [],
    _transferTo: DynArray[TransferData, MAX_COLLATERAL_ASSETS] = [],
    _sig: Signature = empty(Signature),
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentWrapper` | `address` | AgentWrapper contract address |
| `_userWallet` | `address` | User wallet to operate on |
| `_withdrawFrom` | `DynArray[WithdrawYieldPosition, 25]` | Positions to withdraw from |
| `_swapInstructions` | `DynArray[SwapInstruction, 5]` | Optional swap instructions |
| `_depositTo` | `DynArray[DepositYieldPosition, 25]` | Positions to deposit to |
| `_transferTo` | `DynArray[TransferData, 10]` | Transfers (mutually exclusive with deposits) |
| `_sig` | `Signature` | Authentication signature |

#### Workflow Steps

1. Authenticate access (action code 102)
2. Withdraw from all yield positions in `_withdrawFrom`
3. Execute swap if instructions provided
4. Either deposit to yield positions OR transfer (mutually exclusive)

### `claimIncentivesAndSwap`

Multi-step workflow: Claim Rewards → Swap → Deposit to Yield → Add Collateral

```vyper
@external
def claimIncentivesAndSwap(
    _agentWrapper: address,
    _userWallet: address,
    _rewardLegoId: uint256 = 0,
    _rewardToken: address = empty(address),
    _rewardAmount: uint256 = max_value(uint256),
    _rewardProofs: DynArray[bytes32, MAX_PROOFS] = [],
    _swapInstructions: DynArray[Wallet.SwapInstruction, MAX_SWAP_INSTRUCTIONS] = [],
    _depositTo: DynArray[DepositYieldPosition, MAX_YIELD_POSITIONS] = [],
    _debtLegoId: uint256 = 0,
    _addCollateralAssets: DynArray[CollateralAsset, MAX_COLLATERAL_ASSETS] = [],
    _sig: Signature = empty(Signature),
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentWrapper` | `address` | AgentWrapper contract address |
| `_userWallet` | `address` | User wallet to operate on |
| `_rewardLegoId` | `uint256` | Rewards protocol Lego ID |
| `_rewardToken` | `address` | Reward token to claim |
| `_rewardAmount` | `uint256` | Amount to claim |
| `_rewardProofs` | `DynArray[bytes32, 25]` | Merkle proofs for claim |
| `_swapInstructions` | `DynArray[SwapInstruction, 5]` | Optional swap instructions |
| `_depositTo` | `DynArray[DepositYieldPosition, 25]` | Positions to deposit to |
| `_debtLegoId` | `uint256` | Debt protocol for collateral |
| `_addCollateralAssets` | `DynArray[CollateralAsset, 10]` | Collateral assets to add |
| `_sig` | `Signature` | Authentication signature |

#### Workflow Steps

1. Authenticate access (action code 103)
2. Claim incentives if reward token specified
3. Execute swap if instructions provided
4. Deposit to yield positions
5. Add collateral assets if debt lego specified

## Nonce Management

### `incrementNonce`

Manually increment a wallet's nonce.

```vyper
@external
def incrementNonce(_userWallet: address):
```

#### Access

Owner only

### `getNonce`

Get current nonce for a wallet.

```vyper
@view
@external
def getNonce(_userWallet: address) -> uint256:
```

## Events

### `NonceIncremented`

```vyper
event NonceIncremented:
    userWallet: address
    oldNonce: uint256
    newNonce: uint256
```

## Security Considerations

### Signature Validation
- EIP-712 compliant signatures
- Signature malleability prevention (s value check)
- Expiration timestamp validation
- Nonce-based replay protection

### Access Control
- Owner can call directly without signature
- Non-owners must provide valid signature from owner
- Signature must match expected owner address
- Empty signature required when owner calls directly

### Output Chaining
- Previous operation outputs automatically chain to next operation inputs
- Balance checks ensure wallet has sufficient funds
- Min of specified amount and wallet balance used

## Common Integration Patterns

### Leveraged Position Opening
```python
# Add collateral and borrow in one transaction
agent_special.addCollateralAndBorrow(
    agent_wrapper.address,
    user_wallet.address,
    debt_lego_id,
    [CollateralAsset(vault_id=1, asset=weth.address, amount=1e18)],
    green_borrow_amount=1000e18,
    _wantsSavingsGreen=True,
    sender=owner
)
```

### Position Unwinding
```python
# Repay debt and withdraw collateral in one transaction
agent_special.repayAndWithdraw(
    agent_wrapper.address,
    user_wallet.address,
    debt_lego_id,
    _repayAsset=green_token.address,
    _repayAmount=1000e18,
    _removeCollateralAssets=[CollateralAsset(vault_id=1, asset=weth.address, amount=1e18)],
    sender=owner
)
```

### Yield Rebalancing
```python
# Move funds from one yield protocol to another with swap
agent_special.rebalanceYieldPositionsWithSwap(
    agent_wrapper.address,
    user_wallet.address,
    _withdrawFrom=[WithdrawYieldPosition(lego_id=1, vault_token=aave_token.address, vault_token_amount=1e18)],
    _swapInstructions=swap_instructions,
    _depositTo=[DepositYieldPosition(lego_id=2, asset=usdc.address, amount=0, vault_addr=compound_vault.address)],
    sender=owner
)
```

## Testing

For test examples, see: [`tests/core/agent/`](../../../tests/core/agent/)
