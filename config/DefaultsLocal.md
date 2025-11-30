# DefaultsLocal Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/config/DefaultsLocal.vy)

## Overview

DefaultsLocal is the local/development defaults configuration contract for the Underscore Protocol. It implements the Defaults interface and provides permissive configuration values suitable for development, testing, and local environments where security constraints and fees are not required.

**Core Features**:
- **Development Defaults**: No fees, no whitelist enforcement, minimal restrictions
- **Permissive Settings**: Unlimited operations with no slippage or swap limits
- **Fast Testing**: No activation delays that would slow down testing
- **Full Access**: All yield opportunities allowed without approval

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

## Constructor

### `__init__`

```vyper
@deploy
def __init__(
    _walletTemplate: address,
    _configTemplate: address,
    _startingAgent: address,
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_walletTemplate` | `address` | UserWallet blueprint address |
| `_configTemplate` | `address` | UserWalletConfig blueprint address |
| `_startingAgent` | `address` | Default agent address |

#### Notes

Unlike DefaultsBase, DefaultsLocal does not require a rewards asset address since all fee-related features are disabled.

## Configuration Functions

### `userWalletConfig`

Returns default user wallet configuration for development with all restrictions removed.

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
| `numUserWalletsAllowed` | `max_value(uint256)` | Unlimited wallets |
| `enforceCreatorWhitelist` | `False` | No whitelist required |
| `minKeyActionTimeLock` | `DAY_IN_BLOCKS / 2` | 12 hours minimum |
| `maxKeyActionTimeLock` | `7 * DAY_IN_BLOCKS` | 1 week maximum |
| `depositRewardsAsset` | `empty(address)` | No rewards token |
| `lootClaimCoolOffPeriod` | `0` | No cooloff |

**Transaction Fees (All Zero):**

| Field | Value | Description |
|-------|-------|-------------|
| `swapFee` | `0` | No swap fee |
| `stableSwapFee` | `0` | No stable swap fee |
| `rewardsFee` | `0` | No rewards fee |

**Ambassador Revenue Share (All Zero):**

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
| `ambassadorBonusRatio` | `0` | No ambassador bonus |
| `bonusRatio` | `0` | No bonus |
| `bonusAsset` | `empty(address)` | No bonus token |

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

Returns permissive manager configuration with no restrictions.

```vyper
@view
@external
def managerConfig() -> cs.ManagerConfig:
```

#### Returns

| Field | Value | Description |
|-------|-------|-------------|
| `managerPeriod` | `MONTH_IN_BLOCKS` | 1 month period |
| `managerActivationLength` | `YEAR_IN_BLOCKS` | 1 year activation delay |
| `mustHaveUsdValueOnSwaps` | `False` | No USD tracking required |
| `maxNumSwapsPerPeriod` | `0` | Unlimited swaps |
| `maxSlippageOnSwaps` | `0` | No slippage limit |
| `onlyApprovedYieldOpps` | `False` | All yield opportunities allowed |

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

## Use Cases

### Local Development
- Quick iteration without fee deductions
- Test all yield opportunities without approval
- Unlimited swap testing

### Integration Testing
- Test wallet operations without whitelist restrictions
- Verify functionality without slippage constraints
- Full protocol coverage testing

### Fork Testing
- Test against mainnet forks with local settings
- Debug without production constraints
- Performance testing without limits

## Common Integration Patterns

### Deploying for Local Development
```python
# Deploy DefaultsLocal for development
defaults = DefaultsLocal.deploy(
    wallet_template.address,
    config_template.address,
    starting_agent.address
)

# Register with MissionControl
mission_control.setDefaults(defaults.address)
```

### Testing Without Restrictions
```python
# Get manager config - no swap limits
manager_config = defaults.managerConfig()
assert manager_config.maxNumSwapsPerPeriod == 0  # Unlimited
assert manager_config.maxSlippageOnSwaps == 0    # No limit
assert manager_config.onlyApprovedYieldOpps == False  # All allowed

# Get user wallet config - no fees
wallet_config = defaults.userWalletConfig()
assert wallet_config.txFees.swapFee == 0
assert wallet_config.txFees.rewardsFee == 0
```

### Switching Between Environments
```python
# Development environment
if is_development:
    defaults = DefaultsLocal.deploy(...)
else:
    defaults = DefaultsBase.deploy(...)

# Same interface regardless of environment
manager_config = defaults.managerConfig()
```

## Security Considerations

### Development Only
- These settings are NOT suitable for production
- No fee collection means no protocol revenue
- No whitelist enforcement allows anyone to create wallets
- No slippage limits expose users to sandwich attacks
- No swap limits allow potential manipulation

### When to Use
- Local development environment
- Unit and integration testing
- Staging environments with fake tokens
- Fork testing scenarios

### When NOT to Use
- Production deployments
- Mainnet or testnet with real value
- Any environment where security matters

## Testing

For test examples, see: [`tests/config/`](../../../tests/config/)
