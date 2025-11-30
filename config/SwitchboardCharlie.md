# SwitchboardCharlie Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/config/SwitchboardCharlie.vy)

## Overview

SwitchboardCharlie is a specialized administrative contract for managing Earn Vault configurations and yield protocol settings. It provides timelocked governance functions for critical vault operations and immediate-execution functions for emergency security actions.

**Core Features**:
- **Vault Configuration Management**: Approved vault tokens, performance fees, deposit limits
- **Yield Protocol Management**: Snapshot pricing, vault token registration on Legos
- **Leveraged Vault Support**: Collateral/leverage vault configuration, slippage settings
- **Timelock Protection**: Critical changes require timelock delay
- **Emergency Actions**: Immediate execution for security-critical operations
- **Multi-Signature Support**: Integrates with governance modules

## Module Integration

SwitchboardCharlie integrates multiple modules:
- `Addys` - Address registry management
- `LocalGov` - Local governance for permission checks
- `TimeLock` - Action timelock management

## Data Structures

### Action Types (Flag Enum)

```vyper
flag ActionType:
    REDEMPTION_BUFFER
    MIN_YIELD_WITHDRAW_AMOUNT
    SNAPSHOT_PRICE_CONFIG
    APPROVED_VAULT_TOKEN
    APPROVED_VAULT_TOKENS
    PERFORMANCE_FEE
    DEFAULT_TARGET_VAULT_TOKEN
    MAX_DEPOSIT_AMOUNT
    IS_LEVERAGED_VAULT
    COLLATERAL_VAULT
    LEVERAGE_VAULT
    SLIPPAGES
    LEVG_VAULT_HELPER
    MAX_DEBT_RATIO
    ADD_MANAGER
    REMOVE_MANAGER
    REGISTER_VAULT_TOKEN_ON_LEGO
    SET_MORPHO_REWARDS_ADDR
    SET_EULER_REWARDS_ADDR
    SET_COMP_REWARDS_ADDR
```

### Pending Action Structs

Each timelocked action type has a corresponding struct:

| Struct | Fields |
|--------|--------|
| `PendingRedemptionBuffer` | vaultAddr, buffer |
| `PendingMinYieldWithdrawAmount` | vaultAddr, amount |
| `PendingSnapShotPriceConfig` | legoId, config |
| `PendingApprovedVaultToken` | vaultAddr, vaultToken, isApproved, shouldMaxWithdraw |
| `PendingApprovedVaultTokens` | vaultAddr, vaultTokens[], isApproved, shouldMaxWithdraw |
| `PendingPerformanceFee` | vaultAddr, performanceFee |
| `PendingDefaultTargetVaultToken` | vaultAddr, targetVaultToken |
| `PendingMaxDepositAmount` | vaultAddr, maxDepositAmount |
| `PendingIsLeveragedVault` | vaultAddr, isLeveragedVault |
| `PendingCollateralVault` | vaultAddr, vaultToken, ripeVaultId, legoId, shouldMaxWithdraw |
| `PendingLeverageVault` | vaultAddr, vaultToken, legoId, ripeVaultId, shouldMaxWithdraw |
| `PendingSlippages` | vaultAddr, usdcSlippage, greenSlippage |
| `PendingLevgVaultHelper` | vaultAddr, levgVaultHelper |
| `PendingMaxDebtRatio` | vaultAddr, ratio |
| `PendingAddManager` | vaultAddr, manager |
| `PendingRemoveManager` | vaultAddr, manager |
| `PendingRegisterVaultTokenOnLego` | legoId, asset, vaultToken |

## Constants

- `MAX_VAULT_TOKENS: uint256 = 50` - Maximum vault tokens per batch
- `MAX_ALLOWLIST_BATCH: uint256 = 50` - Maximum users per allowlist batch

## Constructor

### `__init__`

```vyper
@deploy
def __init__(
    _undyHq: address,
    _tempGov: address,
    _minConfigTimeLock: uint256,
    _maxConfigTimeLock: uint256,
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyHq` | `address` | UndyHq registry address |
| `_tempGov` | `address` | Temporary governance address |
| `_minConfigTimeLock` | `uint256` | Minimum timelock duration |
| `_maxConfigTimeLock` | `uint256` | Maximum timelock duration |

## Immediate Actions (No Timelock)

These actions execute immediately and are available to governance or security signers.

### `setCanDeposit`

Enable/disable deposits for a vault.

```vyper
@external
def setCanDeposit(_vaultAddr: address, _canDeposit: bool):
```

**Access**: Governance or security signer (for disabling)

### `setCanWithdraw`

Enable/disable withdrawals for a vault.

```vyper
@external
def setCanWithdraw(_vaultAddr: address, _canWithdraw: bool):
```

**Access**: Governance or security signer (for disabling)

### `setVaultOpsFrozen`

Freeze/unfreeze all vault operations (emergency).

```vyper
@external
def setVaultOpsFrozen(_vaultAddr: address, _isFrozen: bool):
```

**Access**: Governance or security signer (for freezing)

### `setShouldAutoDeposit`

Enable/disable auto-deposit for a vault.

```vyper
@external
def setShouldAutoDeposit(_vaultAddr: address, _shouldAutoDeposit: bool):
```

### `setShouldEnforceAllowlist`

Enable/disable allowlist enforcement for a vault.

```vyper
@external
def setShouldEnforceAllowlist(_vaultAddr: address, _shouldEnforce: bool):
```

### `setAllowed`

Add/remove user from vault allowlist.

```vyper
@external
def setAllowed(_vaultAddr: address, _user: address, _isAllowed: bool):
```

### `setAllowedBatch`

Batch add/remove users from vault allowlist.

```vyper
@external
def setAllowedBatch(_vaultAddr: address, _users: DynArray[address, MAX_ALLOWLIST_BATCH], _isAllowed: bool):
```

### `addPriceSnapshot`

Add price snapshot for a yield lego.

```vyper
@external
def addPriceSnapshot(_legoId: uint256, _vaultToken: address) -> bool:
```

### `deregisterVaultTokenOnLego`

Emergency deregister vault token from a lego (no timelock).

```vyper
@external
def deregisterVaultTokenOnLego(_legoId: uint256, _asset: address, _vaultToken: address) -> uint256:
```

### `updateYieldPosition`

Update yield position for a vault.

```vyper
@external
def updateYieldPosition(_vaultAddr: address, _vaultToken: address):
```

### `claimPerformanceFees`

Claim performance fees from a vault.

```vyper
@external
def claimPerformanceFees(_vaultAddr: address) -> uint256:
```

### `sweepLeftovers`

Sweep leftover tokens from a vault.

```vyper
@external
def sweepLeftovers(_vaultAddr: address) -> uint256:
```

### `removeVaultManager`

Emergency remove manager from vault (no timelock).

```vyper
@external
def removeVaultManager(_vaultAddr: address, _manager: address) -> uint256:
```

## Timelocked Actions

These actions require a timelock period before execution.

### `setRedemptionBuffer`

Set redemption buffer for a vault.

```vyper
@external
def setRedemptionBuffer(_vaultAddr: address, _buffer: uint256) -> uint256:
```

**Returns**: Action ID

### `setMinYieldWithdrawAmount`

Set minimum yield withdraw amount for a vault.

```vyper
@external
def setMinYieldWithdrawAmount(_vaultAddr: address, _amount: uint256) -> uint256:
```

### `setSnapShotPriceConfig`

Set snapshot price configuration for a yield lego.

```vyper
@external
def setSnapShotPriceConfig(
    _legoId: uint256,
    _minSnapshotDelay: uint256,
    _maxNumSnapshots: uint256,
    _maxUpsideDeviation: uint256,
    _staleTime: uint256,
) -> uint256:
```

### `setApprovedVaultToken`

Approve/disapprove single vault token.

```vyper
@external
def setApprovedVaultToken(_undyVaultAddr: address, _vaultToken: address, _isApproved: bool, _shouldMaxWithdraw: bool) -> uint256:
```

**Note**: Disapproving executes immediately (emergency)

### `setApprovedVaultTokens`

Batch approve/disapprove vault tokens.

```vyper
@external
def setApprovedVaultTokens(_undyVaultAddr: address, _vaultTokens: DynArray[address, MAX_VAULT_TOKENS], _isApproved: bool, _shouldMaxWithdraw: bool) -> uint256:
```

### `setPerformanceFee`

Set performance fee for a vault.

```vyper
@external
def setPerformanceFee(_vaultAddr: address, _performanceFee: uint256) -> uint256:
```

### `setDefaultTargetVaultToken`

Set default target vault token.

```vyper
@external
def setDefaultTargetVaultToken(_vaultAddr: address, _targetVaultToken: address) -> uint256:
```

### `setMaxDepositAmount`

Set maximum deposit amount for a vault.

```vyper
@external
def setMaxDepositAmount(_vaultAddr: address, _maxDepositAmount: uint256) -> uint256:
```

### `setIsLeveragedVault`

Mark vault as leveraged vault.

```vyper
@external
def setIsLeveragedVault(_vaultAddr: address, _isLeveragedVault: bool) -> uint256:
```

### `setCollateralVault`

Set collateral vault configuration.

```vyper
@external
def setCollateralVault(_vaultAddr: address, _vaultToken: address, _legoId: uint256, _ripeVaultId: uint256, _shouldMaxWithdraw: bool) -> uint256:
```

### `setLeverageVault`

Set leverage vault configuration.

```vyper
@external
def setLeverageVault(_vaultAddr: address, _vaultToken: address, _legoId: uint256, _ripeVaultId: uint256, _shouldMaxWithdraw: bool) -> uint256:
```

### `setSlippagesAllowed`

Set allowed slippage for a leveraged vault.

```vyper
@external
def setSlippagesAllowed(_vaultAddr: address, _usdcSlippage: uint256, _greenSlippage: uint256) -> uint256:
```

**Constraints**: Max 10% each

### `setLevgVaultHelper`

Set leveraged vault helper contract.

```vyper
@external
def setLevgVaultHelper(_vaultAddr: address, _levgVaultHelper: address) -> uint256:
```

### `setMaxDebtRatio`

Set maximum debt ratio for a leveraged vault.

```vyper
@external
def setMaxDebtRatio(_vaultAddr: address, _ratio: uint256) -> uint256:
```

**Constraints**: Max 100%

### `addVaultManager`

Add manager to a vault.

```vyper
@external
def addVaultManager(_vaultAddr: address, _manager: address) -> uint256:
```

### `registerVaultTokenOnLego`

Register vault token on a yield lego.

```vyper
@external
def registerVaultTokenOnLego(_legoId: uint256, _asset: address, _vaultToken: address) -> uint256:
```

### `setMorphoRewardsAddr`

Set Morpho rewards address.

```vyper
@external
def setMorphoRewardsAddr(_legoId: uint256, _rewardsAddr: address) -> uint256:
```

### `setEulerRewardsAddr`

Set Euler rewards address.

```vyper
@external
def setEulerRewardsAddr(_legoId: uint256, _rewardsAddr: address) -> uint256:
```

### `setCompRewardsAddr`

Set Compound rewards address.

```vyper
@external
def setCompRewardsAddr(_legoId: uint256, _rewardsAddr: address) -> uint256:
```

## Execution Functions

### `executePendingAction`

Execute a pending timelocked action.

```vyper
@external
def executePendingAction(_aid: uint256) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_aid` | `uint256` | Action ID to execute |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if executed, False if not ready or expired |

**Access**: Governance only

### `cancelPendingAction`

Cancel a pending timelocked action.

```vyper
@external
def cancelPendingAction(_aid: uint256) -> bool:
```

**Access**: Governance only

## Events

Key events emitted by the contract:

- `PendingRedemptionBufferChange` / `RedemptionBufferSet`
- `PendingApprovedVaultTokenChange` / `ApprovedVaultTokenSet`
- `PendingPerformanceFeeChange` / `PerformanceFeeSet`
- `CanDepositSet`, `CanWithdrawSet`, `VaultOpsFrozenSet`
- `AllowlistUserSet`, `AllowlistBatchSet`
- `PriceSnapshotAdded`, `VaultTokenRegisteredOnLego`
- `ManagerAdded`, `ManagerRemoved`
- And many more...

## Security Considerations

### Permission Levels
1. **Governance**: Full access to all functions
2. **Security Signers**: Can perform emergency actions (freeze, disable, remove)
3. **Public**: No direct access

### Immediate vs Timelocked
- Security-critical removals/disabling: Immediate
- Security-critical additions/enabling: Timelocked
- This asymmetry allows fast response to threats while preventing hasty additions

### Validation
- All vault addresses validated against VaultRegistry
- Vault tokens validated before approval
- Slippage and ratio constraints enforced

## Common Integration Patterns

### Emergency Freeze
```python
# Freeze vault operations immediately
switchboard.setVaultOpsFrozen(vault.address, True, sender=security_signer)

# Later, unfreeze after issue resolved (governance only)
switchboard.setVaultOpsFrozen(vault.address, False, sender=governor)
```

### Adding Approved Vault Token (Timelocked)
```python
# Initiate approval (returns action ID)
action_id = switchboard.setApprovedVaultToken(
    vault.address,
    vault_token.address,
    True,  # isApproved
    True,  # shouldMaxWithdraw
    sender=governor
)

# Wait for timelock...
# Execute pending action
switchboard.executePendingAction(action_id, sender=governor)
```

### Emergency Removal (Immediate)
```python
# Remove vault token immediately (no timelock)
switchboard.setApprovedVaultToken(
    vault.address,
    vault_token.address,
    False,  # isApproved (removal)
    False,
    sender=security_signer
)
```

## Testing

For test examples, see: [`tests/config/`](../../../tests/config/)
