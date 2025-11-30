# UserWalletSignatureHelper Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/core/agent/UserWalletSignatureHelper.vy)

## Overview

UserWalletSignatureHelper is a comprehensive utility contract that generates EIP-712 compliant message hashes for AgentSenderGeneric operations. It provides helper functions for all AgentSenderGeneric methods including transfers, yield operations, swaps, debt management, liquidity operations, and batch actions.

**Core Features**:
- **Complete Coverage**: Hash generation for all AgentSenderGeneric functions
- **Transfer Operations**: Fund transfer hash generation
- **Yield Operations**: Deposit, withdraw, and rebalance hash generation
- **Swap Operations**: Token swap, mint/redeem hash generation
- **Debt Operations**: Collateral, borrow, and repay hash generation
- **Liquidity Operations**: Standard and concentrated liquidity hash generation
- **Batch Actions**: Multi-instruction hash generation

## Data Structures

### ActionInstruction

```vyper
struct ActionInstruction:
    usePrevAmountOut: bool     # Use output from previous instruction as amount
    action: uint8              # Action type (see Action Codes table)
    legoId: uint16             # Protocol/Lego ID
    asset: address             # Primary asset/token
    target: address            # Varies: recipient/vaultAddr/tokenOut/pool
    amount: uint256            # Primary amount (max_value = "all")
    asset2: address            # Secondary asset (tokenB for liquidity)
    amount2: uint256           # Secondary amount or toLegoId
    minOut1: uint256           # Min output for primary asset
    minOut2: uint256           # Min output for secondary asset
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
| 22 | Confirm Mint/Redeem | Confirm pending operation |
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

## Module Dependencies

Uses the `SigHelper` module for:
- `_getNonceAndExpiration`: Retrieve or calculate nonce and expiration
- `_getFullDigest`: Generate EIP-712 full digest with domain separator

## Transfer Functions

### `getTransferFundsHash`

Get message hash for `transferFunds` function.

```vyper
@view
@external
def getTransferFundsHash(
    _agentSender: address,
    _userWallet: address,
    _recipient: address,
    _asset: address = empty(address),
    _amount: uint256 = max_value(uint256),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentSender` | `address` | AgentSenderGeneric contract address |
| `_userWallet` | `address` | User wallet address |
| `_recipient` | `address` | Transfer recipient |
| `_asset` | `address` | Asset to transfer |
| `_amount` | `uint256` | Amount to transfer |
| `_nonce` | `uint256` | Nonce (0 = auto-fetch) |
| `_expiration` | `uint256` | Expiration (0 = auto-calculate) |

#### Returns

| Type | Description |
|------|-------------|
| `bytes32` | EIP-712 message hash to sign |
| `uint256` | Nonce used |
| `uint256` | Expiration timestamp used |

## Yield Functions

### `getDepositForYieldHash`

Get message hash for `depositForYield` function.

```vyper
@view
@external
def getDepositForYieldHash(
    _agentSender: address,
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

### `getWithdrawFromYieldHash`

Get message hash for `withdrawFromYield` function.

```vyper
@view
@external
def getWithdrawFromYieldHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _vaultToken: address,
    _amount: uint256 = max_value(uint256),
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

### `getRebalanceYieldPositionHash`

Get message hash for `rebalanceYieldPosition` function.

```vyper
@view
@external
def getRebalanceYieldPositionHash(
    _agentSender: address,
    _userWallet: address,
    _fromLegoId: uint256,
    _fromVaultToken: address,
    _toLegoId: uint256,
    _toVaultAddr: address = empty(address),
    _fromVaultAmount: uint256 = max_value(uint256),
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

## Swap Functions

### `getSwapTokensHash`

Get message hash for `swapTokens` function.

```vyper
@view
@external
def getSwapTokensHash(
    _agentSender: address,
    _userWallet: address,
    _swapInstructions: DynArray[Wallet.SwapInstruction, MAX_SWAP_INSTRUCTIONS],
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

### `getMintOrRedeemAssetHash`

Get message hash for `mintOrRedeemAsset` function.

```vyper
@view
@external
def getMintOrRedeemAssetHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _tokenIn: address,
    _tokenOut: address,
    _amountIn: uint256 = max_value(uint256),
    _minAmountOut: uint256 = 0,
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

### `getConfirmMintOrRedeemAssetHash`

Get message hash for `confirmMintOrRedeemAsset` function.

```vyper
@view
@external
def getConfirmMintOrRedeemAssetHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _tokenIn: address,
    _tokenOut: address,
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

## Debt Management Functions

### `getAddCollateralHash`

Get message hash for `addCollateral` function.

```vyper
@view
@external
def getAddCollateralHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _asset: address,
    _amount: uint256 = max_value(uint256),
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

### `getRemoveCollateralHash`

Get message hash for `removeCollateral` function.

```vyper
@view
@external
def getRemoveCollateralHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _asset: address,
    _amount: uint256 = max_value(uint256),
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

### `getBorrowHash`

Get message hash for `borrow` function.

```vyper
@view
@external
def getBorrowHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _borrowAsset: address,
    _amount: uint256 = max_value(uint256),
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

### `getRepayDebtHash`

Get message hash for `repayDebt` function.

```vyper
@view
@external
def getRepayDebtHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _paymentAsset: address,
    _paymentAmount: uint256 = max_value(uint256),
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

## Wrapped ETH Functions

### `getConvertWethToEthHash`

Get message hash for `convertWethToEth` function.

```vyper
@view
@external
def getConvertWethToEthHash(
    _agentSender: address,
    _userWallet: address,
    _amount: uint256 = max_value(uint256),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

### `getConvertEthToWethHash`

Get message hash for `convertEthToWeth` function.

```vyper
@view
@external
def getConvertEthToWethHash(
    _agentSender: address,
    _userWallet: address,
    _amount: uint256 = max_value(uint256),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

## Liquidity Functions

### `getAddLiquidityHash`

Get message hash for `addLiquidity` function.

```vyper
@view
@external
def getAddLiquidityHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _pool: address,
    _tokenA: address,
    _tokenB: address,
    _amountA: uint256 = max_value(uint256),
    _amountB: uint256 = max_value(uint256),
    _minAmountA: uint256 = 0,
    _minAmountB: uint256 = 0,
    _minLpAmount: uint256 = 0,
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

### `getRemoveLiquidityHash`

Get message hash for `removeLiquidity` function.

```vyper
@view
@external
def getRemoveLiquidityHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _pool: address,
    _tokenA: address,
    _tokenB: address,
    _lpToken: address,
    _lpAmount: uint256 = max_value(uint256),
    _minAmountA: uint256 = 0,
    _minAmountB: uint256 = 0,
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

### `getAddLiquidityConcentratedHash`

Get message hash for `addLiquidityConcentrated` function.

```vyper
@view
@external
def getAddLiquidityConcentratedHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _nftAddr: address,
    _nftTokenId: uint256,
    _pool: address,
    _tokenA: address,
    _tokenB: address,
    _amountA: uint256 = max_value(uint256),
    _amountB: uint256 = max_value(uint256),
    _tickLower: int24 = min_value(int24),
    _tickUpper: int24 = max_value(int24),
    _minAmountA: uint256 = 0,
    _minAmountB: uint256 = 0,
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

### `getRemoveLiquidityConcentratedHash`

Get message hash for `removeLiquidityConcentrated` function.

```vyper
@view
@external
def getRemoveLiquidityConcentratedHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _nftAddr: address,
    _nftTokenId: uint256,
    _pool: address,
    _tokenA: address,
    _tokenB: address,
    _liqToRemove: uint256 = max_value(uint256),
    _minAmountA: uint256 = 0,
    _minAmountB: uint256 = 0,
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

## Rewards Functions

### `getClaimRewardsHash`

Get message hash for `claimRewards` function (without proofs).

```vyper
@view
@external
def getClaimRewardsHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _rewardToken: address = empty(address),
    _rewardAmount: uint256 = max_value(uint256),
    _extraData: bytes32 = empty(bytes32),
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

### `getClaimIncentivesHash`

Get message hash for `claimIncentives` function (with proofs).

```vyper
@view
@external
def getClaimIncentivesHash(
    _agentSender: address,
    _userWallet: address,
    _legoId: uint256,
    _rewardToken: address = empty(address),
    _rewardAmount: uint256 = max_value(uint256),
    _proofs: DynArray[bytes32, MAX_PROOFS] = [],
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

## Batch Actions

### `getBatchActionsHash`

Get message hash for `performBatchActions` function.

```vyper
@view
@external
def getBatchActionsHash(
    _agentSender: address,
    _userWallet: address,
    _instructions: DynArray[ActionInstruction, MAX_INSTRUCTIONS],
    _nonce: uint256 = 0,
    _expiration: uint256 = 0,
) -> (bytes32, uint256, uint256):
```

## Common Integration Patterns

### Transfer Signature Generation
```python
# Generate hash for transfer
hash, nonce, expiration = sig_helper.getTransferFundsHash(
    agent_sender.address,
    user_wallet.address,
    recipient.address,
    usdc.address,
    1000e6
)

# Sign and execute
signature = sign_eip712_hash(hash, owner_key)
agent_sender.transferFunds(
    agent_wrapper.address,
    user_wallet.address,
    recipient.address,
    usdc.address,
    1000e6,
    False,  # isCheque
    Signature(signature, nonce, expiration),
    sender=relayer
)
```

### Batch Actions Signature
```python
instructions = [
    ActionInstruction(
        usePrevAmountOut=False,
        action=11,  # withdrawYield
        legoId=aave_lego_id,
        asset=ausdc.address,
        target=empty(address),
        amount=max_value(uint256),
        # ... other fields
    ),
    ActionInstruction(
        usePrevAmountOut=True,
        action=10,  # depositYield
        legoId=compound_lego_id,
        asset=usdc.address,
        target=compound_vault.address,
        amount=0,
        # ... other fields
    )
]

hash, nonce, expiration = sig_helper.getBatchActionsHash(
    agent_sender.address,
    user_wallet.address,
    instructions
)
```

### Concentrated Liquidity Signature
```python
hash, nonce, expiration = sig_helper.getAddLiquidityConcentratedHash(
    agent_sender.address,
    user_wallet.address,
    uniswap_v3_lego_id,
    position_manager.address,
    0,  # New position
    pool.address,
    weth.address,
    usdc.address,
    1e18,  # 1 WETH
    2000e6,  # 2000 USDC
    -887220,  # tickLower
    887220,  # tickUpper
    0,  # minAmountA
    0   # minAmountB
)
```

## Security Considerations

### Nonce Management
- Nonce of 0 triggers automatic fetch from AgentSenderGeneric
- Nonces must match exactly for signature validation
- Each wallet has independent nonce tracking

### Expiration Handling
- Expiration of 0 triggers automatic calculation
- Should be set to reasonable future time (not too long)
- Prevents indefinite signature validity

### Hash Uniqueness
- Each function has unique action code prefix
- All parameters included in hash
- Domain separator includes contract address and chain ID

## Testing

For test examples, see: [`tests/core/agent/`](../../../tests/core/agent/)
