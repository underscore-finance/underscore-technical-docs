# LevgVaultAgent Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/core/agent/LevgVaultAgent.vy)

## Overview

LevgVaultAgent is a specialized agent wrapper for Leverage Vault operations. It provides multi-step workflows for managing leveraged yield positions through borrowing, yield generation, deleveraging, and compounding - all with EIP-712 signature authentication.

**Core Features**:
- **Leverage Workflows**: Complex multi-step operations for leveraged yield strategies
- **Three Operation Modes**: Borrow & earn, deleverage, and compound yield
- **Ripe Protocol Integration**: Direct integration with Ripe for debt management
- **Signature Authentication**: EIP-712 compliant signatures for delegated operations
- **Output Chaining**: Automatic chaining of operation outputs between workflow steps
- **Position Type Support**: Collateral, leverage, and stability pool (sGREEN) positions

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                            LevgVaultAgent                                |
|                    (Leverage Workflow Orchestrator)                      |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Workflow Operations                            | |
|  |                                                                   | |
|  |  borrowAndEarnYield()  -->  Multi-step leverage workflow          | |
|  |    1. Remove collateral from Ripe                                 | |
|  |    2. Withdraw from yield vaults                                  | |
|  |    3. Deposit into yield vaults (+ optional add to Ripe)          | |
|  |    4. Add collateral to Ripe                                      | |
|  |    5. Borrow GREEN/sGREEN from Ripe                               | |
|  |    6. Swap tokens                                                 | |
|  |    7. Post-swap deposits                                          | |
|  |                                                                   | |
|  |  deleverage()  -->  Multi-mode deleveraging                       | |
|  |    Mode 0: Auto-deleverage via Ripe                               | |
|  |    Mode 1: Deleverage with specific assets                        | |
|  |    Mode 2: Manual (remove -> withdraw -> swap -> repay)           | |
|  |                                                                   | |
|  |  compoundYieldGains()  -->  Compound yield profits                | |
|  |    1. Remove collateral from Ripe                                 | |
|  |    2. Withdraw from yield vaults                                  | |
|  |    3. Swap to collateral token                                    | |
|  |    4. Post-swap deposits                                          | |
|  |    5. Add as collateral to Ripe                                   | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Position Types                                 | |
|  |                                                                   | |
|  |  Type 0: Collateral   -->  Collateral vault token                 | |
|  |  Type 1: Leverage     -->  Leverage vault token                   | |
|  |  Type 2: StabPool     -->  Savings GREEN (sGREEN)                 | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    External Dependencies                          | |
|  |                                                                   | |
|  |  LevgVault/Wallet  -->  Target vault for operations               | |
|  |  RipeLego          -->  Debt management and deleveraging          | |
|  |  Ownership Module  -->  Owner-based authentication                | |
|  +-------------------------------------------------------------------+ |
+-------------------------------------------------------------------------+
```

## Data Structures

### Signature

Authentication structure for delegated operations:
```vyper
struct Signature:
    signature: Bytes[65]  # ECDSA signature (r, s, v)
    nonce: uint256        # Replay protection nonce
    expiration: uint256   # Signature expiry timestamp
```

### PositionAsset

Specifies a position for collateral/withdrawal operations:
```vyper
struct PositionAsset:
    positionType: uint8  # 0=collateral, 1=leverage, 2=stabPool
    amount: uint256      # Amount (0 or max_value for "all")
```

### DepositYieldPosition

Specifies a deposit operation with optional Ripe integration:
```vyper
struct DepositYieldPosition:
    positionType: uint8              # 0=collateral, 1=leverage, 2=stabPool
    amount: uint256                  # Amount to deposit
    shouldAddToRipeCollateral: bool  # Add vault tokens to Ripe as collateral
    shouldSweepAll: bool             # Deposit full wallet balance
```

### DeleverageAsset

For mode 1 deleveraging with specific assets:
```vyper
struct DeleverageAsset:
    vaultId: uint256        # Ripe vault ID
    asset: address          # Asset address
    targetRepayAmount: uint256  # Target repayment amount
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
| `RIPE_LEGO_ID` | 1 | Lego ID for Ripe Protocol |
| `RIPE_STAB_POOL_ID` | 1 | Ripe stability pool vault ID |
| `LEGO_BOOK_ID` | 3 | UndyHq registry ID for LegoBook |
| `MAX_DELEVERAGE_ASSETS` | 25 | Maximum assets for specific deleveraging |
| `MAX_POSITIONS` | 10 | Maximum positions per workflow step |
| `POSITION_COLLATERAL` | 0 | Collateral position type |
| `POSITION_LEVERAGE` | 1 | Leverage position type |
| `POSITION_STAB_POOL` | 2 | Stability pool (sGREEN) position type |
| `WORKFLOW_BORROW_AND_EARN` | 100 | Action code for borrow+earn workflow |
| `WORKFLOW_DELEVERAGE` | 101 | Action code for deleverage workflow |
| `WORKFLOW_COMPOUND_YIELD` | 102 | Action code for compound workflow |

## State Variables

| Name | Type | Description |
|------|------|-------------|
| `UNDY_HQ` | `address` (immutable) | UndyHq registry address |
| `GREEN` | `address` (immutable) | GREEN token address |
| `SAVINGS_GREEN` | `address` (immutable) | Savings GREEN (sGREEN) token address |
| `currentNonce` | `HashMap[address, uint256]` | Per-vault nonce for replay protection |

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
| `_greenToken` | `address` | GREEN token address |
| `_savingsGreen` | `address` | Savings GREEN token address |

## Workflow Functions

### `borrowAndEarnYield`

Multi-step workflow for leveraging positions. Executes up to 7 steps in sequence with automatic output chaining.

```vyper
@external
def borrowAndEarnYield(
    _levgWallet: address,
    _removeCollateral: DynArray[PositionAsset, MAX_POSITIONS] = [],
    _withdrawPositions: DynArray[PositionAsset, MAX_POSITIONS] = [],
    _depositPositions: DynArray[DepositYieldPosition, MAX_POSITIONS] = [],
    _addCollateral: DynArray[PositionAsset, MAX_POSITIONS] = [],
    _borrowAmount: uint256 = 0,
    _wantsSavingsGreen: bool = True,
    _shouldEnterStabPool: bool = True,
    _swapInstruction: Wallet.SwapInstruction = empty(Wallet.SwapInstruction),
    _postSwapDeposits: DynArray[DepositYieldPosition, MAX_POSITIONS] = [],
    _sig: Signature = empty(Signature),
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgWallet` | `address` | Leverage vault wallet to operate on |
| `_removeCollateral` | `DynArray[PositionAsset, 10]` | Step 1: Collateral to remove from Ripe |
| `_withdrawPositions` | `DynArray[PositionAsset, 10]` | Step 2: Yield positions to withdraw |
| `_depositPositions` | `DynArray[DepositYieldPosition, 10]` | Step 3: Yield deposits |
| `_addCollateral` | `DynArray[PositionAsset, 10]` | Step 4: Collateral to add to Ripe |
| `_borrowAmount` | `uint256` | Step 5: Amount to borrow (0 = skip) |
| `_wantsSavingsGreen` | `bool` | Borrow sGREEN instead of GREEN |
| `_shouldEnterStabPool` | `bool` | Enter stability pool when borrowing |
| `_swapInstruction` | `SwapInstruction` | Step 6: Token swap (empty = skip) |
| `_postSwapDeposits` | `DynArray[DepositYieldPosition, 10]` | Step 7: Post-swap deposits |
| `_sig` | `Signature` | Authentication signature |

#### Workflow Steps

1. **Remove Collateral**: Withdraw vault tokens from Ripe collateral
2. **Withdraw from Yield**: Redeem positions from yield vaults
3. **Deposit into Yield**: Deploy assets into yield protocols (optionally add to Ripe)
4. **Add Collateral**: Supply vault tokens as Ripe collateral
5. **Borrow**: Borrow GREEN or sGREEN against collateral
6. **Swap**: Exchange tokens (auto-chains from borrow output if amountIn=0)
7. **Post-Swap Deposits**: Deploy swapped tokens into yield positions

#### Output Chaining

- If `_swapInstruction.amountIn = 0` and borrow occurred, uses borrow output as swap input
- If post-swap deposit underlying matches swap output, uses swap output amount

### `deleverage`

Multi-mode deleveraging workflow to reduce leverage exposure.

```vyper
@external
def deleverage(
    _levgWallet: address,
    _mode: uint8 = 0,
    _autoDeleverageAmount: uint256 = 0,
    _deleverageAssets: DynArray[DeleverageAsset, MAX_DELEVERAGE_ASSETS] = [],
    _removeCollateral: DynArray[PositionAsset, MAX_POSITIONS] = [],
    _withdrawPositions: DynArray[PositionAsset, MAX_POSITIONS] = [],
    _swapInstruction: Wallet.SwapInstruction = empty(Wallet.SwapInstruction),
    _repayAsset: address = empty(address),
    _repayAmount: uint256 = 0,
    _shouldSweepAllForRepay: bool = False,
    _sig: Signature = empty(Signature),
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgWallet` | `address` | Leverage vault wallet to operate on |
| `_mode` | `uint8` | Deleverage mode (0=auto, 1=specific, 2=manual) |
| `_autoDeleverageAmount` | `uint256` | Mode 0: Target deleverage amount |
| `_deleverageAssets` | `DynArray[DeleverageAsset, 25]` | Mode 1: Specific assets to deleverage |
| `_removeCollateral` | `DynArray[PositionAsset, 10]` | Mode 2: Collateral to remove |
| `_withdrawPositions` | `DynArray[PositionAsset, 10]` | Mode 2: Positions to withdraw |
| `_swapInstruction` | `SwapInstruction` | Mode 2: Swap instruction |
| `_repayAsset` | `address` | Asset for debt repayment |
| `_repayAmount` | `uint256` | Amount to repay |
| `_shouldSweepAllForRepay` | `bool` | Use full balance for repayment |
| `_sig` | `Signature` | Authentication signature |

#### Deleverage Modes

**Mode 0 - Auto Deleverage**:
- Calls `RipeLego.deleverageUser(_levgWallet, _autoDeleverageAmount)`
- Ripe Protocol automatically handles collateral liquidation

**Mode 1 - Specific Assets**:
- Calls `RipeLego.deleverageWithSpecificAssets(_deleverageAssets, _levgWallet)`
- Allows targeting specific vault positions

**Mode 2 - Manual**:
1. Remove collateral from Ripe
2. Withdraw from yield vaults
3. Swap tokens (e.g., USDC → GREEN)
4. Repay debt

**Debt Repayment (All Modes)**:
- After mode execution, optionally repays debt
- Auto-chains swap output if `_repayAsset` matches swap output

### `compoundYieldGains`

Workflow to compound yield profits back into leveraged positions.

```vyper
@external
def compoundYieldGains(
    _levgWallet: address,
    _removeCollateral: DynArray[PositionAsset, MAX_POSITIONS] = [],
    _withdrawPositions: DynArray[PositionAsset, MAX_POSITIONS] = [],
    _swapInstruction: Wallet.SwapInstruction = empty(Wallet.SwapInstruction),
    _postSwapDeposits: DynArray[DepositYieldPosition, MAX_POSITIONS] = [],
    _addCollateral: DynArray[PositionAsset, MAX_POSITIONS] = [],
    _sig: Signature = empty(Signature),
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_levgWallet` | `address` | Leverage vault wallet to operate on |
| `_removeCollateral` | `DynArray[PositionAsset, 10]` | Step 1: Collateral to remove |
| `_withdrawPositions` | `DynArray[PositionAsset, 10]` | Step 2: Positions to withdraw |
| `_swapInstruction` | `SwapInstruction` | Step 3: Swap instruction |
| `_postSwapDeposits` | `DynArray[DepositYieldPosition, 10]` | Step 4: Post-swap deposits |
| `_addCollateral` | `DynArray[PositionAsset, 10]` | Step 5: Collateral to add |
| `_sig` | `Signature` | Authentication signature |

#### Workflow Steps

1. **Remove Collateral**: Withdraw from Ripe collateral
2. **Withdraw Yield**: Exit yield positions to realize profits
3. **Swap**: Convert profits to collateral asset
4. **Post-Swap Deposits**: Re-deploy into yield protocols
5. **Add Collateral**: Increase Ripe collateral position

## Nonce Management

### `incrementNonce`

Manually increment a wallet's nonce (for invalidating pending signatures).

```vyper
@external
def incrementNonce(_levgWallet: address):
```

#### Access

Owner only

### `getNonce`

Get current nonce for a wallet.

```vyper
@view
@external
def getNonce(_levgWallet: address) -> uint256:
```

## Events

### `NonceIncremented`

Emitted when a wallet's nonce is incremented.

```vyper
event NonceIncremented:
    levgVault: address   # Wallet address
    oldNonce: uint256    # Previous nonce
    newNonce: uint256    # New nonce
```

## Security Considerations

### Signature Validation
- EIP-712 compliant domain separator with chain ID binding
- Signature malleability prevention (s-value range check)
- Expiration timestamp validation
- Nonce-based replay protection per wallet

### Access Control
- Owner can call directly without signature (must provide empty sig/nonce/expiration)
- Non-owners must provide valid signature from owner
- Agent must be registered as manager in target LevgVault
- Nonce auto-increments on successful signature use

### Output Chaining Safety
- `amountIn = 0` enables auto-chaining from previous operation output
- `amountIn > 0` uses explicit amount (opt-out of chaining)
- Balance checks prevent over-spending
- `shouldSweepAll` deposits full wallet balance regardless of chaining

### Position Type Validation
- Valid types: 0 (collateral), 1 (leverage), 2 (stabPool)
- Invalid position types silently skip operations
- Empty vault tokens skip operations

## Common Integration Patterns

### Opening a Leveraged Position

```python
# Deposit USDC, borrow GREEN, swap to USDC, deposit to leverage vault
levg_agent.borrowAndEarnYield(
    levg_vault.address,
    _depositPositions=[
        DepositYieldPosition(
            positionType=0,  # collateral
            amount=1000e6,   # 1000 USDC
            shouldAddToRipeCollateral=True,
            shouldSweepAll=False
        )
    ],
    _borrowAmount=2000e6,  # Borrow 2000 GREEN
    _wantsSavingsGreen=False,
    _shouldEnterStabPool=True,
    _swapInstruction=SwapInstruction(
        legoId=dex_lego_id,
        amountIn=0,  # Auto-chain from borrow
        minAmountOut=1900e6,  # 5% slippage
        tokenPath=[green.address, usdc.address],
        poolPath=[pool_id]
    ),
    _postSwapDeposits=[
        DepositYieldPosition(
            positionType=1,  # leverage
            amount=0,  # Auto-chain from swap
            shouldAddToRipeCollateral=False,
            shouldSweepAll=False
        )
    ],
    sender=owner
)
```

### Auto-Deleveraging

```python
# Auto-deleverage to target amount
levg_agent.deleverage(
    levg_vault.address,
    _mode=0,  # Auto mode
    _autoDeleverageAmount=500e6,  # Reduce 500 GREEN debt
    sender=owner
)
```

### Manual Deleveraging

```python
# Manual deleverage: withdraw -> swap -> repay
levg_agent.deleverage(
    levg_vault.address,
    _mode=2,  # Manual mode
    _withdrawPositions=[
        PositionAsset(positionType=1, amount=500e6)  # Withdraw leverage
    ],
    _swapInstruction=SwapInstruction(
        legoId=dex_lego_id,
        amountIn=0,  # Auto-chain from withdraw
        minAmountOut=490e6,
        tokenPath=[usdc.address, green.address],
        poolPath=[pool_id]
    ),
    _repayAsset=green.address,
    _shouldSweepAllForRepay=True,
    sender=owner
)
```

### Compounding Yield

```python
# Compound yield profits back into collateral
levg_agent.compoundYieldGains(
    levg_vault.address,
    _withdrawPositions=[
        PositionAsset(positionType=1, amount=50e6)  # Withdraw profits
    ],
    _swapInstruction=SwapInstruction(
        legoId=dex_lego_id,
        amountIn=0,
        minAmountOut=49e6,
        tokenPath=[usdc.address, collateral_token.address],
        poolPath=[pool_id]
    ),
    _postSwapDeposits=[
        DepositYieldPosition(
            positionType=0,  # collateral
            amount=0,  # Auto-chain
            shouldAddToRipeCollateral=True,
            shouldSweepAll=False
        )
    ],
    sender=owner
)
```

### Signature-Based Delegation

```python
# Create signature for delegated operation
message_hash = keccak256(abi_encode(
    WORKFLOW_BORROW_AND_EARN,  # 100
    levg_vault.address,
    remove_collateral,
    withdraw_positions,
    deposit_positions,
    add_collateral,
    borrow_amount,
    wants_savings_green,
    should_enter_stab_pool,
    swap_instruction,
    post_swap_deposits,
    nonce,
    expiration
))

signature = sign_eip712(message_hash, owner_key)

# Execute via relayer
levg_agent.borrowAndEarnYield(
    levg_vault.address,
    # ... parameters ...
    _sig=Signature(
        signature=signature,
        nonce=nonce,
        expiration=expiration
    ),
    sender=relayer
)
```

## Testing

For test examples, see: [`tests/core/agent/`](../../../tests/core/agent/)
