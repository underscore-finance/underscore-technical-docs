# DefaultsBase Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/config/DefaultsBase.vy)

## Overview

DefaultsBase is the production defaults configuration contract for the Underscore Protocol. It implements the Defaults interface and provides default configuration values for user wallets, agents, managers, payees, and cheques with production-appropriate settings including fees, security constraints, and activation periods.

**Core Features**:
- **Production Defaults**: Configured for production with fees, whitelist enforcement, and security constraints
- **User Wallet Configuration**: Templates, limits, fees, and yield settings
- **Agent Configuration**: Starting agent and activation period
- **Manager Configuration**: Swap limits, slippage controls, and yield restrictions
- **Payee and Cheque Configuration**: Period lengths and activation delays

## Constants

### Block Time Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `EIGHTEEN_DECIMALS` | `10 ** 18` | Standard 18 decimal multiplier |
| `DAY_IN_BLOCKS` | `43,200` | Blocks per day (~2 second blocks) |
| `WEEK_IN_BLOCKS` | `302,400` | 7 days in blocks |
| `MONTH_IN_BLOCKS` | `1,296,000` | 30 days in blocks |
| `YEAR_IN_BLOCKS` | `15,768,000` | 365 days in blocks |

## Immutable Variables

| Name | Type | Description |
|------|------|-------------|
| `USER_WALLET_TEMPLATE` | `address` | UserWallet blueprint contract |
| `USER_WALLET_CONFIG_TEMPLATE` | `address` | UserWalletConfig blueprint contract |
| `STARTING_AGENT` | `address` | Default agent for new wallets |
| `REWARDS_ASSET` | `address` | Asset used for rewards |

## Constructor

### `__init__`

```vyper
@deploy
def __init__(
    _walletTemplate: address,
    _configTemplate: address,
    _startingAgent: address,
    _rewardsAsset: address,
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_walletTemplate` | `address` | UserWallet blueprint address |
| `_configTemplate` | `address` | UserWalletConfig blueprint address |
| `_startingAgent` | `address` | Default agent address |
| `_rewardsAsset` | `address` | Rewards token address |

## Configuration Functions

### `userWalletConfig`

Returns default user wallet configuration for production.

```vyper
@view
@external
def userWalletConfig() -> cs.UserWalletConfig:
```

#### Returns

| Field | Value | Description |
|-------|-------|-------------|
| `walletTemplate` | USER_WALLET_TEMPLATE | Wallet blueprint |
| `configTemplate` | USER_WALLET_CONFIG_TEMPLATE | Config blueprint |
| `numUserWalletsAllowed` | `0` | No limit (0 = unlimited) |
| `enforceCreatorWhitelist` | `True` | Creator whitelist required |
| `minKeyActionTimeLock` | `DAY_IN_BLOCKS / 2` | 12 hours minimum |
| `maxKeyActionTimeLock` | `2 * WEEK_IN_BLOCKS` | 2 weeks maximum |
| `depositRewardsAsset` | REWARDS_ASSET | Rewards token |
| `lootClaimCoolOffPeriod` | `0` | No cooloff |

**Transaction Fees:**

| Field | Value | Description |
|-------|-------|-------------|
| `swapFee` | `25` | 0.25% swap fee |
| `stableSwapFee` | `25` | 0.25% stable swap fee |
| `rewardsFee` | `20_00` | 20% rewards fee |

**Ambassador Revenue Share:**

| Field | Value | Description |
|-------|-------|-------------|
| `swapRatio` | `0` | No swap revenue share |
| `rewardsRatio` | `0` | No rewards revenue share |
| `yieldRatio` | `0` | No yield revenue share |

**Yield Configuration:**

| Field | Value | Description |
|-------|-------|-------------|
| `maxYieldIncrease` | `5_00` | 5% max yield increase |
| `performanceFee` | `20_00` | 20% performance fee |
| `ambassadorBonusRatio` | `100_00` | 100% ambassador bonus |
| `bonusRatio` | `100_00` | 100% bonus ratio |
| `bonusAsset` | REWARDS_ASSET | Bonus token |

### `agentConfig`

Returns default agent configuration.

```vyper
@view
@external
def agentConfig() -> cs.AgentConfig:
```

#### Returns

| Field | Value | Description |
|-------|-------|-------------|
| `startingAgent` | STARTING_AGENT | Default agent address |
| `startingAgentActivationLength` | `2 * YEAR_IN_BLOCKS` | 2 year activation period |

### `managerConfig`

Returns default manager configuration with production security settings.

```vyper
@view
@external
def managerConfig() -> cs.ManagerConfig:
```

#### Returns

| Field | Value | Description |
|-------|-------|-------------|
| `managerPeriod` | `DAY_IN_BLOCKS` | 1 day period |
| `managerActivationLength` | `MONTH_IN_BLOCKS` | 1 month activation delay |
| `mustHaveUsdValueOnSwaps` | `True` | USD tracking required |
| `maxNumSwapsPerPeriod` | `2` | 2 swaps per day max |
| `maxSlippageOnSwaps` | `5_00` | 5% max slippage |
| `onlyApprovedYieldOpps` | `True` | Approved yield only |

### `payeeConfig`

Returns default payee configuration.

```vyper
@view
@external
def payeeConfig() -> cs.PayeeConfig:
```

#### Returns

| Field | Value | Description |
|-------|-------|-------------|
| `payeePeriod` | `MONTH_IN_BLOCKS` | 1 month period |
| `payeeActivationLength` | `YEAR_IN_BLOCKS` | 1 year activation delay |

### `chequeConfig`

Returns default cheque configuration.

```vyper
@view
@external
def chequeConfig() -> cs.ChequeConfig:
```

#### Returns

| Field | Value | Description |
|-------|-------|-------------|
| `maxNumActiveCheques` | `3` | Max 3 active cheques |
| `instantUsdThreshold` | `100 * 10^18` | $100 instant threshold |
| `periodLength` | `DAY_IN_BLOCKS` | 1 day period |
| `expensiveDelayBlocks` | `DAY_IN_BLOCKS` | 1 day expensive delay |
| `defaultExpiryBlocks` | `2 * DAY_IN_BLOCKS` | 2 day default expiry |

## Comparison with DefaultsLocal

| Setting | DefaultsBase (Production) | DefaultsLocal (Development) |
|---------|---------------------------|----------------------------|
| `numUserWalletsAllowed` | 0 (unlimited) | max_value (unlimited) |
| `enforceCreatorWhitelist` | True | False |
| `maxKeyActionTimeLock` | 2 weeks | 1 week |
| `swapFee` | 25 (0.25%) | 0 |
| `stableSwapFee` | 25 (0.25%) | 0 |
| `rewardsFee` | 20_00 (20%) | 0 |
| `managerPeriod` | 1 day | 1 month |
| `managerActivationLength` | 1 month | 1 year |
| `mustHaveUsdValueOnSwaps` | True | False |
| `maxNumSwapsPerPeriod` | 2 | 0 (unlimited) |
| `maxSlippageOnSwaps` | 5% | 0 (no limit) |
| `onlyApprovedYieldOpps` | True | False |
| `ambassadorBonusRatio` | 100% | 0 |
| `bonusRatio` | 100% | 0 |

## Security Considerations

### Production Settings
- Creator whitelist enforcement prevents unauthorized wallet creation
- Swap limits prevent manipulation
- Slippage controls protect against sandwich attacks
- USD value tracking ensures audit trail
- Approved yield opportunities only

### Fee Structure
- Reasonable fees for sustainability
- Ambassador revenue sharing disabled by default
- Performance fees fund protocol operations

## Common Integration Patterns

### Deploying with Defaults
```python
# Deploy DefaultsBase for production
defaults = DefaultsBase.deploy(
    wallet_template.address,
    config_template.address,
    starting_agent.address,
    rewards_token.address
)

# Register with MissionControl
mission_control.setDefaults(defaults.address)
```

### Reading Configuration
```python
# Get user wallet defaults
wallet_config = defaults.userWalletConfig()
print(f"Max slippage: {wallet_config.maxSlippageOnSwaps / 100}%")

# Get manager defaults
manager_config = defaults.managerConfig()
print(f"Max swaps per period: {manager_config.maxNumSwapsPerPeriod}")
```

## Testing

For test examples, see: [`tests/config/`](../../../tests/config/)
