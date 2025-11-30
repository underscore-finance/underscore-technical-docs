# EarnVaultAgentSigHelper Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/core/agent/EarnVaultAgentSigHelper.vy)

## Overview

EarnVaultAgentSigHelper is a utility contract that generates EIP-712 compliant message hashes for EarnVaultAgent operations. It provides helper functions for each EarnVaultAgent method, enabling off-chain signature generation for delegated operations.

**Core Features**:
- **Message Hash Generation**: Creates properly formatted hashes for each EarnVaultAgent function
- **Nonce Management**: Automatically retrieves current nonce or uses provided value
- **Expiration Handling**: Sets default expiration or uses provided value
- **EIP-712 Compliance**: Full digest generation with domain separator

## Data Structures

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

## Constants

- `MAX_INSTRUCTIONS: uint256 = 15` - Maximum batch instructions
- `MAX_SWAP_INSTRUCTIONS: uint256 = 5` - Maximum swap instructions
- `MAX_PROOFS: uint256 = 25` - Maximum merkle proofs

## Module Dependencies

Uses the `SigHelper` module for:
- `_getNonceAndExpiration`: Retrieve or calculate nonce and expiration
- `_getFullDigest`: Generate EIP-712 full digest with domain separator

## Yield Functions

### `getDepositForYieldHash`

Get message hash for `depositForYield` function.

```vyper
@view
@external
def getDepositForYieldHash(
    _agentWrapper: address,
    _userWallet: address,
    _legoId: uint256,
    _asset: address,
    _vaultAddr: address = empty(address),
    _amount: uint256 = max_value(uint256),
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentWrapper` | `address` | EarnVaultAgent contract address |
| `_userWallet` | `address` | User wallet address |
| `_legoId` | `uint256` | Yield protocol Lego ID |
| `_asset` | `address` | Asset to deposit |
| `_vaultAddr` | `address` | Target vault address |
| `_amount` | `uint256` | Amount to deposit |
| `_extraData` | `bytes32` | Protocol-specific data |
| `_nonce` | `uint256` | Nonce (0 = auto-fetch current) |
| `_expiration` | `uint256` | Expiration timestamp (0 = auto-calculate) |

#### Returns

| Type | Description |
|------|-------------|
| `bytes32` | EIP-712 message hash to sign |
| `uint256` | Nonce used |
| `uint256` | Expiration timestamp used |

### `getWithdrawFromYieldHash`

Get message hash for `withdrawFromYield` function.

```vyper
@view
@external
def getWithdrawFromYieldHash(
    _agentWrapper: address,
    _userWallet: address,
    _legoId: uint256,
    _vaultToken: address,
    _amount: uint256 = max_value(uint256),
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentWrapper` | `address` | EarnVaultAgent contract address |
| `_userWallet` | `address` | User wallet address |
| `_legoId` | `uint256` | Yield protocol Lego ID |
| `_vaultToken` | `address` | Vault token to withdraw |
| `_amount` | `uint256` | Amount of vault tokens |
| `_extraData` | `bytes32` | Protocol-specific data |
| `_nonce` | `uint256` | Nonce (0 = auto-fetch) |
| `_expiration` | `uint256` | Expiration timestamp (0 = auto-calculate) |

#### Returns

| Type | Description |
|------|-------------|
| `bytes32` | EIP-712 message hash to sign |
| `uint256` | Nonce used |
| `uint256` | Expiration timestamp used |

## Swap Functions

### `getSwapTokensHash`

Get message hash for `swapTokens` function.

```vyper
@view
@external
def getSwapTokensHash(
    _agentWrapper: address,
    _userWallet: address,
    _swapInstructions: DynArray[Wallet.SwapInstruction, MAX_SWAP_INSTRUCTIONS],
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentWrapper` | `address` | EarnVaultAgent contract address |
| `_userWallet` | `address` | User wallet address |
| `_swapInstructions` | `DynArray[SwapInstruction, 5]` | Swap instructions |
| `_nonce` | `uint256` | Nonce (0 = auto-fetch) |
| `_expiration` | `uint256` | Expiration timestamp (0 = auto-calculate) |

#### Returns

| Type | Description |
|------|-------------|
| `bytes32` | EIP-712 message hash to sign |
| `uint256` | Nonce used |
| `uint256` | Expiration timestamp used |

## Rewards Functions

### `getClaimIncentivesHash`

Get message hash for `claimIncentives` function.

```vyper
@view
@external
def getClaimIncentivesHash(
    _agentWrapper: address,
    _userWallet: address,
    _legoId: uint256,
    _rewardToken: address = empty(address),
    _rewardAmount: uint256 = max_value(uint256),
    _proofs: DynArray[bytes32, MAX_PROOFS] = [],
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentWrapper` | `address` | EarnVaultAgent contract address |
| `_userWallet` | `address` | User wallet address |
| `_legoId` | `uint256` | Rewards protocol Lego ID |
| `_rewardToken` | `address` | Reward token to claim |
| `_rewardAmount` | `uint256` | Amount to claim |
| `_proofs` | `DynArray[bytes32, 25]` | Merkle proofs |
| `_nonce` | `uint256` | Nonce (0 = auto-fetch) |
| `_expiration` | `uint256` | Expiration timestamp (0 = auto-calculate) |

#### Returns

| Type | Description |
|------|-------------|
| `bytes32` | EIP-712 message hash to sign |
| `uint256` | Nonce used |
| `uint256` | Expiration timestamp used |

## Batch Actions

### `getBatchActionsHash`

Get message hash for `performBatchActions` function.

```vyper
@view
@external
def getBatchActionsHash(
    _agentWrapper: address,
    _userWallet: address,
    _instructions: DynArray[ActionInstruction, MAX_INSTRUCTIONS],
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentWrapper` | `address` | EarnVaultAgent contract address |
| `_userWallet` | `address` | User wallet address |
| `_instructions` | `DynArray[ActionInstruction, 15]` | Batch instructions |
| `_nonce` | `uint256` | Nonce (0 = auto-fetch) |
| `_expiration` | `uint256` | Expiration timestamp (0 = auto-calculate) |

#### Returns

| Type | Description |
|------|-------------|
| `bytes32` | EIP-712 message hash to sign |
| `uint256` | Nonce used |
| `uint256` | Expiration timestamp used |

## Message Hash Format

Each function generates a message hash using the action code and parameters:

| Function | Action Code | Hash Components |
|----------|-------------|-----------------|
| depositForYield | 10 | userWallet, legoId, asset, vaultAddr, amount, extraData, nonce, expiration |
| withdrawFromYield | 11 | userWallet, legoId, vaultToken, amount, extraData, nonce, expiration |
| swapTokens | 20 | userWallet, swapInstructions, nonce, expiration |
| claimIncentives | 50 | userWallet, legoId, rewardToken, rewardAmount, proofs, nonce, expiration |
| performBatchActions | N/A | userWallet, instructions, nonce, expiration |

## Common Integration Patterns

### Generate Hash for Yield Deposit
```python
# Get hash with auto-fetched nonce and calculated expiration
hash, nonce, expiration = sig_helper.getDepositForYieldHash(
    earn_agent.address,
    user_wallet.address,
    aave_lego_id,
    usdc.address,
    aave_vault.address,
    1000e6
)

# Sign the hash
signature = sign_eip712_hash(hash, owner_private_key)

# Execute with signature
earn_agent.depositForYield(
    user_wallet.address,
    aave_lego_id,
    usdc.address,
    aave_vault.address,
    1000e6,
    empty(bytes32),
    Signature(signature=signature, nonce=nonce, expiration=expiration),
    sender=relayer
)
```

### Generate Hash with Custom Expiration
```python
# Set expiration 1 hour from now
custom_expiration = block.timestamp + 3600

hash, nonce, expiration = sig_helper.getWithdrawFromYieldHash(
    earn_agent.address,
    user_wallet.address,
    aave_lego_id,
    ausdc.address,
    max_value(uint256),
    empty(bytes32),
    0,  # Auto-fetch nonce
    custom_expiration
)
```

### Batch Actions Hash Generation
```python
instructions = [
    ActionInstruction(
        usePrevAmountOut=False,
        action=11,
        legoId=aave_lego_id,
        asset=ausdc.address,
        target=empty(address),
        amount=max_value(uint256),
        extraData=empty(bytes32)
    ),
    ActionInstruction(
        usePrevAmountOut=True,
        action=10,
        legoId=compound_lego_id,
        asset=usdc.address,
        target=compound_vault.address,
        amount=0,
        extraData=empty(bytes32)
    )
]

hash, nonce, expiration = sig_helper.getBatchActionsHash(
    earn_agent.address,
    user_wallet.address,
    instructions
)
```

## Security Considerations

### Nonce Handling
- Nonce of 0 triggers automatic current nonce fetch
- Prevents replay attacks when nonces are properly incremented
- Each wallet has independent nonce tracking

### Expiration Handling
- Expiration of 0 triggers automatic calculation
- Prevents stale signatures from being used
- Should be set to reasonable future time

### Hash Verification
- Always verify returned nonce matches expected value
- Verify expiration is acceptable before signing
- Hash should be signed using EIP-712 personal_sign

## Testing

For test examples, see: [`tests/core/agent/`](../../../tests/core/agent/)
