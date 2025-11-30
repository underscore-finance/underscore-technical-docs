# SigHelper Technical Documentation

[View Source Code](https://github.com/underscore-finance/underscore-protocol/blob/master/contracts/modules/SigHelper.vy)

## Overview

SigHelper is an internal utility module that provides EIP-712 signature helper functions for agent contracts in the Underscore Protocol. It generates domain separators, full message digests, and handles nonce/expiration management for signature-based authentication.

**Core Features**:
- **EIP-712 Domain Separator**: Generate domain separators for agent contracts
- **Full Digest Generation**: Create complete EIP-712 digests ready for signing
- **Nonce Management**: Auto-fetch current nonce from agent contracts
- **Expiration Handling**: Default 1-hour expiration when not specified

## Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `SIG_PREFIX` | `0x1901...` | EIP-712 signature prefix bytes |

## Interface Dependencies

### AgentSenderGeneric

```vyper
interface AgentSenderGeneric:
    def currentNonce(_userWallet: address) -> uint256: view
```

Used to fetch the current nonce for a user wallet from an agent contract.

## Internal Functions

### `_domainSeparator`

Generates the EIP-712 domain separator for an agent contract.

```vyper
@view
@internal
def _domainSeparator(_agentSender: address) -> bytes32:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentSender` | `address` | Agent contract address |

#### Returns

| Type | Description |
|------|-------------|
| `bytes32` | EIP-712 domain separator hash |

#### Domain Structure

The domain separator is computed using:
- **Type Hash**: `keccak256('EIP712Domain(string name,uint256 chainId,address verifyingContract)')`
- **Name**: `'UnderscoreAgent'`
- **Chain ID**: Current chain ID
- **Verifying Contract**: Agent sender address

#### Example Domain Separator

```
domainSeparator = keccak256(abi_encode(
    TYPE_HASH,
    keccak256('UnderscoreAgent'),
    chain.id,
    agentSenderAddress
))
```

### `_getFullDigest`

Generates the full EIP-712 digest that needs to be signed.

```vyper
@view
@internal
def _getFullDigest(_agentSender: address, _messageHash: bytes32) -> bytes32:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentSender` | `address` | Agent contract address |
| `_messageHash` | `bytes32` | Message hash (action parameters) |

#### Returns

| Type | Description |
|------|-------------|
| `bytes32` | Full EIP-712 digest to sign |

#### Digest Structure

```
fullDigest = keccak256(concat(
    0x1901,                    // EIP-712 prefix
    domainSeparator,           // Domain-specific hash
    messageHash                // Action-specific hash
))
```

### `_getNonce`

Fetches the current nonce for a user wallet from an agent contract.

```vyper
@view
@internal
def _getNonce(_agentSender: address, _userWallet: address) -> uint256:
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentSender` | `address` | Agent contract to query |
| `_userWallet` | `address` | User wallet address |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Current nonce for the wallet |

### `_getNonceAndExpiration`

Gets or calculates nonce and expiration values.

```vyper
@view
@internal
def _getNonceAndExpiration(_agentSender: address, _userWallet: address, _nonce: uint256, _expiration: uint256) -> (uint256, uint256):
```

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| `_agentSender` | `address` | Agent contract address |
| `_userWallet` | `address` | User wallet address |
| `_nonce` | `uint256` | Nonce (0 = auto-fetch current) |
| `_expiration` | `uint256` | Expiration (0 = auto-calculate) |

#### Returns

| Type | Description |
|------|-------------|
| `uint256` | Resolved nonce value |
| `uint256` | Resolved expiration timestamp |

#### Behavior

1. If `_nonce == 0`: Fetches current nonce from agent contract
2. If `_expiration == 0`: Sets to `block.timestamp + 3600` (1 hour)
3. Otherwise: Uses provided values

#### Default Expiration

When expiration is set to 0, the function automatically sets it to 1 hour (3600 seconds) from the current block timestamp. This provides a reasonable default for most use cases while preventing indefinitely valid signatures.

## Usage in Signature Helpers

SigHelper is imported and used by signature helper contracts:

### EarnVaultAgentSigHelper Usage

```vyper
import contracts.modules.SigHelper as sigHelper

@view
@external
def getDepositForYieldHash(...) -> (bytes32, uint256, uint256):
    nonce, expiration = sigHelper._getNonceAndExpiration(
        _agentWrapper,
        _userWallet,
        _nonce,
        _expiration
    )
    return (
        sigHelper._getFullDigest(
            _agentWrapper,
            keccak256(abi_encode(action_code, params..., nonce, expiration))
        ),
        nonce,
        expiration
    )
```

### UserWalletSignatureHelper Usage

```vyper
import contracts.modules.SigHelper as sigHelper

@view
@external
def getTransferFundsHash(...) -> (bytes32, uint256, uint256):
    nonce, expiration = sigHelper._getNonceAndExpiration(
        _agentSender,
        _userWallet,
        _nonce,
        _expiration
    )
    messageHash = keccak256(abi_encode(
        convert(1, uint8),  # Action code
        _userWallet,
        _recipient,
        _asset,
        _amount,
        nonce,
        expiration
    ))
    return (
        sigHelper._getFullDigest(_agentSender, messageHash),
        nonce,
        expiration
    )
```

## EIP-712 Compliance

The module implements [EIP-712](https://eips.ethereum.org/EIPS/eip-712) structured data hashing and signing:

### Domain Separator Components

| Field | Type | Value |
|-------|------|-------|
| name | string | "UnderscoreAgent" |
| chainId | uint256 | Current chain ID |
| verifyingContract | address | Agent contract address |

### Signature Structure

```
signature = sign(
    keccak256(
        "\x19\x01" ||
        domainSeparator ||
        messageHash
    )
)
```

## Security Considerations

### Chain ID Binding
- Domain separator includes chain ID
- Prevents replay attacks across different chains

### Contract Binding
- Domain separator includes verifying contract address
- Signatures are only valid for specific agent contract

### Nonce Protection
- Each signature includes incrementing nonce
- Prevents signature replay within same contract

### Expiration Protection
- Default 1-hour expiration
- Signatures become invalid after expiration
- Prevents indefinite signature validity

### Auto-Fetch Behavior
- Nonce of 0 triggers auto-fetch from contract
- Ensures users get correct current nonce
- Prevents using stale nonces

## Common Integration Patterns

### Generating a Signable Hash
```python
# In a signature helper contract
def get_hash_for_action(agent, wallet, params):
    # Get nonce and expiration
    nonce, expiration = sigHelper._getNonceAndExpiration(
        agent, wallet, 0, 0  # Auto-fetch both
    )

    # Create message hash
    message_hash = keccak256(abi_encode(action_code, params, nonce, expiration))

    # Get full digest
    return sigHelper._getFullDigest(agent, message_hash), nonce, expiration
```

### Client-Side Signing Flow
```python
# 1. Call signature helper to get hash
hash, nonce, expiration = sig_helper.getActionHash(agent, wallet, params)

# 2. Sign hash with user's private key (EIP-712)
signature = personal_sign(hash, private_key)

# 3. Call agent with signature
agent.performAction(
    wallet,
    params,
    Signature(signature, nonce, expiration),
    sender=relayer
)
```

## Testing

For test examples, see: [`tests/modules/`](../../../tests/modules/)
