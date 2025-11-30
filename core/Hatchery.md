# Hatchery Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/core/Hatchery.vy)

## Overview

Hatchery is the wallet creation factory for the Underscore Protocol. As a Department contract, it manages the deployment of new user wallets with configurable templates, default settings, and proper initialization, serving as the entry point for new users joining the protocol ecosystem.

**Core Features**:
- **User Wallet Creation**: Deploy wallet and configuration contracts with customizable settings
- **Template System**: Blueprint-based deployment for consistent wallet patterns
- **Default Configuration**: Automatic setup of manager, payee, and cheque settings
- **Ambassador Referral**: Track referrals for new wallet creations
- **Backpack Integration**: Uses WalletBackpack for component addresses

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                                Hatchery                                 |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Wallet Creation Flow                           | |
|  |                                                                   | |
|  |  1. Validation:                                                   | |
|  |     ├─> Check creator permissions (whitelist/switchboard)        | |
|  |     ├─> Verify template addresses are valid                      | |
|  |     ├─> Ensure owner ≠ starting agent                            | |
|  |     └─> Check global wallet limits                               | |
|  |                                                                   | |
|  |  2. Get Backpack Components:                                       | |
|  |     ├─> HighCommand address                                       | |
|  |     ├─> Paymaster address                                         | |
|  |     ├─> ChequeBook address                                        | |
|  |     ├─> Kernel address                                            | |
|  |     ├─> Sentinel address                                          | |
|  |     └─> Migrator address                                          | |
|  |                                                                   | |
|  |  3. Configuration Setup:                                          | |
|  |     ├─> Create default manager settings via HighCommand           | |
|  |     ├─> Create default payee settings via Paymaster               | |
|  |     ├─> Create default cheque settings via ChequeBook             | |
|  |     └─> Setup starting agent settings (if configured)             | |
|  |                                                                   | |
|  |  4. Contract Deployment:                                          | |
|  |     ├─> Deploy UserWalletConfig from blueprint                    | |
|  |     ├─> Deploy UserWallet from blueprint                          | |
|  |     ├─> Link wallet to configuration                              | |
|  |     └─> Register in Ledger                                       | |
|  |                                                                   | |
|  |  5. Post-Creation:                                                | |
|  |     ├─> Set ambassador relationship (if valid)                    | |
|  |     └─> Emit UserWalletCreated event                              | |
|  +-------------------------------------------------------------------+ |
+-------------------------------------------------------------------------+
```

## Module Integration

Hatchery implements the Department interface and integrates:
- `Addys` module for address registry management
- `DeptBasics` module for pause functionality

## Data Structures

### UserWalletCreationConfig

Configuration retrieved from MissionControl for wallet creation.

```vyper
struct UserWalletCreationConfig:
    numUserWalletsAllowed: uint256           # Max wallets allowed (0 = unlimited)
    isCreatorAllowed: bool                   # Creator permission
    walletTemplate: address                  # Wallet blueprint
    configTemplate: address                  # Config blueprint
    startingAgent: address                   # Default agent
    startingAgentActivationLength: uint256   # Agent activation period
    managerPeriod: uint256                   # Manager period length
    managerActivationLength: uint256         # Manager activation delay
    mustHaveUsdValueOnSwaps: bool            # USD tracking required
    maxNumSwapsPerPeriod: uint256            # Swap limit per period
    maxSlippageOnSwaps: uint256              # Max slippage (basis points)
    onlyApprovedYieldOpps: bool              # Approved yield only
    payeePeriod: uint256                     # Payee period length
    payeeActivationLength: uint256           # Payee activation delay
    chequeMaxNumActiveCheques: uint256       # Max active cheques
    chequeInstantUsdThreshold: uint256       # Instant threshold
    chequePeriodLength: uint256              # Cheque period length
    chequeExpensiveDelayBlocks: uint256      # Expensive delay
    chequeDefaultExpiryBlocks: uint256       # Default expiry
    minKeyActionTimeLock: uint256            # Min timelock
    maxKeyActionTimeLock: uint256            # Max timelock
```

## Immutable Variables

```vyper
WETH: public(immutable(address))  # Wrapped ETH address
ETH: public(immutable(address))   # ETH placeholder address
```

## Constructor

### `__init__`

Initializes Hatchery with registry and token addresses.

```vyper
@deploy
def __init__(_undyHq: address, _wethAddr: address, _ethAddr: address):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyHq` | `address` | UndyHq registry contract address |
| `_wethAddr` | `address` | WETH token address |
| `_ethAddr` | `address` | ETH placeholder address |

## Wallet Creation Functions

### `createUserWallet`

Creates a new user wallet with configuration.

```vyper
@external
def createUserWallet(
    _owner: address = msg.sender,
    _ambassador: address = empty(address),
    _groupId: uint256 = 1,
) -> address:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_owner` | `address` | Wallet owner address (defaults to sender) |
| `_ambassador` | `address` | Ambassador wallet for referral |
| `_groupId` | `uint256` | Group identifier for organization |

#### Returns

| Type | Description |
|------|-------------|
| `address` | Deployed wallet address |

#### Access

- Public (if `isCreatorAllowed` in config)
- Switchboard addresses (always allowed)

#### Behavior

1. **Validation Phase**:
   - Check not paused
   - Get UserWalletCreationConfig from MissionControl
   - Ensure starting agent is not the owner
   - Non-switchboard callers must be allowed creators
   - Verify template addresses are valid
   - Check global wallet limit not exceeded

2. **Ambassador Validation**:
   - Ambassador must be a valid user wallet
   - Creator must be whitelisted for ambassador assignment

3. **Get Backpack Components**:
   - HighCommand, Paymaster, ChequeBook
   - Kernel, Sentinel, Migrator

4. **Create Default Settings**:
   - GlobalManagerSettings via HighCommand.createDefaultGlobalManagerSettings()
   - GlobalPayeeSettings via Paymaster.createDefaultGlobalPayeeSettings()
   - ChequeSettings via ChequeBook.createDefaultChequeSettings()
   - StarterAgentSettings via HighCommand.createStarterAgentSettings() (if agent configured)

5. **Deploy Contracts**:
   - Deploy UserWalletConfig from blueprint with all settings
   - Deploy UserWallet from blueprint with config reference
   - Link wallet to configuration via setWallet()

6. **Finalize**:
   - Register wallet in Ledger with ambassador
   - Emit UserWalletCreated event

#### Events Emitted

```vyper
event UserWalletCreated:
    mainAddr: indexed(address)
    configAddr: indexed(address)
    owner: indexed(address)
    agent: address
    ambassador: address
    creator: address
    groupId: uint256
```

#### Example Usage

```python
# Simple wallet creation
wallet = hatchery.createUserWallet(
    sender=user
)

# With ambassador referral
wallet = hatchery.createUserWallet(
    _owner=user.address,
    _ambassador=referrer_wallet.address,
    _groupId=5,
    sender=creator
)
```

## Legacy Functions

### `doesWalletStillHaveTrialFundsWithAddys`

Legacy function for backwards compatibility with older wallets.

```vyper
@view
@external
def doesWalletStillHaveTrialFundsWithAddys(
    _user: address,
    _walletConfig: address,
    _missionControl: address,
    _legoBook: address,
    _appraiser: address,
    _ledger: address,
) -> bool:
    return True  # backwards compatibility (legacy wallets)
```

Trial funds functionality has been removed. This function exists only for backwards compatibility with legacy wallets and always returns `True`.

## Security Considerations

### Creation Controls
- Template validation prevents empty addresses
- Global limits prevent spam creation
- Starting agent cannot be the owner
- Group IDs for organizational boundaries

### Permission Hierarchy
- Switchboard has administrative access (always allowed)
- Public creation controlled via `isCreatorAllowed` flag
- Ambassador assignment requires whitelisted creator

### Template System
- Blueprint-based deployment for consistency
- Immutable templates after deployment
- Configuration separation from implementation

## Common Integration Patterns

### Basic Wallet Creation
```python
# Simple wallet creation (defaults to msg.sender as owner)
wallet = hatchery.createUserWallet()

# With ambassador referral
wallet = hatchery.createUserWallet(
    _ambassador=referrer_wallet.address
)
```

### Wallet Creation with Custom Settings
```python
# Create wallet for another user
wallet = hatchery.createUserWallet(
    _owner=new_user.address,
    _ambassador=referrer.address,
    _groupId=organization_id
)
```

### Pre-Creation Validation
```python
# Get creation config
config = mission_control.getUserWalletCreationConfig(creator)

# Check if can create
if config.isCreatorAllowed:
    # Check global limit
    current_wallets = ledger.numUserWallets()
    if config.numUserWalletsAllowed == 0 or current_wallets < config.numUserWalletsAllowed:
        # Can create wallet
        wallet = hatchery.createUserWallet()
```

### Checking Ambassador Validity
```python
# Ambassador must be:
# 1. A valid user wallet
# 2. Creator must be whitelisted
if ledger.isUserWallet(ambassador) and mission_control.creatorWhitelist(creator):
    wallet = hatchery.createUserWallet(
        _ambassador=ambassador
    )
```

## Configuration Dependencies

The wallet creation process integrates with multiple components:

| Component | Function Called | Purpose |
|-----------|-----------------|---------|
| MissionControl | getUserWalletCreationConfig() | Get creation parameters |
| MissionControl | creatorWhitelist() | Validate ambassador assignment |
| Ledger | numUserWallets() | Check global limits |
| Ledger | isUserWallet() | Validate ambassador |
| HighCommand | createDefaultGlobalManagerSettings() | Manager config |
| HighCommand | createStarterAgentSettings() | Agent config |
| Paymaster | createDefaultGlobalPayeeSettings() | Payee config |
| ChequeBook | createDefaultChequeSettings() | Cheque config |
| WalletBackpack | Various getters | Component addresses |
