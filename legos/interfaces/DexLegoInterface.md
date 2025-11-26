# DEX Lego Interface Technical Documentation

[View Interface Source](https://github.com/underscore-finance/underscore/blob/master/interfaces/DexLego.vyi)

## Overview

The DEX Lego Interface defines the standard contract interface that all decentralized exchange integrations must implement. This interface ensures consistent swap and liquidity operations across different DEX protocols (Uniswap, Aerodrome, Curve, etc.) enabling the Underscore Protocol to route trades through any compliant lego.

**Implementing Legos**:
- Uniswap V2
- Uniswap V3
- Aerodrome Classic
- Aerodrome Slipstream (concentrated liquidity)
- Curve

All DEX legos implement this interface plus the base [Lego interface](#base-lego-interface) functions.

## Core Swap Functions

### `swapTokens`

Executes a token swap through the DEX.

```vyper
@external
def swapTokens(
    _amountIn: uint256,
    _minAmountOut: uint256,
    _tokenPath: DynArray[address, MAX_TOKEN_PATH],
    _poolPath: DynArray[address, MAX_TOKEN_PATH],
    _recipient: address,
    _miniAddys: address[5]
) -> (uint256, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_amountIn` | `uint256` | Amount of input token |
| `_minAmountOut` | `uint256` | Minimum output amount (slippage protection) |
| `_tokenPath` | `DynArray[address, 5]` | Tokens in swap path |
| `_poolPath` | `DynArray[address, 4]` | Pools to route through |
| `_recipient` | `address` | Address to receive output |
| `_miniAddys` | `address[5]` | Helper addresses array |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Actual input amount used |
| `uint256` | Output amount received |
| `uint256` | USD value of transaction |

## Quote Functions

### `getSwapAmountOut`

Calculates expected output for a given input.

```vyper
@view
@external
def getSwapAmountOut(
    _pool: address,
    _tokenIn: address,
    _tokenOut: address,
    _amountIn: uint256
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_pool` | `address` | Pool to query |
| `_tokenIn` | `address` | Input token |
| `_tokenOut` | `address` | Output token |
| `_amountIn` | `uint256` | Input amount |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Expected output amount |

### `getSwapAmountIn`

Calculates required input for a desired output.

```vyper
@view
@external
def getSwapAmountIn(
    _pool: address,
    _tokenIn: address,
    _tokenOut: address,
    _amountOut: uint256
) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Required input amount |

### `getBestSwapAmountOut`

Finds the best pool and output for a swap.

```vyper
@view
@external
def getBestSwapAmountOut(
    _tokenIn: address,
    _tokenOut: address,
    _amountIn: uint256
) -> (address, uint256):
```

#### Returns

| Type | Description |
|------|-------------|
| `address` | Best pool address |
| `uint256` | Best output amount |

### `getBestSwapAmountIn`

Finds the best pool and required input for a swap.

```vyper
@view
@external
def getBestSwapAmountIn(
    _tokenIn: address,
    _tokenOut: address,
    _amountOut: uint256
) -> (address, uint256):
```

#### Returns

| Type | Description |
|------|-------------|
| `address` | Best pool address |
| `uint256` | Required input amount |

## LP Token Functions

### `getLpToken`

Returns the LP token for a pool.

```vyper
@view
@external
def getLpToken(_pool: address) -> address:
```

### `getPoolForLpToken`

Returns the pool for an LP token (reverse lookup).

```vyper
@view
@external
def getPoolForLpToken(_lpToken: address) -> address:
```

## Liquidity Functions

### `getAddLiqAmountsIn`

Calculates optimal amounts for adding liquidity.

```vyper
@view
@external
def getAddLiqAmountsIn(
    _pool: address,
    _tokenA: address,
    _tokenB: address,
    _availAmountA: uint256,
    _availAmountB: uint256
) -> (uint256, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_pool` | `address` | Pool address |
| `_tokenA` | `address` | First token |
| `_tokenB` | `address` | Second token |
| `_availAmountA` | `uint256` | Available amount of token A |
| `_availAmountB` | `uint256` | Available amount of token B |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Token A amount to add |
| `uint256` | Token B amount to add |
| `uint256` | Expected LP tokens |

### `getRemoveLiqAmountsOut`

Calculates expected amounts from removing liquidity.

```vyper
@view
@external
def getRemoveLiqAmountsOut(
    _pool: address,
    _tokenA: address,
    _tokenB: address,
    _lpAmount: uint256
) -> (uint256, uint256):
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Token A amount received |
| `uint256` | Token B amount received |

## Price Functions

### `getPrice`

Returns the price of an asset.

```vyper
@view
@external
def getPrice(_asset: address, _decimals: uint256) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_asset` | `address` | Asset to price |
| `_decimals` | `uint256` | Desired decimal precision |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Asset price |

### `getPriceUnsafe`

Returns price without safety checks (faster but potentially manipulable).

```vyper
@view
@external
def getPriceUnsafe(
    _pool: address,
    _targetToken: address,
    _appraiser: address = empty(address)
) -> uint256:
```

### `getCoreRouterPool`

Returns the main routing pool (typically USDC-WETH).

```vyper
@view
@external
def getCoreRouterPool() -> address:
```

---

## Base Lego Interface

All legos (yield and DEX) must also implement these base functions:

### `hasCapability`

Returns whether lego supports an action type.

```vyper
@view
@external
def hasCapability(_action: ActionType) -> bool:
```

#### Common Action Types for DEX Legos:
- `SWAP` - Token swaps
- `ADD_LIQ` - Add liquidity (standard AMM)
- `REMOVE_LIQ` - Remove liquidity (standard AMM)
- `ADD_LIQ_CONC` - Add concentrated liquidity
- `REMOVE_LIQ_CONC` - Remove concentrated liquidity

### `isYieldLego`

Returns False for DEX legos.

```vyper
@view
@external
def isYieldLego() -> bool:
```

### `isDexLego`

Returns True for DEX legos.

```vyper
@view
@external
def isDexLego() -> bool:
```

### `getRegistries`

Returns external registries used by the lego.

```vyper
@view
@external
def getRegistries() -> DynArray[address, 10]:
```

### `getAccessForLego`

Returns access control requirements for an action.

```vyper
@view
@external
def getAccessForLego(
    _user: address,
    _action: ActionType
) -> (address, String[64], uint256):
```

---

## Data Structures

### SwapRoute Struct (used by LegoTools)

```vyper
struct SwapRoute:
    legoId: uint256
    pool: address
    tokenIn: address
    tokenOut: address
    amountIn: uint256
    amountOut: uint256
```

### SwapInstruction Struct (used by LegoTools)

```vyper
struct SwapInstruction:
    legoId: uint256
    amountIn: uint256
    minAmountOut: uint256
    tokenPath: DynArray[address, 5]
    poolPath: DynArray[address, 4]
```

---

## Implementation Notes

### For Protocol Integrators

When implementing a new DEX lego:

1. **Inherit DexLegoData module** for base functionality
2. **Implement all required functions** from this interface
3. **Handle multi-hop routing** correctly
4. **Return accurate USD values** via Appraiser
5. **Support both single-pool and multi-pool swaps**

### For Developers Using DEX Legos

When using DEX legos:

1. **Use LegoTools for routing** - Finds best routes across all DEXes
2. **Always set minAmountOut** for slippage protection
3. **Check capabilities** before calling actions
4. **Use getBestSwapAmountOut** for optimal routes

### Common Patterns

```python
# Find best swap route
pool, expected_out = lego.getBestSwapAmountOut(token_in, token_out, amount_in)

# Execute swap with slippage protection
min_out = expected_out * 99 // 100  # 1% slippage
actual_in, actual_out, usd_value = lego.swapTokens(
    amount_in,
    min_out,
    [token_in, token_out],
    [pool],
    recipient,
    mini_addys
)

# Calculate liquidity amounts
token_a_amount, token_b_amount, expected_lp = lego.getAddLiqAmountsIn(
    pool, token_a, token_b, available_a, available_b
)
```

### Protocol-Specific Notes

#### Uniswap V2 / Aerodrome Classic
- Standard constant-product AMM
- LP tokens are ERC20
- Single price per pool

#### Uniswap V3 / Aerodrome Slipstream
- Concentrated liquidity
- Positions represented as NFTs
- Multiple price ticks per pool
- Different interface for liquidity operations

#### Curve
- Optimized for stableswaps
- May have 2, 3, or 4+ tokens per pool
- Uses different math for pricing

---

## Routing via LegoTools

For optimal swap routing across multiple DEXes, use [LegoTools](../LegoTools.md):

```python
# Get best route across all DEX legos
routes = lego_tools.getBestSwapRoutesAmountOut(
    token_in, token_out, amount_in, include_lego_ids
)

# Prepare executable instructions
instructions = lego_tools.prepareSwapInstructionsAmountOut(
    slippage, routes
)

# Execute via UserWallet or Vault
wallet.swapTokens(instructions)
```

LegoTools handles:
- Multi-DEX routing
- Multi-hop optimization (via USDC-WETH router pool)
- Slippage calculation
- Instruction preparation
