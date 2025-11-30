# UnderscoreLego Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/legos/UnderscoreLego.vy)

## Overview

UnderscoreLego is a specialized Lego partner that integrates with Underscore's own earn vault system. Unlike external yield protocol integrations, UnderscoreLego connects user wallets to Underscore's [EarnVault](../vaults/EarnVault.md) products, enabling users to deposit into managed yield strategies through the standard Lego interface.

**Key Characteristics**:
- **Yield-Only**: Supports EARN_DEPOSIT, EARN_WITHDRAW, and REWARDS actions
- **No Debt Management**: Unlike RipeLego, does not support borrowing/lending
- **ERC4626 Integration**: Interacts with Underscore vaults via standard ERC4626 interface
- **Loot Rewards**: Integrates with LootDistributor for yield bonus distribution
- **VaultRegistry Validation**: Only works with registered Underscore vaults

This lego enables composability between user wallets and Underscore's managed vault products.

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                          UnderscoreLego                                  |
|              (Underscore Earn Vault Integration)                        |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Capabilities                                   | |
|  |                                                                   | |
|  |  EARN_DEPOSIT   -->  Deposit into earn vaults                    | |
|  |  EARN_WITHDRAW  -->  Withdraw from earn vaults                   | |
|  |  REWARDS        -->  Claim loot rewards                          | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Registry Integration                           | |
|  |                                                                   | |
|  |  VaultRegistry  -->  Validates vault is Underscore earn vault    | |
|  |  LegoBook       -->  Provides lego metadata                      | |
|  |  LootDistributor ->  Handles yield bonus rewards                 | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Yield Position Tracking                        | |
|  |                     (via YieldLegoData module)                    | |
|  |                                                                   | |
|  |  assets[]           -->  Registered underlying assets            | |
|  |  assetOpportunities -->  Vault tokens per asset                  | |
|  |  vaultToAsset       -->  Vault token info mapping                | |
|  +-------------------------------------------------------------------+ |
+-------------------------------------------------------------------------+
```

## State Variables (Immutable)

```vyper
RIPE_REGISTRY: public(immutable(address))
# Ripe registry (for reward claiming)

RIPE_TOKEN: public(immutable(address))
# Ripe governance token
```

## Constants

```vyper
MAX_TOKEN_PATH: constant(uint256) = 5
MAX_PROOFS: constant(uint256) = 25
```

## Capabilities

### `hasCapability`

Returns whether the lego supports an action type.

```vyper
@view
@external
def hasCapability(_action: ActionType) -> bool:
```

**Supported Actions**:
- `EARN_DEPOSIT` - Deposit into earn vaults
- `EARN_WITHDRAW` - Withdraw from earn vaults
- `REWARDS` - Claim loot rewards

**Not Supported**:
- `ADD_COLLATERAL` / `REMOVE_COLLATERAL`
- `BORROW` / `REPAY_DEBT`
- `SWAP`
- Any liquidity operations

### `isYieldLego`

```vyper
@view
@external
def isYieldLego() -> bool:
    return True
```

### `isDexLego`

```vyper
@view
@external
def isDexLego() -> bool:
    return False
```

### `getRegistries`

Returns the VaultRegistry address.

```vyper
@view
@external
def getRegistries() -> DynArray[address, 10]:
```

## Yield Operations

### `depositForYield`

Deposits underlying asset into an Underscore earn vault.

```vyper
@external
def depositForYield(
    _asset: address,
    _amount: uint256,
    _vaultAddr: address,
    _extraData: bytes32,
    _recipient: address,
    _miniAddys: address[5]
) -> (uint256, address, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_asset` | `address` | Asset to deposit |
| `_amount` | `uint256` | Amount to deposit |
| `_vaultAddr` | `address` | Underscore earn vault address |
| `_extraData` | `bytes32` | Unused |
| `_recipient` | `address` | Address to receive vault shares |
| `_miniAddys` | `address[5]` | [ledger, appraiser, ...] |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of asset deposited |
| `address` | Vault share token received |
| `uint256` | Amount of shares received |
| `uint256` | USD value of deposit |

#### Behavior

1. Validates vault is registered in VaultRegistry
2. Validates vault is an Underscore earn vault (not leveraged)
3. Pulls asset from caller
4. Approves vault to spend asset
5. Calls vault.deposit() via ERC4626 interface
6. Returns shares to recipient

#### Events Emitted

- `UnderscoreEarnVaultDeposit`

### `withdrawFromYield`

Withdraws from an Underscore earn vault.

```vyper
@external
def withdrawFromYield(
    _vaultToken: address,
    _amount: uint256,
    _extraData: bytes32,
    _recipient: address,
    _miniAddys: address[5]
) -> (uint256, address, uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_vaultToken` | `address` | Vault share token |
| `_amount` | `uint256` | Amount of shares to redeem |
| `_extraData` | `bytes32` | Unused |
| `_recipient` | `address` | Address to receive underlying |
| `_miniAddys` | `address[5]` | Helper addresses |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Shares burned |
| `address` | Underlying asset received |
| `uint256` | Amount of underlying received |
| `uint256` | USD value of withdrawal |

#### Behavior

1. Validates vault token is registered
2. Pulls shares from caller
3. Calls vault.redeem() via ERC4626 interface
4. Returns underlying to recipient

#### Events Emitted

- `UnderscoreEarnVaultWithdrawal`

## Price Snapshots

### `addPriceSnapshot`

Records a price snapshot for yield tracking.

```vyper
@external
def addPriceSnapshot(_vaultToken: address) -> bool:
```

Used by the yield tracking system to calculate realized profits.

## Reward Functions

### `claimIncentives`

Claims loot rewards with merkle proofs.

```vyper
@external
def claimIncentives(
    _user: address,
    _rewardToken: address,
    _rewardAmount: uint256,
    _proofs: DynArray[bytes32, MAX_PROOFS],
    _miniAddys: address[5]
) -> (uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_user` | `address` | User claiming rewards |
| `_rewardToken` | `address` | Token being claimed |
| `_rewardAmount` | `uint256` | Amount to claim |
| `_proofs` | `DynArray[bytes32, 25]` | Merkle proofs |
| `_miniAddys` | `address[5]` | Helper addresses |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount claimed |
| `uint256` | USD value of rewards |

### `claimRewards`

Claims rewards without merkle proofs.

```vyper
@external
def claimRewards(
    _user: address,
    _rewardToken: address,
    _rewardAmount: uint256,
    _extraData: bytes32,
    _miniAddys: address[5]
) -> (uint256, uint256):
```

### `hasClaimableRewards`

Checks if user has claimable rewards.

```vyper
@view
@external
def hasClaimableRewards(_user: address) -> bool:
```

## Vault Token Registration

### `canRegisterVaultToken`

Validates a vault token can be registered.

```vyper
@view
@external
def canRegisterVaultToken(_asset: address, _vaultToken: address) -> bool:
```

#### Validation

Returns True only if:
1. Vault is registered in VaultRegistry
2. Vault is an Underscore earn vault (not leveraged)
3. Vault's underlying asset matches `_asset`

### `registerVaultTokenLocally`

Registers a vault token in the lego's registry.

```vyper
@external
def registerVaultTokenLocally(
    _asset: address,
    _vaultAddr: address
) -> VaultTokenInfo:
```

### `deregisterVaultTokenLocally`

Removes a vault token from the registry.

```vyper
@external
def deregisterVaultTokenLocally(_asset: address, _vaultAddr: address):
```

## Bonus Eligibility

### `isEligibleForYieldBonus`

Returns True - Underscore vaults are always eligible for yield bonuses.

```vyper
@view
@external
def isEligibleForYieldBonus(_asset: address) -> bool:
    return True
```

This differs from RipeLego which returns False.

## Stub Functions

These functions exist for interface compatibility but return empty/zero values:

### Debt Management (Not Supported)

```vyper
def addCollateral(...) -> (uint256, uint256):
    return (0, 0)

def removeCollateral(...) -> (uint256, uint256):
    return (0, 0)

def borrow(...) -> (uint256, uint256):
    return (0, 0)

def repayDebt(...) -> (uint256, uint256):
    return (0, 0)
```

### Swaps (Not Supported)

```vyper
def swapTokens(...) -> (uint256, uint256, uint256):
    return (0, 0, 0)
```

## Events

```vyper
event UnderscoreEarnVaultDeposit:
    asset: indexed(address)
    assetAmount: uint256
    vaultToken: indexed(address)
    vaultTokenAmount: uint256
    usdValue: uint256
    recipient: indexed(address)

event UnderscoreEarnVaultWithdrawal:
    vaultToken: indexed(address)
    vaultTokenAmount: uint256
    underlyingAsset: indexed(address)
    underlyingAmount: uint256
    usdValue: uint256
    recipient: indexed(address)
```

## Comparison with RipeLego

| Feature | UnderscoreLego | RipeLego |
|---------|----------------|----------|
| Yield Deposits | Yes | Yes (Savings GREEN) |
| Yield Withdrawals | Yes | Yes |
| Collateral Management | No | Yes |
| Borrowing | No | Yes |
| Debt Repayment | No | Yes |
| Token Swaps | No | Yes (PSM) |
| Rewards | Loot Rewards | Ripe Governance |
| Eligible for Bonus | Yes | No |

## Integration Example

```python
# Deposit USDC into Underscore earn vault
underscore_lego = LegoBook.legos(UNDERSCORE_LEGO_ID)

# Check if vault is valid
assert underscore_lego.canRegisterVaultToken(usdc, usdc_earn_vault)

# Deposit
asset_amount, vault_token, shares, usd_value = underscore_lego.depositForYield(
    usdc,
    1000 * 10**6,           # 1000 USDC
    usdc_earn_vault,
    empty(bytes32),
    user_wallet,
    mini_addys
)

# Later: withdraw
shares_burned, underlying, amount, usd_value = underscore_lego.withdrawFromYield(
    vault_token,
    shares,
    empty(bytes32),
    user_wallet,
    mini_addys
)
```

## Security Considerations

### VaultRegistry Validation
- Only registered Underscore vaults are accepted
- Leveraged vaults are explicitly rejected
- Prevents deposits to unauthorized contracts

### Access Control
- All operations go through standard Lego interface
- UserWallet/Vault enforces caller permissions
- No direct external access

### ERC4626 Safety
- Uses standard ERC4626 deposit/redeem
- Shares are burned before underlying sent
- Minimizes reentrancy risk
