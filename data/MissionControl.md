# MissionControl Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/data/MissionControl.vy)

## Overview

MissionControl is the central configuration management contract for the Underscore Protocol. As a Department contract, it stores and manages all protocol-wide settings including wallet creation parameters, fee structures, asset configurations, yield settings, and security controls. It serves as the single source of truth for protocol behavior and limits.

**Core Features**:
- **Global Configuration Management**: Store protocol-wide settings for wallets, agents, managers, payees, and cheques
- **Asset-Specific Settings**: Configure per-asset fees, yield parameters, and integration details
- **Security Controls**: Manage creator whitelists, security signers, and locked addresses
- **Fee Structure**: Define swap fees, rewards fees, and ambassador revenue shares
- **Ledger Integration**: Retrieves vault token data from Ledger for yield asset calculations
- **Dynamic Configuration**: Update settings through governance without contract upgrades via [SwitchboardAlpha](../config/SwitchboardAlpha.md) and [SwitchboardBravo](../config/SwitchboardBravo.md)

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                            MissionControl                               |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Configuration Hierarchy                         | |
|  |                                                                   | |
|  |  Global Configs:                                                  | |
|  |    ├─> UserWalletConfig - Default settings for all wallets        | |
|  |    ├─> AgentConfig - Agent creation and defaults                  | |
|  |    ├─> ManagerConfig - Manager periods and activation             | |
|  |    ├─> PayeeConfig - Payee periods and activation                 | |
|  |    └─> ChequeConfig - Cheque limits and timing                    | |
|  |                                                                   | |
|  |  Asset-Specific Configs:                                          | |
|  |    ├─> AssetConfig[asset] - Per-asset overrides                   | |
|  |    ├─> Transaction fees (swap, rewards)                            | |
|  |    ├─> Yield parameters (bonus ratios, fees)                      | |
|  |    └─> Ambassador revenue shares                                   | |
|  |                                                                   | |
|  |  Precedence Rules:                                                | |
|  |    1. Asset-specific config (if hasConfig == true)                | |
|  |    2. Global default config                                       | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                   Ledger Integration                              | |
|  |                                                                   | |
|  |  VaultToken Data (from Ledger):                                   | |
|  |    • legoId - Which lego manages this vault token                 | |
|  |    • underlyingAsset - Base asset for yield tokens                | |
|  |    • decimals - Token decimals                                    | |
|  |    • isRebasing - Whether token is rebasing                       | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                      Security Architecture                         | |
|  |                                                                   | |
|  |  Creator Control:                                                  | |
|  |    • enforceCreatorWhitelist flag per entity type                 | |
|  |    • creatorWhitelist mapping for allowed creators                | |
|  |    • Public creation when whitelist not enforced                  | |
|  |                                                                   | |
|  |  Security Actions:                                                 | |
|  |    • canPerformSecurityAction - Emergency intervention rights      | |
|  |    • isLockedSigner - Prevent specific addresses from actions     | |
|  |    • Switchboard-only updates for all settings                    | |
|  +-------------------------------------------------------------------+ |
+-------------------------------------------------------------------------+
```

## Module Integration

MissionControl implements the Department interface and integrates:
- `Addys` module for address registry management
- `DeptBasics` module for pause functionality

## Data Structures

### ConfigStructs Interface Definitions

#### UserWalletConfig

Global wallet settings including templates, fees, and yield parameters.

```vyper
struct UserWalletConfig:
    walletTemplate: address           # Template for new wallets
    configTemplate: address           # Template for wallet configs
    numUserWalletsAllowed: uint256    # Max wallets (0 = unlimited)
    enforceCreatorWhitelist: bool     # Require whitelisted creators
    minKeyActionTimeLock: uint256     # Min timelock for key actions
    maxKeyActionTimeLock: uint256     # Max timelock for key actions
    depositRewardsAsset: address      # Asset for deposit rewards
    lootClaimCoolOffPeriod: uint256   # Cooldown between loot claims
    txFees: TxFees                    # Transaction fee settings
    ambassadorRevShare: AmbassadorRevShare  # Ambassador revenue shares
    yieldConfig: YieldConfig          # Yield bonus settings
```

#### AssetConfig

Per-asset configuration overrides.

```vyper
struct AssetConfig:
    hasConfig: bool                   # Whether this config is active
    txFees: TxFees                    # Asset-specific fees
    ambassadorRevShare: AmbassadorRevShare  # Asset-specific ambassador shares
    yieldConfig: YieldConfig          # Asset-specific yield settings
```

#### TxFees

Transaction fee structure.

```vyper
struct TxFees:
    swapFee: uint256        # Fee on swaps (basis points)
    stableSwapFee: uint256  # Fee on stablecoin swaps (basis points)
    rewardsFee: uint256     # Fee on rewards claims (basis points)
```

#### AmbassadorRevShare

Revenue share ratios for different action types.

```vyper
struct AmbassadorRevShare:
    swapRatio: uint256      # Share of swap fees to ambassador
    rewardsRatio: uint256   # Share of rewards fees to ambassador
    yieldRatio: uint256     # Share of yield fees to ambassador
```

#### YieldConfig

Yield asset parameters including bonus ratios and performance fees.

```vyper
struct YieldConfig:
    maxYieldIncrease: uint256      # Max yield increase threshold
    performanceFee: uint256        # Performance fee (basis points)
    ambassadorBonusRatio: uint256  # Ambassador bonus multiplier
    bonusRatio: uint256            # User bonus multiplier
    bonusAsset: address            # Asset for bonus distribution
```

#### AgentConfig

Agent creation settings.

```vyper
struct AgentConfig:
    startingAgent: address              # Default agent for new wallets
    startingAgentActivationLength: uint256  # Agent activation period
```

#### ManagerConfig

Manager activation settings.

```vyper
struct ManagerConfig:
    managerPeriod: uint256           # Manager period length
    managerActivationLength: uint256 # Blocks until manager active
    mustHaveUsdValueOnSwaps: bool    # Require USD value tracking
    maxNumSwapsPerPeriod: uint256    # Swap limit per period
    maxSlippageOnSwaps: uint256      # Max slippage (basis points)
    onlyApprovedYieldOpps: bool      # Restrict to approved yield
```

#### PayeeConfig

Payee activation settings.

```vyper
struct PayeeConfig:
    payeePeriod: uint256           # Payee period length
    payeeActivationLength: uint256 # Blocks until payee active
```

#### ChequeConfig

Cheque limits and timing parameters.

```vyper
struct ChequeConfig:
    maxNumActiveCheques: uint256    # Max concurrent active cheques
    instantUsdThreshold: uint256    # USD threshold for instant cheques
    periodLength: uint256           # Cheque period length
    expensiveDelayBlocks: uint256   # Delay for expensive cheques
    defaultExpiryBlocks: uint256    # Default cheque expiry
```

### Local Structs

#### VaultToken

Data retrieved from Ledger for vault tokens.

```vyper
struct VaultToken:
    legoId: uint256           # ID of lego managing this token
    underlyingAsset: address  # Base asset (if yield token)
    decimals: uint256         # Token decimals
    isRebasing: bool          # Whether token is rebasing
```

#### AssetUsdValueConfig

Configuration for USD value calculations.

```vyper
struct AssetUsdValueConfig:
    legoId: uint256           # Lego ID from Ledger
    legoAddr: address         # Resolved lego address
    isYieldAsset: bool        # True if has underlying
    underlyingAsset: address  # Underlying asset address
```

#### ProfitCalcConfig

Configuration for profit calculations.

```vyper
struct ProfitCalcConfig:
    legoId: uint256           # Lego ID from Ledger
    legoAddr: address         # Resolved lego address
    isYieldAsset: bool        # True if has underlying
    underlyingAsset: address  # Underlying asset address
    maxYieldIncrease: uint256 # Max yield increase threshold
    performanceFee: uint256   # Performance fee (basis points)
    isRebasing: bool          # Whether token is rebasing
    decimals: uint256         # Token decimals
```

#### LootDistroConfig

Configuration for loot distribution.

```vyper
struct LootDistroConfig:
    legoId: uint256                      # Lego ID from Ledger
    legoAddr: address                    # Resolved lego address
    underlyingAsset: address             # Underlying asset
    ambassador: address                  # Ambassador address (always empty from MissionControl)
    ambassadorRevShare: AmbassadorRevShare  # Revenue share config
    ambassadorBonusRatio: uint256        # Ambassador bonus ratio
    bonusRatio: uint256                  # User bonus ratio
    bonusAsset: address                  # Bonus distribution asset
```

#### UserWalletCreationConfig

Aggregated configuration for wallet creation.

```vyper
struct UserWalletCreationConfig:
    numUserWalletsAllowed: uint256           # Max wallets allowed
    isCreatorAllowed: bool                   # Creator permission
    walletTemplate: address                  # Wallet template
    configTemplate: address                  # Config template
    startingAgent: address                   # Default agent
    startingAgentActivationLength: uint256   # Agent activation
    managerPeriod: uint256                   # Manager period
    managerActivationLength: uint256         # Manager activation
    mustHaveUsdValueOnSwaps: bool            # USD tracking required
    maxNumSwapsPerPeriod: uint256            # Swap limit
    maxSlippageOnSwaps: uint256              # Max slippage
    onlyApprovedYieldOpps: bool              # Approved yield only
    payeePeriod: uint256                     # Payee period
    payeeActivationLength: uint256           # Payee activation
    chequeMaxNumActiveCheques: uint256       # Max active cheques
    chequeInstantUsdThreshold: uint256       # Instant threshold
    chequePeriodLength: uint256              # Cheque period
    chequeExpensiveDelayBlocks: uint256      # Expensive delay
    chequeDefaultExpiryBlocks: uint256       # Default expiry
    minKeyActionTimeLock: uint256            # Min timelock
    maxKeyActionTimeLock: uint256            # Max timelock
```

## State Variables

### Global Configurations
```vyper
userWalletConfig: public(UserWalletConfig)
# Protocol-wide wallet settings

agentConfig: public(AgentConfig)
# Agent creation configuration

managerConfig: public(ManagerConfig)
# Manager settings

payeeConfig: public(PayeeConfig)
# Payee settings

chequeConfig: public(ChequeConfig)
# Cheque settings
```

### Asset Configurations
```vyper
assetConfig: public(HashMap[address, AssetConfig])
# Per-asset settings (checked via hasConfig flag)

isStablecoin: public(HashMap[address, bool])
# Stablecoin flags for fee logic
```

### Security Settings
```vyper
creatorWhitelist: public(HashMap[address, bool])
# creator -> is whitelisted

canPerformSecurityAction: public(HashMap[address, bool])
# signer -> can perform security action

isLockedSigner: public(HashMap[address, bool])
# signer -> is locked
```

## Constructor

### `__init__`

Initializes MissionControl with optional default configuration.

```vyper
@deploy
def __init__(_undyHq: address, _defaults: address):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyHq` | `address` | UndyHq registry contract address |
| `_defaults` | `address` | Optional Defaults contract for initial config |

#### Behavior

1. Initializes Addys module with UndyHq address
2. Initializes DeptBasics with no minting capability
3. If defaults contract provided, loads all config structs from it

## Global Configuration Functions

### `setUserWalletConfig`

Updates global user wallet configuration.

```vyper
@external
def setUserWalletConfig(_config: UserWalletConfig):
```

#### Access

Only callable by switchboard addresses when not paused.

### `setManagerConfig`

Updates manager configuration.

```vyper
@external
def setManagerConfig(_config: ManagerConfig):
```

### `setPayeeConfig`

Updates payee configuration.

```vyper
@external
def setPayeeConfig(_config: PayeeConfig):
```

### `setChequeConfig`

Updates cheque configuration.

```vyper
@external
def setChequeConfig(_config: ChequeConfig):
```

### `getUserWalletCreationConfig`

Gets aggregated configuration for wallet creation.

```vyper
@view
@external
def getUserWalletCreationConfig(_creator: address) -> UserWalletCreationConfig:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_creator` | `address` | Address attempting to create wallet |

#### Returns

| Type | Description |
|------|-------------|
| `UserWalletCreationConfig` | Aggregated creation configuration |

#### Behavior

Combines settings from:
- UserWalletConfig (templates, limits, timelocks)
- ManagerConfig (periods, slippage, swap limits)
- PayeeConfig (periods)
- AgentConfig (starting agent)
- ChequeConfig (limits, thresholds)

### `getDepositRewardsAsset`

Gets the configured deposit rewards asset.

```vyper
@view
@external
def getDepositRewardsAsset() -> address:
```

### `getLootClaimCoolOffPeriod`

Gets the loot claim cooldown period.

```vyper
@view
@external
def getLootClaimCoolOffPeriod() -> uint256:
```

## Agent Configuration Functions

### `setAgentConfig`

Updates agent configuration.

```vyper
@external
def setAgentConfig(_config: AgentConfig):
```

### `setStarterAgent`

Updates the default starting agent only (without full config update).

```vyper
@external
def setStarterAgent(_agent: address):
```

## Asset Configuration Functions

### `setAssetConfig`

Sets configuration for a specific asset.

```vyper
@external
def setAssetConfig(_asset: address, _config: AssetConfig):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_asset` | `address` | Asset address to configure |
| `_config` | `AssetConfig` | Asset configuration struct |

### `setIsStablecoin`

Marks an asset as a stablecoin.

```vyper
@external
def setIsStablecoin(_asset: address, _isStablecoin: bool):
```

## Fee Functions

### `getSwapFee`

Gets swap fee for a token pair.

```vyper
@view
@external
def getSwapFee(_tokenIn: address, _tokenOut: address) -> uint256:
```

#### Fee Logic

1. If both tokens are stablecoins: use `stableSwapFee` from userWalletConfig
2. If output token has asset config (hasConfig == true): use asset-specific `swapFee`
3. Otherwise: use global default `swapFee` from userWalletConfig

### `getRewardsFee`

Gets rewards claiming fee for an asset.

```vyper
@view
@external
def getRewardsFee(_asset: address) -> uint256:
```

#### Fee Logic

1. If asset has config (hasConfig == true): use asset-specific `rewardsFee`
2. Otherwise: use global default `rewardsFee` from userWalletConfig

## Helper Functions

### `getProfitCalcConfig`

Gets profit calculation configuration for an asset.

```vyper
@view
@external
def getProfitCalcConfig(_asset: address) -> ProfitCalcConfig:
```

#### Behavior

1. Checks asset-specific config first (if hasConfig == true)
2. Falls back to global userWalletConfig for maxYieldIncrease and performanceFee
3. Retrieves VaultToken data from Ledger
4. Resolves legoAddr from LegoBook using legoId

### `getAssetUsdValueConfig`

Gets USD value calculation configuration.

```vyper
@view
@external
def getAssetUsdValueConfig(_asset: address) -> AssetUsdValueConfig:
```

#### Behavior

1. Retrieves VaultToken data from Ledger
2. Resolves legoAddr from LegoBook using legoId
3. Determines if yield asset based on underlyingAsset presence

### `getLootDistroConfig`

Gets loot distribution configuration for an asset.

```vyper
@view
@external
def getLootDistroConfig(_asset: address) -> LootDistroConfig:
```

#### Behavior

1. Checks asset-specific config first (if hasConfig == true)
2. Falls back to global userWalletConfig for ambassador shares and bonus settings
3. Retrieves VaultToken data from Ledger
4. Resolves legoAddr from LegoBook

## Security Functions

### `setCanPerformSecurityAction`

Grants or revokes security action permissions.

```vyper
@external
def setCanPerformSecurityAction(_signer: address, _canPerform: bool):
```

### `setCreatorWhitelist`

Adds or removes addresses from creator whitelist.

```vyper
@external
def setCreatorWhitelist(_creator: address, _isWhitelisted: bool):
```

### `setLockedSigner`

Locks or unlocks a signer address.

```vyper
@external
def setLockedSigner(_signer: address, _isLocked: bool):
```

## Internal Functions

### `_isCreatorAllowed`

Checks if creator is allowed based on whitelist settings.

```vyper
@view
@internal
def _isCreatorAllowed(_shouldEnforceWhitelist: bool, _creator: address) -> bool:
```

Returns True if:
- Whitelist is not enforced, OR
- Creator is in whitelist

## Configuration Patterns

### Fee Calculation Pattern
```python
# Get appropriate fee for swap
fee = mission_control.getSwapFee(token_in, token_out)

# Apply fee to swap amount
fee_amount = swap_amount * fee // 10000
net_amount = swap_amount - fee_amount
```

### Security Check Pattern
```python
# Check if address can perform security action
if mission_control.canPerformSecurityAction(caller):
    # Allow emergency intervention
    perform_security_action()

# Check if signer is locked
if mission_control.isLockedSigner(signer):
    raise Exception("Signer is locked")
```

### Creator Validation Pattern
```python
# Get creation config
config = mission_control.getUserWalletCreationConfig(creator)

if config.isCreatorAllowed:
    if config.numUserWalletsAllowed == 0 or existing_wallets < config.numUserWalletsAllowed:
        # Can create wallet
        pass
else:
    raise Exception("Creator not whitelisted")
```

## Security Considerations

### Access Control
- All configuration updates restricted to switchboard addresses
- No direct user access to modify settings
- Pause protection on all setters

### Configuration Safety
- Structured data prevents partial updates
- `hasConfig` flag ensures explicit asset configuration
- Validation in consuming contracts

### Ledger Dependency
- VaultToken data comes from Ledger
- LegoAddr resolution through LegoBook
- Missing data handled gracefully (returns empty address)
