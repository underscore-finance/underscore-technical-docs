# LootDistributor Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore/blob/master/contracts/core/LootDistributor.vy)

## Overview

LootDistributor is the comprehensive rewards and revenue share engine for the Underscore Protocol. As a Department contract, it manages all protocol fee distribution, yield bonuses, ambassador revenue sharing, and deposit rewards, ensuring proper incentive alignment between users, ambassadors, and the protocol while maintaining transparent and secure distribution mechanisms.

**Core Features**:
- **Ambassador Revenue Share**: Automatic distribution of swap, yield, and rewards fees to ambassadors
- **Yield Bonus System**: RIPE token bonuses based on yield performance
- **Deposit Rewards**: Points-based rewards system for protocol deposits
- **RIPE Staking Integration**: Automatic staking of RIPE rewards via RipeTeller
- **Revenue to Governance**: Leftover fees transferred to protocol governance
- **Claimable Loot Management**: Secure tracking and distribution of accumulated rewards

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                            LootDistributor                              |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                      Revenue Flow Architecture                     | |
|  |                                                                   | |
|  |  1. Fee Collection:                                               | |
|  |     ├─> Swap fees from trading activities                         | |
|  |     ├─> Yield performance fees from profit realization            | |
|  |     └─> Rewards fees from external claims                        | |
|  |                                                                   | |
|  |  2. Ambassador Revenue Share:                                     | |
|  |     ├─> Calculate ambassador's share based on action type        | |
|  |     ├─> Add to claimable loot for ambassador                     | |
|  |     └─> Leftover transferred to governance                       | |
|  |                                                                   | |
|  |  3. Yield Bonus Distribution:                                     | |
|  |     ├─> Check eligibility via YieldLego                           | |
|  |     ├─> Convert yield to USD value                                | |
|  |     ├─> Convert USD to RIPE token amount                          | |
|  |     ├─> Distribute to user and ambassador                        | |
|  |     └─> Respect available balance limits                         | |
|  |                                                                   | |
|  |  4. RIPE Rewards:                                                 | |
|  |     ├─> Optional auto-staking via RipeTeller                     | |
|  |     ├─> Configurable stake ratio                                  | |
|  |     └─> Remaining amount sent directly to user                   | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                      Points Calculation Logic                      | |
|  |                                                                   | |
|  |  Deposit Points = USD Value × Blocks Held / 10^18                 | |
|  |                                                                   | |
|  |  User Rewards = Available Rewards × User Points / Global Points   | |
|  |                                                                   | |
|  |  Points stored in Ledger:                                         | |
|  |    • Per-user: usdValue, depositPoints, lastUpdate                | |
|  |    • Global: usdValue, depositPoints, lastUpdate                  | |
|  +-------------------------------------------------------------------+ |
+-------------------------------------------------------------------------+
```

## Module Integration

LootDistributor implements the Department interface and integrates:
- `Addys` module for address registry management
- `DeptBasics` module for pause functionality

## Data Structures

### PointsData

```vyper
struct PointsData:
    usdValue: uint256       # Current USD value of deposits
    depositPoints: uint256  # Accumulated deposit points
    lastUpdate: uint256     # Block number of last update
```

### LootDistroConfig

```vyper
struct LootDistroConfig:
    legoId: uint256                      # Lego ID managing the asset
    legoAddr: address                    # Resolved lego address
    underlyingAsset: address             # Underlying asset (for yield tokens)
    ambassador: address                  # Ambassador address
    ambassadorRevShare: AmbassadorRevShare  # Revenue share config
    ambassadorBonusRatio: uint256        # Ambassador bonus ratio
    bonusRatio: uint256                  # User bonus ratio
    bonusAsset: address                  # RIPE token address
```

### DepositRewards

```vyper
struct DepositRewards:
    asset: address   # Reward asset (typically RIPE)
    amount: uint256  # Total rewards available
```

## State Variables

### Claimable Loot Tracking
```vyper
lastClaim: public(HashMap[address, uint256])
# user -> last claim block

totalClaimableLoot: public(HashMap[address, uint256])
# asset -> total claimable amount

claimableLoot: public(HashMap[address, HashMap[address, uint256]])
# user -> asset -> claimable amount

claimableAssets: public(HashMap[address, HashMap[uint256, address]])
# user -> index -> asset address

indexOfClaimableAsset: public(HashMap[address, HashMap[address, uint256]])
# user -> asset -> index

numClaimableAssets: public(HashMap[address, uint256])
# user -> number of claimable assets
```

### Deposit Rewards
```vyper
depositRewards: public(DepositRewards)
# Current deposit rewards pool

ripeStakeRatio: public(uint256)
# Percentage of RIPE rewards to stake (basis points)

ripeLockDuration: public(uint256)
# Lock duration for staked RIPE
```

### Immutables
```vyper
RIPE_TOKEN: public(immutable(address))
# RIPE governance token address

RIPE_REGISTRY: public(immutable(address))
# Ripe Protocol registry address
```

## Constants

```vyper
HUNDRED_PERCENT: constant(uint256) = 100_00  # 100.00%
EIGHTEEN_DECIMALS: constant(uint256) = 10 ** 18
MAX_DEREGISTER_ASSETS: constant(uint256) = 20
RIPE_TELLER_ID: constant(uint256) = 17
```

## Constructor

### `__init__`

Initializes LootDistributor with RIPE integration.

```vyper
@deploy
def __init__(
    _undyHq: address,
    _ripeToken: address,
    _ripeRegistry: address,
    _ripeStakeRatio: uint256,
    _ripeLockDuration: uint256,
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyHq` | `address` | UndyHq registry contract address |
| `_ripeToken` | `address` | RIPE governance token address |
| `_ripeRegistry` | `address` | Ripe Protocol registry address |
| `_ripeStakeRatio` | `uint256` | Initial stake ratio (basis points) |
| `_ripeLockDuration` | `uint256` | Initial lock duration |

## Revenue Flow Functions

### `addLootFromSwapOrRewards`

Collects fees from swap or rewards actions and distributes to ambassador.

```vyper
@external
def addLootFromSwapOrRewards(
    _asset: address,
    _feeAmount: uint256,
    _action: ActionType,
    _missionControl: address = empty(address),
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_asset` | `address` | Fee asset address |
| `_feeAmount` | `uint256` | Fee amount to collect |
| `_action` | `ActionType` | Type of action (SWAP or REWARDS) |
| `_missionControl` | `address` | Optional MissionControl address |

#### Behavior

1. Validates caller is a user wallet
2. Transfers fee from caller to this contract
3. If ambassador exists, calculates and distributes ambassador share
4. Transfers leftover fee to governance

#### Events Emitted

- `TransactionFeePaid` - Records fee payment from user
- `AmbassadorTxFeePaid` - Records ambassador's share
- `RevenueTransferredToGov` - Records transfer to governance

### `addLootFromYieldProfit`

Handles yield profit distribution including fees and bonuses.

```vyper
@external
def addLootFromYieldProfit(
    _asset: address,
    _feeAmount: uint256,
    _yieldRealized: uint256,
    _missionControl: address = empty(address),
    _appraiser: address = empty(address),
    _legoBook: address = empty(address),
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_asset` | `address` | Yield asset address |
| `_feeAmount` | `uint256` | Performance fee amount (may be 0) |
| `_yieldRealized` | `uint256` | Total yield realized |
| `_missionControl` | `address` | Optional MissionControl |
| `_appraiser` | `address` | Optional Appraiser |
| `_legoBook` | `address` | Optional LegoBook |

#### Behavior

1. Validates caller is a user wallet
2. Logs performance fee (already transferred to contract)
3. Distributes ambassador fee share if applicable
4. Transfers leftover to governance
5. If eligible, distributes yield bonus in RIPE tokens

#### Events Emitted

- `YieldPerformanceFeePaid` - Records yield fee
- `AmbassadorTxFeePaid` - Records ambassador's share
- `RevenueTransferredToGov` - Records transfer to governance
- `YieldBonusPaid` - Records RIPE bonus distribution

## Yield Bonus System

### Internal: `_handleYieldBonus`

Calculates and distributes RIPE token bonuses based on yield.

#### Bonus Calculation Flow

1. Check eligibility via `YieldLego.isEligibleForYieldBonus()`
2. Convert yield to underlying amount (if vault token)
3. Get USD value via Appraiser
4. Convert USD to RIPE amount via `Appraiser.getAssetAmountFromRipe()`
5. Calculate user bonus: `yield × bonusRatio`
6. Calculate ambassador bonus: `yield × ambassadorBonusRatio`
7. Check available balance (excluding reserved amounts)
8. Add to claimable loot

## Claim Functions

### `claimRevShareAndBonusLoot`

Claims accumulated revenue share and bonus rewards.

```vyper
@external
def claimRevShareAndBonusLoot(_user: address) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Number of assets claimed |

#### Behavior

1. Validates claim permissions
2. Gets RIPE staking parameters
3. Iterates through all claimable assets
4. For RIPE tokens: applies stake ratio via RipeTeller
5. For other tokens: direct transfer
6. Deregisters empty asset slots
7. Updates last claim block

### `claimDepositRewards`

Claims accumulated deposit rewards based on points.

```vyper
@external
def claimDepositRewards(_user: address) -> uint256:
```

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Amount of rewards claimed |

#### Behavior

1. Validates this is current LootDistributor
2. Updates user deposit points
3. Calculates pro-rata share: `rewards × userPoints / globalPoints`
4. Applies RIPE staking if applicable
5. Resets user points to 0
6. Updates global points

### `claimAllLoot`

Claims both revenue share/bonuses and deposit rewards.

```vyper
@external
def claimAllLoot(_user: address) -> bool:
```

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if any rewards were claimed |

### `getClaimableLootForAsset`

Gets claimable amount for specific asset.

```vyper
@view
@external
def getClaimableLootForAsset(_user: address, _asset: address) -> uint256:
```

### `getClaimableDepositRewards`

Calculates expected deposit rewards without claiming.

```vyper
@view
@external
def getClaimableDepositRewards(_user: address) -> uint256:
```

## Deposit Points Functions

### `updateDepositPoints`

Updates deposit points for a user (switchboard only).

```vyper
@external
def updateDepositPoints(_user: address):
```

### `updateDepositPointsWithNewValue`

Updates points with a new USD value.

```vyper
@external
def updateDepositPointsWithNewValue(_user: address, _newUsdValue: uint256):
```

#### Access

- User wallets
- Valid wallet configurations

#### Behavior

1. Gets current user and global points from Ledger
2. Calculates new points based on time elapsed
3. Updates user usdValue if changed
4. Updates global usdValue (subtract old, add new)
5. Saves to Ledger

### `updateDepositPointsOnEjection`

Updates deposit points when wallet is ejected (sets USD value to 0).

```vyper
@external
def updateDepositPointsOnEjection(_user: address):
```

### `getLatestDepositPoints`

Calculates accumulated points since last update.

```vyper
@view
@external
def getLatestDepositPoints(_usdValue: uint256, _lastUpdate: uint256) -> uint256:
```

#### Formula

```
Points = USD Value × (Current Block - Last Update) / 10^18
```

## Deposit Rewards Management

### `addDepositRewards`

Adds rewards to the deposit rewards pool.

```vyper
@external
def addDepositRewards(_asset: address, _amount: uint256):
```

#### Access

Anyone can add rewards (typically protocol treasury)

#### Behavior

1. Validates asset matches configured depositRewardsAsset
2. Transfers tokens from caller
3. Adds to deposit rewards pool

### `recoverDepositRewards`

Recovers deposit rewards (emergency function).

```vyper
@external
def recoverDepositRewards(_recipient: address):
```

#### Access

Only callable by switchboard addresses

## RIPE Staking Integration

### Internal: `_handleRipeRewards`

Handles RIPE token distribution with optional staking.

```vyper
@internal
def _handleRipeRewards(
    _user: address,
    _amount: uint256,
    _ripeStakeRatio: uint256,
    _ripeLockDuration: uint256,
    _ripeToken: address,
    _ripeTeller: address,
):
```

#### Behavior

1. If stake ratio is 0: direct transfer to user
2. Otherwise:
   - Calculate stake amount: `amount × stakeRatio / 100_00`
   - Stake via RipeTeller.depositIntoGovVault()
   - Transfer remaining to user

### `setRipeRewardsConfig`

Updates RIPE staking configuration.

```vyper
@external
def setRipeRewardsConfig(_ripeStakeRatio: uint256, _ripeLockDuration: uint256):
```

#### Access

Only callable by switchboard addresses

## Administrative Functions

### `adjustLoot`

Adjusts claimable loot for a user (can only reduce, not increase).

```vyper
@external
def adjustLoot(_user: address, _asset: address, _newClaimable: uint256) -> bool:
```

#### Access

Only callable by switchboard addresses

#### Use Cases

- Anti-fraud adjustments
- Security interventions

## View Functions

### `getTotalClaimableAssets`

Gets count of claimable assets with non-zero balance.

```vyper
@view
@external
def getTotalClaimableAssets(_user: address) -> uint256:
```

### `getSwapFee`

Gets swap fee for a token pair (delegates to MissionControl).

```vyper
@view
@external
def getSwapFee(_user: address, _tokenIn: address, _tokenOut: address, _missionControl: address = empty(address)) -> uint256:
```

### `getRewardsFee`

Gets rewards fee for an asset (delegates to MissionControl).

```vyper
@view
@external
def getRewardsFee(_user: address, _asset: address, _missionControl: address = empty(address)) -> uint256:
```

### `validateCanClaimLoot`

Checks if caller can claim loot for user.

```vyper
@view
@external
def validateCanClaimLoot(_user: address, _caller: address) -> bool:
```

#### Validation Logic

1. User must be registered wallet
2. Check cooloff period (unless switchboard)
3. LegoBook always allowed
4. Wallet owner always allowed
5. Manager with `canClaimLoot` permission
6. Switchboard always allowed

### `isValidWalletConfig`

Validates if caller is the wallet configuration for a user wallet.

```vyper
@view
@external
def isValidWalletConfig(_wallet: address, _caller: address) -> bool:
```

## Events

```vyper
event TransactionFeePaid:
    user: indexed(address)
    asset: indexed(address)
    feeAmount: uint256
    action: ActionType

event RevenueTransferredToGov:
    asset: indexed(address)
    amount: uint256
    action: ActionType

event YieldPerformanceFeePaid:
    user: indexed(address)
    asset: indexed(address)
    feeAmount: uint256
    yieldRealized: uint256

event AmbassadorTxFeePaid:
    asset: indexed(address)
    totalFee: uint256
    ambassadorFeeRatio: uint256
    ambassadorFee: uint256
    ambassador: indexed(address)
    action: ActionType

event YieldBonusPaid:
    bonusAsset: indexed(address)
    bonusAmount: uint256
    bonusRatio: uint256
    yieldRealized: uint256
    recipient: indexed(address)
    isAmbassador: bool

event LootAdjusted:
    user: indexed(address)
    asset: indexed(address)
    newClaimable: uint256

event LootClaimed:
    user: indexed(address)
    asset: indexed(address)
    amount: uint256

event DepositRewardsAdded:
    asset: indexed(address)
    addedAmount: uint256
    newTotalAmount: uint256
    adder: indexed(address)

event DepositRewardsClaimed:
    user: indexed(address)
    asset: indexed(address)
    userRewards: uint256
    remainingRewards: uint256

event DepositRewardsRecovered:
    asset: indexed(address)
    recipient: indexed(address)
    amount: uint256

event RipeRewardsConfigSet:
    ripeStakeRatio: uint256
    ripeLockDuration: uint256
```

## Security Considerations

### Access Control
- User wallet validation for fee collection
- Manager permissions for claims
- Switchboard privileges for admin functions
- Cooldown periods for claim security

### Value Protection
- Balance checks before transfers
- Reserved amount tracking for deposit rewards
- Minimum transfer validation
- Can only adjust loot down, never up

### RIPE Integration Safety
- Approval cleared after staking
- Balance checks before transfers
- Graceful failure on pause
