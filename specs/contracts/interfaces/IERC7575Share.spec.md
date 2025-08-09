# Interface Specification: IERC7575Share

## 1. Overview

### Purpose
IERC7575Share defines the share token interface for the ERC7575 Multi-Asset Vault standard. It enables a single share token to be used across multiple vault implementations, each potentially accepting different assets.

### Context in ERC7575 Standard
- ERC7575 extends ERC4626 to support vaults with multiple assets
- The share token (implementing IERC7575Share) is separated from vault logic
- Multiple vaults can mint/burn the same share token
- Enables liquidity provider tokens, multi-collateral systems, and token converters

## 2. Interface Definition

```solidity
interface IERC7575Share is IERC20, IERC165 {
    /**
     * @notice Returns the vault address for a given asset
     * @param asset The address of the asset to look up
     * @return The address of the vault for this asset, or address(0) if not registered
     */
    function vault(address asset) external view returns (address);

    /**
     * @notice Emitted when a vault is registered or updated for an asset
     * @param asset The address of the asset
     * @param vault The address of the vault (address(0) if removing)
     */
    event VaultUpdate(address indexed asset, address vault);
}
```

## 3. Functions

### vault(address asset) → address

**Description**: Returns the vault address associated with a specific asset, enabling discovery of which vault to use for depositing a particular asset.

**Parameters**:
- `asset`: The ERC20 token address to query

**Returns**: 
- Vault address if registered
- `address(0)` if no vault registered for this asset

**Notes**:
- Optional function per ERC7575 standard
- May not be implemented if share token is pre-existing

### supportsInterface(bytes4 interfaceId) → bool

**Description**: Returns true if the contract implements the queried interface (ERC165 standard)

**Parameters**:
- `interfaceId`: The interface identifier to check

**Returns**: Boolean indicating interface support

**Expected Support**:
- `IERC7575Share` interface ID
- `IERC20` interface ID (0x36372b07)
- `IERC165` interface ID (0x01ffc9a7)

## 4. Events

### VaultUpdate(address indexed asset, address vault)

**Description**: Emitted when a vault registration changes for an asset

**When Emitted**:
- New vault registered for an asset
- Existing vault updated for an asset
- Vault removed (vault = address(0))

**Parameters**:
- `asset`: The asset token address (indexed for filtering)
- `vault`: The vault address (or address(0) if removing)

## 5. Relationship with ERC7575 Vaults

The share token works in conjunction with ERC7575 vaults:

1. **Vault Discovery**: The `vault(asset)` function allows users and integrators to find the appropriate vault for a given asset
2. **Multi-Asset Support**: Multiple vaults can exist for different assets, all using the same share token
3. **Minting/Burning**: Vaults are responsible for minting/burning the share token during deposits/withdrawals

## 6. Interface Compliance

Contracts implementing IERC7575Share MUST:
- Implement all functions defined in the interface
- Emit `VaultUpdate` events when vault associations change
- Support ERC165 interface detection
- Be fully ERC20 compliant

## 7. References

- [EIP-7575: Multi-Asset Vaults](https://eips.ethereum.org/EIPS/eip-7575)
- [EIP-4626: Tokenized Vaults](https://eips.ethereum.org/EIPS/eip-4626)
- [EIP-165: Standard Interface Detection](https://eips.ethereum.org/EIPS/eip-165)
- [EIP-20: Token Standard](https://eips.ethereum.org/EIPS/eip-20)