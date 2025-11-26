# VaultRegistry Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore/blob/master/contracts/registries/VaultRegistry.vy)

## Overview

VaultRegistry is the central registry and configuration contract for all Underscore vaults (both [EarnVault](../vaults/EarnVault.md) and [LevgVault](../vaults/LevgVault.md)). It manages vault registration, configuration settings, approved vault tokens, user allowlists, and emergency controls. All vault operations reference VaultRegistry for their operational parameters.

**Core Responsibilities**:
- **Vault Registration**: Register and deregister vaults with timelocked processes
- **Configuration Management**: Deposit/withdrawal rules, fees, and limits per vault
- **Vault Token Approval**: Whitelist yield protocol tokens vaults can use
- **User Allowlists**: Control which users can deposit into restricted vaults
- **Emergency Controls**: Freeze operations on compromised vaults

The registry implements the [Department](../modules/DeptBasics.md) pattern with governance controls via [UndyHq](UndyHq.md).

## System Architecture Diagram

```
+-------------------------------------------------------------------------+
|                            VaultRegistry                                 |
|                    (Central Vault Configuration)                        |
+-------------------------------------------------------------------------+
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Vault Configuration                            | |
|  |                                                                   | |
|  |  Per-Vault Settings:                                              | |
|  |    - canDeposit / canWithdraw                                     | |
|  |    - maxDepositAmount                                             | |
|  |    - performanceFee                                               | |
|  |    - redemptionBuffer / minYieldWithdrawAmount                    | |
|  |    - shouldAutoDeposit / defaultTargetVaultToken                  | |
|  |    - isLeveragedVault / isVaultOpsFrozen                         | |
|  |    - shouldEnforceAllowlist                                       | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Vault Token Management                         | |
|  |                                                                   | |
|  |  approvedVaultTokens[]  -->  Whitelist per vault                 | |
|  |  assetVaultTokens[]     -->  Global vault token registry         | |
|  |  Reference counting     -->  Track usage across vaults           | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    User Allowlists                                | |
|  |                                                                   | |
|  |  isAllowed[vault][user]  -->  Per-vault user whitelist           | |
|  |  Batch operations        -->  Efficient allowlist management     | |
|  +-------------------------------------------------------------------+ |
|                                                                         |
|  +-------------------------------------------------------------------+ |
|  |                    Registration Flow                              | |
|  |                                                                   | |
|  |  startAddNewAddressToRegistry()  -->  Initiate with timelock     | |
|  |  confirmNewAddressToRegistry()   -->  Confirm after delay        | |
|  |  cancelNewAddressToRegistry()    -->  Cancel pending             | |
|  +-------------------------------------------------------------------+ |
+-------------------------------------------------------------------------+
```

## Data Structures

### VaultConfig Struct
Complete configuration for a vault:
```vyper
struct VaultConfig:
    canDeposit: bool                  # Whether deposits are enabled
    canWithdraw: bool                 # Whether withdrawals are enabled
    maxDepositAmount: uint256         # Maximum deposit cap (0 = unlimited)
    isVaultOpsFrozen: bool            # Emergency freeze flag
    redemptionBuffer: uint256         # Min % to keep in yield positions (basis points)
    minYieldWithdrawAmount: uint256   # Min withdrawal per position (avoids dust)
    performanceFee: uint256           # Fee on yield profits (basis points)
    shouldAutoDeposit: bool           # Auto-route deposits to yield
    defaultTargetVaultToken: address  # Default yield position for auto-deposit
    isLeveragedVault: bool            # Flag for leveraged vaults
    shouldEnforceAllowlist: bool      # Whether to enforce user allowlist
```

### VaultToken Struct
Information about a vault token:
```vyper
struct VaultToken:
    legoId: uint256         # Lego partner ID
    underlyingAsset: address # Asset in the vault
    decimals: uint256       # Token decimals
    isRebasing: bool        # Whether token rebases
```

### VaultActionData Struct
Context bundle for vault operations:
```vyper
struct VaultActionData:
    ledger: address
    missionControl: address
    legoBook: address
    appraiser: address
    vaultRegistry: address
    vaultAsset: address
    signer: address
    legoId: uint256
    legoAddr: address
```

## State Variables

### Vault Configuration
```vyper
vaultConfigs: public(HashMap[address, VaultConfig])
# Maps vault address to its configuration
```

### Approved Vault Tokens (Per Vault)
```vyper
isApprovedVaultToken: public(HashMap[address, HashMap[address, bool]])
# Maps (vault, vaultToken) to approval status

approvedVaultTokens: public(HashMap[address, HashMap[uint256, address]])
# Maps (vault, index) to vault token address

indexOfApprovedVaultToken: public(HashMap[address, HashMap[address, uint256]])
# Maps (vault, vaultToken) to index

numApprovedVaultTokens: public(HashMap[address, uint256])
# Maps vault to count of approved tokens
```

### Asset Vault Tokens (Global)
```vyper
assetVaultTokens: public(HashMap[address, HashMap[uint256, address]])
# Maps (asset, index) to vault token

indexOfAssetVaultToken: public(HashMap[address, HashMap[address, uint256]])
# Maps (asset, vaultToken) to index

numAssetVaultTokens: public(HashMap[address, uint256])
# Maps asset to count of vault tokens

assetVaultTokenRefCount: public(HashMap[address, HashMap[address, uint256]])
# Reference count for vault tokens across vaults
```

### User Allowlists
```vyper
isAllowed: public(HashMap[address, HashMap[address, bool]])
# Maps (vault, user) to allowlist status
```

### Constants
```vyper
MAX_VAULT_TOKENS: constant(uint256) = 50
HUNDRED_PERCENT: constant(uint256) = 100_00  # 10,000 basis points
MAX_ALLOWLIST_BATCH: constant(uint256) = 50
```

## Vault Registration Functions

### `startAddNewAddressToRegistry`

Initiates vault registration with timelock.

```vyper
@external
def startAddNewAddressToRegistry(
    _undyVaultAddr: address,
    _description: String[64]
) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyVaultAddr` | `address` | Vault contract address |
| `_description` | `String[64]` | Human-readable description |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True on success |

#### Access

Governance only

### `confirmNewAddressToRegistry`

Confirms vault registration after timelock.

```vyper
@external
def confirmNewAddressToRegistry(
    _undyVaultAddr: address,
    _isLeveragedVault: bool,
    _shouldEnforceAllowlist: bool,
    _approvedVaultTokens: DynArray[address, MAX_VAULT_TOKENS],
    _maxDepositAmount: uint256,
    _minYieldWithdrawAmount: uint256,
    _performanceFee: uint256,
    _defaultTargetVaultToken: address,
    _shouldAutoDeposit: bool,
    _canDeposit: bool,
    _canWithdraw: bool,
    _isVaultOpsFrozen: bool,
    _redemptionBuffer: uint256
) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyVaultAddr` | `address` | Vault address |
| `_isLeveragedVault` | `bool` | Whether vault is leveraged |
| `_shouldEnforceAllowlist` | `bool` | Enforce user allowlist |
| `_approvedVaultTokens` | `DynArray[address, 50]` | Initial approved tokens |
| `_maxDepositAmount` | `uint256` | Deposit cap |
| `_minYieldWithdrawAmount` | `uint256` | Min per-position withdrawal |
| `_performanceFee` | `uint256` | Fee on yield (basis points) |
| `_defaultTargetVaultToken` | `address` | Default auto-deposit target |
| `_shouldAutoDeposit` | `bool` | Enable auto-deposit |
| `_canDeposit` | `bool` | Enable deposits |
| `_canWithdraw` | `bool` | Enable withdrawals |
| `_isVaultOpsFrozen` | `bool` | Freeze vault |
| `_redemptionBuffer` | `uint256` | Buffer percentage |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Registry ID |

### `cancelNewAddressToRegistry`

Cancels pending vault registration.

```vyper
@external
def cancelNewAddressToRegistry(_undyVaultAddr: address) -> bool:
```

## Vault Disable Functions

### `startAddressDisableInRegistry`

Initiates vault disabling with timelock.

```vyper
@external
def startAddressDisableInRegistry(_regId: uint256) -> bool:
```

### `confirmAddressDisableInRegistry`

Confirms vault disabling after timelock.

```vyper
@external
def confirmAddressDisableInRegistry(_regId: uint256) -> bool:
```

### `cancelAddressDisableInRegistry`

Cancels pending vault disable.

```vyper
@external
def cancelAddressDisableInRegistry(_regId: uint256) -> bool:
```

## Configuration Setter Functions

### `setCanDeposit`

Enables/disables deposits for a vault.

```vyper
@external
def setCanDeposit(_undyVaultAddr: address, _canDeposit: bool):
```

#### Events Emitted

- `CanDepositSet(vaultAddr, canDeposit)`

### `setCanWithdraw`

Enables/disables withdrawals for a vault.

```vyper
@external
def setCanWithdraw(_undyVaultAddr: address, _canWithdraw: bool):
```

#### Events Emitted

- `CanWithdrawSet(vaultAddr, canWithdraw)`

### `setMaxDepositAmount`

Sets maximum deposit cap.

```vyper
@external
def setMaxDepositAmount(_undyVaultAddr: address, _maxDepositAmount: uint256):
```

#### Events Emitted

- `MaxDepositAmountSet(vaultAddr, maxDepositAmount)`

### `setVaultOpsFrozen`

Emergency freeze for vault operations.

```vyper
@external
def setVaultOpsFrozen(_undyVaultAddr: address, _isFrozen: bool):
```

#### Events Emitted

- `VaultOpsFrozenSet(vaultAddr, isFrozen)`

### `setMinYieldWithdrawAmount`

Sets minimum withdrawal per yield position.

```vyper
@external
def setMinYieldWithdrawAmount(_undyVaultAddr: address, _amount: uint256):
```

#### Events Emitted

- `MinYieldWithdrawAmountSet(vaultAddr, amount)`

### `setShouldAutoDeposit`

Enables/disables auto-deposit to yield.

```vyper
@external
def setShouldAutoDeposit(_undyVaultAddr: address, _shouldAutoDeposit: bool):
```

#### Events Emitted

- `ShouldAutoDepositSet(vaultAddr, shouldAutoDeposit)`

### `setPerformanceFee`

Sets performance fee on yield profits.

```vyper
@external
def setPerformanceFee(_undyVaultAddr: address, _performanceFee: uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_performanceFee` | `uint256` | Fee in basis points (max varies) |

#### Events Emitted

- `PerformanceFeeSet(vaultAddr, performanceFee)`

### `setRedemptionBuffer`

Sets minimum percentage to keep in yield positions.

```vyper
@external
def setRedemptionBuffer(_undyVaultAddr: address, _buffer: uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_buffer` | `uint256` | Buffer in basis points (min 200 = 2%) |

#### Events Emitted

- `RedemptionBufferSet(vaultAddr, buffer)`

### `setDefaultTargetVaultToken`

Sets default yield position for auto-deposit.

```vyper
@external
def setDefaultTargetVaultToken(_undyVaultAddr: address, _targetVaultToken: address):
```

#### Events Emitted

- `DefaultTargetVaultTokenSet(vaultAddr, targetVaultToken)`

### `setIsLeveragedVault`

Marks vault as leveraged or non-leveraged.

```vyper
@external
def setIsLeveragedVault(_undyVaultAddr: address, _isLeveragedVault: bool):
```

#### Events Emitted

- `IsLeveragedVaultSet(vaultAddr, isLeveragedVault)`

## Allowlist Management Functions

### `setShouldEnforceAllowlist`

Enables/disables allowlist enforcement.

```vyper
@external
def setShouldEnforceAllowlist(_undyVaultAddr: address, _shouldEnforce: bool):
```

#### Events Emitted

- `ShouldEnforceAllowlistSet(undyVault, shouldEnforce)`

### `setAllowed`

Updates allowlist status for a single user.

```vyper
@external
def setAllowed(_undyVaultAddr: address, _user: address, _isAllowed: bool):
```

#### Events Emitted

- `AllowlistSet(undyVault, user, isAllowed)`

### `setAllowedBatch`

Updates allowlist status for multiple users.

```vyper
@external
def setAllowedBatch(
    _undyVaultAddr: address,
    _users: DynArray[address, MAX_ALLOWLIST_BATCH],
    _isAllowed: bool
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_users` | `DynArray[address, 50]` | User addresses to update |
| `_isAllowed` | `bool` | Allowlist status for all users |

## Vault Token Approval Functions

### `setApprovedVaultToken`

Approves or revokes a single vault token.

```vyper
@external
def setApprovedVaultToken(
    _undyVaultAddr: address,
    _vaultToken: address,
    _isApproved: bool,
    _shouldMaxWithdraw: bool
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyVaultAddr` | `address` | Vault address |
| `_vaultToken` | `address` | Vault token to approve/revoke |
| `_isApproved` | `bool` | Approval status |
| `_shouldMaxWithdraw` | `bool` | Withdraw all if revoking |

#### Events Emitted

- `ApprovedVaultTokenSet(undyVaultAddr, underlyingAsset, vaultToken, isApproved, shouldMaxWithdraw)`

### `setApprovedVaultTokens`

Batch approves or revokes vault tokens.

```vyper
@external
def setApprovedVaultTokens(
    _undyVaultAddr: address,
    _vaultTokens: DynArray[address, MAX_VAULT_TOKENS],
    _isApproved: bool,
    _shouldMaxWithdraw: bool
):
```

## Query Functions

### Vault Type Queries

```vyper
@view
@external
def isEarnVault(_undyVaultAddr: address) -> bool:

@view
@external
def isLeveragedVault(_undyVaultAddr: address) -> bool:

@view
@external
def isBasicEarnVault(_undyVaultAddr: address) -> bool:
```

### Configuration Queries

```vyper
@view
@external
def hasConfig(_undyVaultAddr: address) -> bool:

@view
@external
def canDeposit(_undyVaultAddr: address) -> bool:

@view
@external
def canWithdraw(_undyVaultAddr: address) -> bool:

@view
@external
def maxDepositAmount(_undyVaultAddr: address) -> uint256:

@view
@external
def isVaultOpsFrozen(_undyVaultAddr: address) -> bool:

@view
@external
def redemptionBuffer(_undyVaultAddr: address) -> uint256:

@view
@external
def minYieldWithdrawAmount(_undyVaultAddr: address) -> uint256:

@view
@external
def getPerformanceFee(_undyVaultAddr: address) -> uint256:

@view
@external
def getDefaultTargetVaultToken(_undyVaultAddr: address) -> address:

@view
@external
def shouldAutoDeposit(_undyVaultAddr: address) -> bool:

@view
@external
def shouldEnforceAllowlist(_undyVaultAddr: address) -> bool:
```

### Allowlist Queries

```vyper
@view
@external
def isUserAllowed(_undyVaultAddr: address, _userAddr: address) -> bool:

@view
@external
def canUserDeposit(_undyVaultAddr: address, _user: address) -> bool:
```

### Vault Token Queries

```vyper
@view
@external
def isApprovedVaultTokenByAddr(_undyVaultAddr: address, _vaultToken: address) -> bool:

@view
@external
def checkVaultApprovals(_undyVaultAddr: address, _vaultToken: address) -> bool:

@view
@external
def getApprovedVaultTokens(_undyVaultAddr: address) -> DynArray[address, MAX_VAULT_TOKENS]:

@view
@external
def getAssetVaultTokens(_asset: address) -> DynArray[address, MAX_VAULT_TOKENS]:

@view
@external
def getNumApprovedVaultTokens(_undyVaultAddr: address) -> uint256:

@view
@external
def getNumAssetVaultTokens(_asset: address) -> uint256:

@view
@external
def isApprovedVaultTokenForAsset(_underlyingAsset: address, _vaultToken: address) -> bool:
```

### Bundle Queries

```vyper
@view
@external
def getVaultConfig(_regId: uint256) -> VaultConfig:

@view
@external
def getVaultConfigByAddr(_undyVaultAddr: address) -> VaultConfig:

@view
@external
def getVaultActionDataBundle(_legoId: uint256, _signer: address) -> VaultActionData:

@view
@external
def getVaultActionDataWithFrozenStatus(
    _legoId: uint256,
    _signer: address,
    _undyVaultAddr: address
) -> (VaultActionData, bool):

@view
@external
def getDepositConfig(
    _undyVaultAddr: address,
    _user: address
) -> (bool, uint256, bool, address):
# Returns: (canDeposit, maxAmount, shouldAutoDeposit, defaultTarget)
```

### Validation Queries

```vyper
@view
@external
def isValidDefaultTargetVaultToken(_undyVaultAddr: address, _targetVaultToken: address) -> bool:

@view
@external
def isValidPerformanceFee(_performanceFee: uint256) -> bool:

@view
@external
def isValidRedemptionBuffer(_buffer: uint256) -> bool:
```

### Lego Queries

```vyper
@view
@external
def getLegoDataFromVaultToken(_vaultToken: address) -> (uint256, address):
# Returns: (legoId, legoAddr)

@view
@external
def getLegoAddrFromVaultToken(_vaultToken: address) -> address:
```

## Events

```vyper
event CanDepositSet:
    vaultAddr: indexed(address)
    canDeposit: bool

event CanWithdrawSet:
    vaultAddr: indexed(address)
    canWithdraw: bool

event MaxDepositAmountSet:
    vaultAddr: indexed(address)
    maxDepositAmount: uint256

event VaultOpsFrozenSet:
    vaultAddr: indexed(address)
    isFrozen: bool

event RedemptionBufferSet:
    vaultAddr: indexed(address)
    buffer: uint256

event MinYieldWithdrawAmountSet:
    vaultAddr: indexed(address)
    amount: uint256

event PerformanceFeeSet:
    vaultAddr: indexed(address)
    performanceFee: uint256

event DefaultTargetVaultTokenSet:
    vaultAddr: indexed(address)
    targetVaultToken: address

event ShouldAutoDepositSet:
    vaultAddr: indexed(address)
    shouldAutoDeposit: bool

event IsLeveragedVaultSet:
    vaultAddr: indexed(address)
    isLeveragedVault: bool

event ApprovedVaultTokenSet:
    undyVaultAddr: indexed(address)
    underlyingAsset: indexed(address)
    vaultToken: indexed(address)
    isApproved: bool
    shouldMaxWithdraw: bool

event VaultTokenAdded:
    undyVaultAddr: indexed(address)
    underlyingAsset: indexed(address)
    vaultToken: indexed(address)

event VaultTokenRemoved:
    undyVaultAddr: indexed(address)
    underlyingAsset: indexed(address)
    vaultToken: indexed(address)

event AssetVaultTokenAdded:
    asset: indexed(address)
    vaultToken: indexed(address)

event AssetVaultTokenRemoved:
    asset: indexed(address)
    vaultToken: indexed(address)

event ShouldEnforceAllowlistSet:
    undyVault: indexed(address)
    shouldEnforce: bool

event AllowlistSet:
    undyVault: indexed(address)
    user: indexed(address)
    isAllowed: bool
```

## Configuration Defaults & Constraints

### Performance Fee
- Validated by `isValidPerformanceFee()`
- Maximum varies by vault type
- Typically 20% (2000 basis points) for earn vaults

### Redemption Buffer
- Minimum: 200 basis points (2%)
- Validated by `isValidRedemptionBuffer()`
- Ensures some liquidity stays in yield positions

### Auto-Deposit
- Requires valid `defaultTargetVaultToken`
- Target must be in approved vault tokens list

## Security Considerations

### Timelock Protection
- Vault registration requires timelock
- Vault disabling requires timelock
- Prevents instant configuration attacks

### Allowlist Enforcement
- Can restrict deposits to whitelisted users
- Useful for compliance or beta testing
- Batch operations capped at 50 users

### Emergency Controls
- `isVaultOpsFrozen` immediately stops operations
- Can disable deposits/withdrawals independently
- No timelock on freeze (emergency action)

### Vault Token Management
- Reference counting prevents orphaned tokens
- Withdrawal triggered on token removal
- Global asset registry for cross-vault queries
