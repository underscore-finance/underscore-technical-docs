# VaultErc20Token Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/vaults/modules/VaultErc20Token.vy)

## Overview

VaultErc20Token is a Vyper module that provides the ERC20 token implementation for vault shares in [EarnVault](EarnVault.md) and [LevgVault](LevgVault.md). It implements the standard ERC20 interface with additional features for compliance and security including blacklisting, pausing, and EIP-2612 permit functionality.

**Core Features**:
- **ERC20 Compliance**: Full ERC20 token standard implementation
- **EIP-2612 Permit**: Gasless approvals via signatures
- **EIP-712 Signatures**: Domain-separated signature verification
- **Blacklist Support**: Address blacklisting for compliance
- **Pause Capability**: Emergency pause of all transfers
- **ERC-1271 Support**: Smart contract signature verification

This module is imported by EarnVault and LevgVault to provide their share token functionality.

## Data Structures

### EIP-712 Constants
```vyper
EIP712_TYPEHASH: constant(bytes32) = keccak256(
    "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
)

EIP2612_TYPEHASH: constant(bytes32) = keccak256(
    "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
)
```

## State Variables

### Token Metadata (Immutable)
```vyper
TOKEN_NAME: public(immutable(String[64]))
# Token name (e.g., "Underscore USDC Earn Vault")

TOKEN_SYMBOL: public(immutable(String[32]))
# Token symbol (e.g., "usdcEARN")

TOKEN_DECIMALS: public(immutable(uint8))
# Token decimals (typically 18)
```

### EIP-712 Domain (Immutable)
```vyper
NAME_HASH: immutable(bytes32)
# Keccak256 of token name

VERSION_HASH: immutable(bytes32)
# Keccak256 of version string

CACHED_CHAIN_ID: immutable(uint256)
# Chain ID at deployment

CACHED_DOMAIN_SEPARATOR: immutable(bytes32)
# Pre-computed domain separator
```

### Token State
```vyper
balanceOf: public(HashMap[address, uint256])
# User token balances

allowance: public(HashMap[address, HashMap[address, uint256]])
# Spender allowances

totalSupply: public(uint256)
# Total shares issued

nonces: public(HashMap[address, uint256])
# EIP-2612 nonce tracking per address
```

### Governance & Control
```vyper
undyHq: public(address)
# Governance contract address

blacklisted: public(HashMap[address, bool])
# Blacklisted addresses

isPaused: public(bool)
# Global pause flag
```

### Constants
```vyper
VERSION: constant(String[8]) = "v1.0.0"
ECRECOVER_PRECOMPILE: constant(address) = 0x0000000000000000000000000000000000000001
ERC1271_MAGIC_VAL: constant(bytes4) = 0x1626ba7e
```

## Standard ERC20 Functions

### `name`

Returns the token name.

```vyper
@view
@external
def name() -> String[64]:
```

### `symbol`

Returns the token symbol.

```vyper
@view
@external
def symbol() -> String[32]:
```

### `decimals`

Returns the token decimals.

```vyper
@view
@external
def decimals() -> uint8:
```

### `transfer`

Transfers tokens from caller to recipient.

```vyper
@external
def transfer(_recipient: address, _amount: uint256) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_recipient` | `address` | Recipient address |
| `_amount` | `uint256` | Amount to transfer |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True on success |

#### Validation

- Token must not be paused
- Sender must not be blacklisted
- Recipient must not be blacklisted
- Sender must have sufficient balance

#### Events Emitted

- `Transfer(sender, recipient, amount)`

### `transferFrom`

Transfers tokens from sender to recipient using allowance.

```vyper
@external
def transferFrom(_sender: address, _recipient: address, _amount: uint256) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_sender` | `address` | Token owner |
| `_recipient` | `address` | Recipient address |
| `_amount` | `uint256` | Amount to transfer |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True on success |

#### Validation

- Token must not be paused
- Sender must not be blacklisted
- Recipient must not be blacklisted
- Sender must have sufficient balance
- Caller must have sufficient allowance

#### Events Emitted

- `Transfer(sender, recipient, amount)`

## Approval Functions

### `approve`

Sets allowance for spender.

```vyper
@external
def approve(_spender: address, _amount: uint256) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_spender` | `address` | Spender address |
| `_amount` | `uint256` | Allowance amount |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True on success |

#### Validation

- Spender must not be empty address
- Spender must not equal caller

#### Events Emitted

- `Approval(owner, spender, amount)`

### `increaseAllowance`

Increases allowance for spender.

```vyper
@external
def increaseAllowance(_spender: address, _amount: uint256) -> bool:
```

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True on success |

### `decreaseAllowance`

Decreases allowance for spender.

```vyper
@external
def decreaseAllowance(_spender: address, _amount: uint256) -> bool:
```

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True on success |

## EIP-2612 Permit Functions

### `permit`

Approves allowance via off-chain signature (gasless approval).

```vyper
@external
def permit(
    _owner: address,
    _spender: address,
    _value: uint256,
    _deadline: uint256,
    _signature: Bytes[65]
) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_owner` | `address` | Token owner signing the permit |
| `_spender` | `address` | Spender to approve |
| `_value` | `uint256` | Allowance amount |
| `_deadline` | `uint256` | Signature expiry timestamp |
| `_signature` | `Bytes[65]` | EIP-712 signature |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True on success |

#### Validation

- Current timestamp must be <= deadline
- Signature must be valid for owner
- Supports both EOA (ecrecover) and smart contract (ERC-1271) signatures
- Nonce must match owner's current nonce

#### Events Emitted

- `Approval(owner, spender, value)`

### `DOMAIN_SEPARATOR`

Returns the EIP-712 domain separator.

```vyper
@view
@external
def DOMAIN_SEPARATOR() -> bytes32:
```

#### Returns

| Type | Description |
|------|-------------|
| `bytes32` | Domain separator (recomputed if chain ID changed) |

## Burn Functions

### `burn`

Burns caller's tokens, reducing total supply.

```vyper
@external
def burn(_amount: uint256) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_amount` | `uint256` | Amount to burn |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True on success |

#### Events Emitted

- `Transfer(caller, empty(address), amount)`

## Blacklist Functions

### `setBlacklist`

Updates blacklist status for an address.

```vyper
@external
def setBlacklist(_addr: address, _shouldBlacklist: bool) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_addr` | `address` | Address to modify |
| `_shouldBlacklist` | `bool` | True to blacklist, False to remove |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True on success |

#### Access

Governance only (via undyHq)

#### Events Emitted

- `BlacklistModified(addr, isBlacklisted)`

### `burnBlacklistTokens`

Burns tokens from a blacklisted address (compliance seizure).

```vyper
@external
def burnBlacklistTokens(_addr: address, _amount: uint256) -> bool:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_addr` | `address` | Blacklisted address |
| `_amount` | `uint256` | Amount to burn |

#### Returns

| Type | Description |
|------|-------------|
| `bool` | True on success |

#### Access

Governance only (via undyHq)

#### Validation

- Address must be blacklisted
- Address must have sufficient balance

#### Events Emitted

- `Transfer(addr, empty(address), amount)`

## Pause Functions

### `pause`

Toggles pause state for all transfers.

```vyper
@external
def pause(_shouldPause: bool):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_shouldPause` | `bool` | True to pause, False to unpause |

#### Access

Governance only (via undyHq)

#### Events Emitted

- `TokenPauseModified(isPaused)`

## Events

```vyper
event Transfer:
    sender: indexed(address)
    recipient: indexed(address)
    amount: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    amount: uint256

event BlacklistModified:
    addr: indexed(address)
    isBlacklisted: bool

event TokenPauseModified:
    isPaused: bool
```

## Internal Functions

### `_transfer`

Internal transfer with all validations.

```vyper
@internal
def _transfer(_sender: address, _recipient: address, _amount: uint256):
```

#### Validations

1. Token not paused
2. Sender not blacklisted
3. Recipient not blacklisted
4. Sender has sufficient balance

### `_approve`

Internal approval logic.

```vyper
@internal
def _approve(_owner: address, _spender: address, _amount: uint256):
```

### `_spendAllowance`

Internal allowance spending with decrement.

```vyper
@internal
def _spendAllowance(_owner: address, _spender: address, _amount: uint256):
```

Handles max_value(uint256) as infinite allowance (no decrement).

### `_mint`

Internal minting (called by vault on deposits).

```vyper
@internal
def _mint(_recipient: address, _amount: uint256) -> bool:
```

#### Validations

- Recipient not blacklisted

### `_burn`

Internal burning (called by vault on withdrawals).

```vyper
@internal
def _burn(_owner: address, _amount: uint256):
```

### `_domainSeparator`

Computes EIP-712 domain separator.

```vyper
@view
@internal
def _domainSeparator() -> bytes32:
```

Returns cached separator if chain ID unchanged, otherwise recomputes.

### `_validateNewApprovals`

Validates approval parameters.

```vyper
@view
@internal
def _validateNewApprovals(_spender: address, _owner: address):
```

## Security Considerations

### Blacklist Protection
- Blacklisted addresses cannot send or receive tokens
- Governance can burn blacklisted tokens for compliance
- Blacklist status checked on every transfer

### Pause Protection
- All transfers blocked when paused
- Only governance can pause/unpause
- Used for emergency situations

### Signature Replay Protection
- Nonce incremented on every permit
- Deadline enforced for signature expiry
- Domain separator includes chain ID and contract address

### ERC-1271 Support
- Smart contract wallets can sign permits
- Validates via `isValidSignature` callback
- Falls back to ecrecover for EOA signatures

### Infinite Allowance
- max_value(uint256) allowance is not decremented
- Common pattern for DEX/vault integrations
- Reduces gas on repeated operations

## Usage in Vaults

### Minting on Deposit
```python
# In EarnVault.deposit()
shares = calculateShares(assets)
self._mint(receiver, shares)
```

### Burning on Withdrawal
```python
# In EarnVault.redeem()
assets = calculateAssets(shares)
self._burn(owner, shares)
```

### Permit for Gasless Deposits
```python
# User signs permit off-chain
signature = signPermit(vault, amount, deadline)

# Relayer submits permit and deposit
vault.permit(user, vault, amount, deadline, signature)
vault.deposit(amount, user, sender=relayer)
```
