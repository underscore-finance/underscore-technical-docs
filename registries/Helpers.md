# Helpers Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/registries/Helpers.vy)

## Overview

Helpers is a registry contract for managing helper contract addresses within the Underscore Protocol. It provides a time-locked registry pattern for adding, updating, and disabling helper contracts with governance controls.

**Core Features**:
- **Helper Registration**: Register new helper contracts with descriptions
- **Time-Locked Updates**: All registry changes require a time-lock period
- **Address Validation**: Check if an address is a registered helper
- **Governance Controlled**: All operations require governance permissions

## Architecture

Helpers composes multiple protocol modules:

```
+-------------------------------------------------------+
|                       Helpers                          |
|              (Helper Contract Registry)                |
+-------------------------------------------------------+
|                                                       |
|  +-------------------+  +-------------------------+   |
|  |     LocalGov      |  |    AddressRegistry      |   |
|  |   (Governance)    |  |  (Registry Management)  |   |
|  +-------------------+  +-------------------------+   |
|                                                       |
|  +-------------------+  +-------------------------+   |
|  |      Addys        |  |      DeptBasics         |   |
|  | (Addr Resolution) |  |   (Pause/Dept Utils)    |   |
|  +-------------------+  +-------------------------+   |
|                                                       |
+-------------------------------------------------------+
```

## Inherited Interfaces

The contract exports interfaces from its composed modules:

| Module | Interface | Purpose |
|--------|-----------|---------|
| `LocalGov` | `gov.__interface__` | Governance functions |
| `AddressRegistry` | `registry.__interface__` | Registry view functions |
| `Addys` | `addys.__interface__` | Address resolution |
| `DeptBasics` | `deptBasics.__interface__` | Department utilities |

## Constructor

### `__init__`

```vyper
@deploy
def __init__(
    _undyHq: address,
    _tempGov: address,
    _minRegistryTimeLock: uint256,
    _maxRegistryTimeLock: uint256,
):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_undyHq` | `address` | UndyHq registry address |
| `_tempGov` | `address` | Temporary governance address |
| `_minRegistryTimeLock` | `uint256` | Minimum time-lock for registry changes |
| `_maxRegistryTimeLock` | `uint256` | Maximum time-lock for registry changes |

#### Initialization

- Initializes governance with no time-lock constraints (0, 0, 0)
- Initializes registry with specified time-lock bounds
- Initializes address resolution via Addys
- Initializes department basics with minting disabled

---

## View Functions

### `isHelpersAddr`

Checks if an address is registered as a valid helper.

```vyper
@view
@external
def isHelpersAddr(_addr: address) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_addr` | `address` | Address to check |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if address is a registered helper |

---

## Registry Functions

All registry functions require governance permissions and are subject to time-lock delays.

### Add New Address

#### `startAddNewAddressToRegistry`

Initiates the process of adding a new helper address.

```vyper
@external
def startAddNewAddressToRegistry(_addr: address, _description: String[64]) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_addr` | `address` | Helper contract address to add |
| `_description` | `String[64]` | Description of the helper |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if initiation succeeded |

#### Access

Governance only (not paused)

---

#### `confirmNewAddressToRegistry`

Confirms adding a new helper after time-lock expires.

```vyper
@external
def confirmNewAddressToRegistry(_addr: address) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_addr` | `address` | Helper address to confirm |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Registry ID assigned to the helper |

#### Access

Governance only (not paused)

---

#### `cancelNewAddressToRegistry`

Cancels a pending helper registration.

```vyper
@external
def cancelNewAddressToRegistry(_addr: address) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_addr` | `address` | Helper address to cancel |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if cancellation succeeded |

#### Access

Governance only (not paused)

---

### Update Existing Address

#### `startAddressUpdateToRegistry`

Initiates updating an existing helper to a new address.

```vyper
@external
def startAddressUpdateToRegistry(_regId: uint256, _newAddr: address) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_regId` | `uint256` | Registry ID of helper to update |
| `_newAddr` | `address` | New address for the helper |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if initiation succeeded |

#### Access

Governance only (not paused)

---

#### `confirmAddressUpdateToRegistry`

Confirms an address update after time-lock expires.

```vyper
@external
def confirmAddressUpdateToRegistry(_regId: uint256) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_regId` | `uint256` | Registry ID to confirm update |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if confirmation succeeded |

#### Access

Governance only (not paused)

---

#### `cancelAddressUpdateToRegistry`

Cancels a pending address update.

```vyper
@external
def cancelAddressUpdateToRegistry(_regId: uint256) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_regId` | `uint256` | Registry ID to cancel update |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if cancellation succeeded |

#### Access

Governance only (not paused)

---

### Disable Address

#### `startAddressDisableInRegistry`

Initiates disabling a helper address.

```vyper
@external
def startAddressDisableInRegistry(_regId: uint256) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_regId` | `uint256` | Registry ID of helper to disable |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if initiation succeeded |

#### Access

Governance only (not paused)

---

#### `confirmAddressDisableInRegistry`

Confirms disabling a helper after time-lock expires.

```vyper
@external
def confirmAddressDisableInRegistry(_regId: uint256) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_regId` | `uint256` | Registry ID to confirm disable |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if confirmation succeeded |

#### Access

Governance only (not paused)

---

#### `cancelAddressDisableInRegistry`

Cancels a pending disable operation.

```vyper
@external
def cancelAddressDisableInRegistry(_regId: uint256) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_regId` | `uint256` | Registry ID to cancel disable |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True if cancellation succeeded |

#### Access

Governance only (not paused)

---

## Inherited Functions

### From AddressRegistry

| Function | Description |
|----------|-------------|
| `getAddr(_regId)` | Get address by registry ID |
| `getRegId(_addr)` | Get registry ID by address |
| `numAddrs()` | Get total registered addresses |
| `hasPendingAddition(_addr)` | Check for pending addition |
| `hasPendingUpdate(_regId)` | Check for pending update |
| `hasPendingDisable(_regId)` | Check for pending disable |

### From LocalGov

| Function | Description |
|----------|-------------|
| `owner()` | Get owner address |
| `governance()` | Get governance address |
| `isGovernance(_addr)` | Check if address is governance |

### From DeptBasics

| Function | Description |
|----------|-------------|
| `isPaused()` | Check if contract is paused |

---

## Access Control

### `_canPerformAction`

Internal function that determines if a caller can perform registry operations.

```vyper
@view
@internal
def _canPerformAction(_caller: address) -> bool:
    return gov._canGovern(_caller) and not deptBasics.isPaused
```

Requirements:
1. Caller must have governance permissions
2. Contract must not be paused

---

## Security Considerations

### Time-Lock Protection
- All registry changes require a time-lock period between initiation and confirmation
- Prevents immediate malicious changes to helper addresses
- Time-lock bounds are set at deployment

### Governance Control
- Only governance addresses can modify the registry
- Pause mechanism can halt all registry operations

### No Minting
- Department basics initialized with minting disabled
- Prevents any token minting from this contract

---

## Common Integration Patterns

### Checking Helper Validity

```python
helpers = Helpers(helpers_address)

# Check if an address is a registered helper
if helpers.isHelpersAddr(some_contract):
    print("Valid helper contract")
else:
    print("Not a registered helper")
```

### Adding a New Helper

```python
helpers = Helpers(helpers_address)

# Step 1: Initiate addition (governance only)
helpers.startAddNewAddressToRegistry(
    new_helper_address,
    "LevgVaultTools"
)

# Wait for time-lock period...

# Step 2: Confirm addition (governance only)
reg_id = helpers.confirmNewAddressToRegistry(new_helper_address)
print(f"Helper registered with ID: {reg_id}")
```

### Updating a Helper Address

```python
helpers = Helpers(helpers_address)

# Step 1: Get current registry ID
reg_id = helpers.getRegId(old_helper_address)

# Step 2: Initiate update (governance only)
helpers.startAddressUpdateToRegistry(reg_id, new_helper_address)

# Wait for time-lock period...

# Step 3: Confirm update (governance only)
helpers.confirmAddressUpdateToRegistry(reg_id)
```

### Disabling a Helper

```python
helpers = Helpers(helpers_address)

# Step 1: Get registry ID
reg_id = helpers.getRegId(helper_to_disable)

# Step 2: Initiate disable (governance only)
helpers.startAddressDisableInRegistry(reg_id)

# Wait for time-lock period...

# Step 3: Confirm disable (governance only)
helpers.confirmAddressDisableInRegistry(reg_id)
```
