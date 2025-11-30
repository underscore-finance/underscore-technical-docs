# EarnVaultAgent Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/core/agent/EarnVaultAgent.vy)

## Overview

EarnVaultAgent is a specialized agent wrapper for Earn Vault yield operations. It provides a focused interface for yield deposits, withdrawals, swaps, reward claims, and batch actions specifically tailored for Earn Vault use cases with EIP-712 signature authentication.

**Core Features**:
- **Yield Operations**: Deposit and withdraw from yield protocols
- **Token Swaps**: Execute token swaps via user wallet
- **Reward Claims**: Claim protocol incentives
- **Batch Actions**: Execute multiple yield operations in sequence
- **Group-Based Access**: Operations scoped to specific group IDs
- **Signature Authentication**: EIP-712 compliant signatures for delegated operations

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
    action: uint8              # Action type: 10=depositYield, 11=withdrawYield
    legoId: uint16             # Protocol/Lego ID
    asset: address             # Primary asset/token (or vaultToken for withdrawals)
    target: address            # vaultAddr for deposits, unused for withdrawals
    amount: uint256            # Primary amount (or max_value for "all")
    extraData: bytes32         # Protocol-specific extra data
```

### Action Codes

| Code | Action | Description |
|------|--------|-------------|
| 10 | Deposit Yield | Deposit into yield protocol |
| 11 | Withdraw Yield | Withdraw from yield protocol |
| 20 | Swap | Execute token swap |
| 50 | Claim Rewards | Claim protocol incentives |

## Constants

- `MAX_INSTRUCTIONS: uint256 = 15` - Maximum batch instructions
- `MAX_SWAP_INSTRUCTIONS: uint256 = 5` - Maximum swap instructions per swap
- `MAX_PROOFS: uint256 = 25` - Maximum merkle proofs for claims

## State Variables

| Name | Type | Description |
|------|------|-------------|
| `groupId` | `uint256` | Group identifier for this agent |
| `currentNonce` | `HashMap[address, uint256]` | Per-wallet nonce for replay protection |

## Constructor

### `__init__`

```vyper
@deploy
def __init__(
    _undyHq: address,
    _owner: address,
    _groupId: uint256,
    _minTimeLock: uint256,
    _maxTimeLock: uint256,
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyHq` | `address` | UndyHq registry address |
| `_owner` | `address` | Contract owner |
| `_groupId` | `uint256` | Group identifier for access control |
| `_minTimeLock` | `uint256` | Minimum timelock duration |
| `_maxTimeLock` | `uint256` | Maximum timelock duration |

## Yield Functions

### `depositForYield`

Deposit assets into a yield protocol.

```vyper
@external
def depositForYield(
    _userWallet: address,
    _legoId: uint256,
    _asset: address,
    _vaultAddr: address = empty(address),
    _amount: uint256 = max_value(uint256),
    _extraData: bytes32 = empty(bytes32),
    _sig: Signature = empty(Signature),
) -> (uint256, address, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_userWallet` | `address` | User wallet to operate on |
| `_legoId` | `uint256` | Yield protocol Lego ID |
| `_asset` | `address` | Asset to deposit |
| `_vaultAddr` | `address` | Target vault address (optional) |
| `_amount` | `uint256` | Amount to deposit (max_value = all) |
| `_extraData` | `bytes32` | Protocol-specific data |
| `_sig` | `Signature` | Authentication signature |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of underlying deposited |
| `address` | Vault token received |
| `uint256` | Amount of vault tokens received |
| `uint256` | USD value of transaction |

### `withdrawFromYield`

Withdraw assets from a yield protocol.

```vyper
@external
def withdrawFromYield(
    _userWallet: address,
    _legoId: uint256,
    _vaultToken: address,
    _amount: uint256 = max_value(uint256),
    _extraData: bytes32 = empty(bytes32),
    _sig: Signature = empty(Signature),
) -> (uint256, address, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_userWallet` | `address` | User wallet to operate on |
| `_legoId` | `uint256` | Yield protocol Lego ID |
| `_vaultToken` | `address` | Vault token to withdraw |
| `_amount` | `uint256` | Amount of vault tokens (max_value = all) |
| `_extraData` | `bytes32` | Protocol-specific data |
| `_sig` | `Signature` | Authentication signature |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Vault tokens used |
| `address` | Underlying asset received |
| `uint256` | Amount of underlying received |
| `uint256` | USD value of transaction |

## Swap Functions

### `swapTokens`

Execute token swaps.

```vyper
@external
def swapTokens(
    _userWallet: address,
    _swapInstructions: DynArray[Wallet.SwapInstruction, MAX_SWAP_INSTRUCTIONS],
    _sig: Signature = empty(Signature),
) -> (address, uint256, address, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_userWallet` | `address` | User wallet to operate on |
| `_swapInstructions` | `DynArray[SwapInstruction, 5]` | Swap instructions |
| `_sig` | `Signature` | Authentication signature |

#### Returns

| Type | Description |
|------|-------------|
| `address` | Token in address |
| `uint256` | Amount in |
| `address` | Token out address |
| `uint256` | Amount out |
| `uint256` | USD value |

## Rewards Functions

### `claimIncentives`

Claim protocol rewards.

```vyper
@external
def claimIncentives(
    _userWallet: address,
    _legoId: uint256,
    _rewardToken: address = empty(address),
    _rewardAmount: uint256 = max_value(uint256),
    _proofs: DynArray[bytes32, MAX_PROOFS] = [],
    _sig: Signature = empty(Signature),
) -> (uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_userWallet` | `address` | User wallet to operate on |
| `_legoId` | `uint256` | Rewards protocol Lego ID |
| `_rewardToken` | `address` | Reward token to claim |
| `_rewardAmount` | `uint256` | Amount to claim |
| `_proofs` | `DynArray[bytes32, 25]` | Merkle proofs for claim |
| `_sig` | `Signature` | Authentication signature |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount claimed |
| `uint256` | USD value |

## Batch Actions

### `performBatchActions`

Execute multiple operations in sequence.

```vyper
@external
def performBatchActions(
    _userWallet: address,
    _instructions: DynArray[ActionInstruction, MAX_INSTRUCTIONS],
    _sig: Signature = empty(Signature),
) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_userWallet` | `address` | User wallet to operate on |
| `_instructions` | `DynArray[ActionInstruction, 15]` | Batch instructions |
| `_sig` | `Signature` | Authentication signature |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True on success |

#### Notes

- Instructions execute sequentially
- `usePrevAmountOut` chains output to next instruction's input
- Only supports action codes 10 (deposit) and 11 (withdraw)

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
- Empty signature/nonce/expiration required when owner calls directly
- Group ID scopes operations

### Batch Action Safety
- Only deposit (10) and withdraw (11) actions supported
- Invalid actions raise "Invalid action" error
- Output chaining validates non-zero previous amounts

## Common Integration Patterns

### Direct Yield Deposit
```python
# Owner deposits directly
earn_agent.depositForYield(
    user_wallet.address,
    aave_lego_id,
    usdc.address,
    aave_vault.address,
    1000e6,  # 1000 USDC
    sender=owner
)
```

### Signature-Based Yield Withdrawal
```python
# Get signature hash
hash, nonce, expiration = sig_helper.getWithdrawFromYieldHash(
    earn_agent.address,
    user_wallet.address,
    aave_lego_id,
    ausdc.address,
    max_value(uint256)
)

# Sign and execute
sig = sign_message(hash, owner_key)
earn_agent.withdrawFromYield(
    user_wallet.address,
    aave_lego_id,
    ausdc.address,
    max_value(uint256),
    empty(bytes32),
    Signature(signature=sig, nonce=nonce, expiration=expiration),
    sender=relayer
)
```

### Batch Yield Rebalancing
```python
# Withdraw from one vault, deposit to another
instructions = [
    ActionInstruction(
        usePrevAmountOut=False,
        action=11,  # withdraw
        legoId=aave_lego_id,
        asset=ausdc.address,
        target=empty(address),
        amount=max_value(uint256),
        extraData=empty(bytes32)
    ),
    ActionInstruction(
        usePrevAmountOut=True,  # Use withdrawn amount
        action=10,  # deposit
        legoId=compound_lego_id,
        asset=usdc.address,
        target=compound_vault.address,
        amount=0,  # Will use prev output
        extraData=empty(bytes32)
    )
]

earn_agent.performBatchActions(
    user_wallet.address,
    instructions,
    sender=owner
)
```

## Testing

For test examples, see: [`tests/core/agent/`](../../../tests/core/agent/)
