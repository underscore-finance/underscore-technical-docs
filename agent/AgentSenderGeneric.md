# AgentSenderGeneric Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/core/agent/AgentSenderGeneric.vy)

## Overview

AgentSenderGeneric is a generic agent sender contract that enables batch actions and signature verification for UserWallet operations. It acts as an intermediary between external callers and the AgentWrapper, providing EIP-712 signature-based authentication for delegated wallet operations.

**Core Features**:
- **Signature-Based Authentication**: EIP-712 compliant signatures for secure delegated operations
- **Batch Actions**: Execute multiple wallet operations in a single transaction
- **Nonce Management**: Replay protection through per-wallet nonce tracking
- **Full Action Coverage**: Support for transfers, yield, swaps, debt, liquidity, and rewards

## Data Structures

### Signature

```vyper
struct Signature:
    signature: Bytes[65]
    nonce: uint256
    expiration: uint256
```

### ActionInstruction

```vyper
struct ActionInstruction:
    usePrevAmountOut: bool     # Use output from previous instruction as amount
    action: uint8              # Action type (see Action Codes below)
    legoId: uint16             # Protocol/Lego ID
    asset: address             # Primary asset/token
    target: address            # Varies by action type
    amount: uint256            # Primary amount (max_value = "all")
    asset2: address            # Secondary asset
    amount2: uint256           # Secondary amount or toLegoId
    minOut1: uint256           # Minimum output 1
    minOut2: uint256           # Minimum output 2
    tickLower: int24           # For concentrated liquidity
    tickUpper: int24           # For concentrated liquidity
    extraData: bytes32         # Protocol-specific data
    auxData: bytes32           # Packed auxiliary data
    swapInstructions: DynArray[SwapInstruction, 5]
    proofs: DynArray[bytes32, 25]
```

### Action Codes

| Code | Action | Description |
|------|--------|-------------|
| 1 | Transfer | Transfer funds to recipient |
| 2 | WETH→ETH | Convert wrapped ETH to ETH |
| 3 | ETH→WETH | Convert ETH to wrapped ETH |
| 10 | Deposit Yield | Deposit into yield protocol |
| 11 | Withdraw Yield | Withdraw from yield protocol |
| 12 | Rebalance Yield | Move between yield protocols |
| 20 | Swap | Execute token swap |
| 21 | Mint/Redeem | Mint or redeem synthetic assets |
| 22 | Confirm Mint/Redeem | Confirm pending mint/redeem |
| 30 | Add Liquidity | Add to liquidity pool |
| 31 | Remove Liquidity | Remove from liquidity pool |
| 32 | Add Concentrated Liq | Add concentrated liquidity |
| 33 | Remove Concentrated Liq | Remove concentrated liquidity |
| 40 | Add Collateral | Add collateral to debt position |
| 41 | Remove Collateral | Remove collateral |
| 42 | Borrow | Borrow against collateral |
| 43 | Repay | Repay debt |
| 50 | Claim Rewards | Claim protocol incentives |

## Constants

- `MAX_INSTRUCTIONS: uint256 = 15` - Maximum batch instructions
- `MAX_SWAP_INSTRUCTIONS: uint256 = 5` - Maximum swap instructions
- `MAX_PROOFS: uint256 = 25` - Maximum merkle proofs

## State Variables

| Name | Type | Description |
|------|------|-------------|
| `currentNonce` | `HashMap[address, uint256]` | Per-wallet nonce for replay protection |

## Constructor

### `__init__`

```vyper
@deploy
def __init__(
    _undyHq: address,
    _owner: address,
    _minTimeLock: uint256,
    _maxTimeLock: uint256,
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyHq` | `address` | UndyHq registry address |
| `_owner` | `address` | Contract owner |
| `_minTimeLock` | `uint256` | Minimum timelock duration |
| `_maxTimeLock` | `uint256` | Maximum timelock duration |

## Transfer Functions

### `transferFunds`

Transfer assets from a user wallet.

```vyper
@external
def transferFunds(
    _agentWrapper: address,
    _userWallet: address,
    _recipient: address,
    _asset: address = empty(address),
    _amount: uint256 = max_value(uint256),
    _isCheque: bool = False,
    _sig: Signature = empty(Signature),
) -> (uint256, uint256):
```

## Yield Functions

### `depositForYield`

Deposit assets into a yield protocol.

```vyper
@external
def depositForYield(
    _agentWrapper: address,
    _userWallet: address,
    _legoId: uint256,
    _asset: address,
    _vaultAddr: address = empty(address),
    _amount: uint256 = max_value(uint256),
    _extraData: bytes32 = empty(bytes32),
    _sig: Signature = empty(Signature),
) -> (uint256, address, uint256, uint256):
```

### `withdrawFromYield`

Withdraw assets from a yield protocol.

### `rebalanceYieldPosition`

Move assets between yield protocols.

## Swap Functions

### `swapTokens`

Execute token swaps.

```vyper
@external
def swapTokens(
    _agentWrapper: address,
    _userWallet: address,
    _swapInstructions: DynArray[SwapInstruction, 5],
    _sig: Signature = empty(Signature),
) -> (address, uint256, address, uint256, uint256):
```

### `mintOrRedeemAsset`

Mint or redeem synthetic assets.

### `confirmMintOrRedeemAsset`

Confirm pending mint/redeem operations.

## Debt Management Functions

### `addCollateral`

Add collateral to a debt position.

### `removeCollateral`

Remove collateral from a debt position.

### `borrow`

Borrow against collateral.

### `repayDebt`

Repay borrowed debt.

## Liquidity Functions

### `addLiquidity`

Add liquidity to a pool.

### `removeLiquidity`

Remove liquidity from a pool.

### `addLiquidityConcentrated`

Add concentrated liquidity (Uniswap V3 style).

### `removeLiquidityConcentrated`

Remove concentrated liquidity.

## Rewards Functions

### `claimIncentives`

Claim protocol rewards.

## Batch Actions

### `performBatchActions`

Execute multiple operations in sequence.

```vyper
@external
def performBatchActions(
    _agentWrapper: address,
    _userWallet: address,
    _instructions: DynArray[ActionInstruction, 15],
    _sig: Signature = empty(Signature),
) -> bool:
```

#### Notes

- Instructions execute sequentially
- `usePrevAmountOut` chains output to next instruction's input
- Supports all action types via action codes

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

## Testing

For test examples, see: [`tests/core/agent/test_agent_batch_actions.py`](../../../tests/core/agent/test_agent_batch_actions.py)
