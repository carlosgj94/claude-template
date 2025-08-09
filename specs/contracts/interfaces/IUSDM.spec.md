# Interface Specification: IUSDM

## 1. Overview

### Purpose
IUSDM defines the custom interface for USDM-specific functionality that extends beyond standard ERC20 and IERC7575Share capabilities. It provides the vault operation and management functions that allow authorized vaults to mint and burn USDM tokens.

### Role in the System
- Defines vault interaction methods (mint, burn, cross-chain mint)
- Specifies admin functions for vault management
- Provides view functions for vault status queries
- Establishes the permission model for multi-vault operations

### Key Design Principles
- Role-based access control for all privileged operations
- Per-vault pause mechanism for granular control
- Cross-chain native through LayerZero integration
- Clear separation between vault operations and admin functions

## 2. Interface Definition

```solidity
interface IUSDM {
    // Events
    event VaultAdded(address indexed vault);
    event VaultRemoved(address indexed vault);
    event VaultPaused(address indexed vault);
    event VaultUnpaused(address indexed vault);
    event VaultCapUpdated(address indexed vault, uint256 maxBps);

    // Custom Errors
    error VaultNotAuthorized(address vault);
    error VaultIsPaused(address vault);
    error ZeroAddress();
    error ZeroAmount();
    error ContractPaused();
    error UnauthorizedUpgrade();
    error VaultCapExceeded(uint256 requested, uint256 available);
    error InvalidCap(uint256 maxBps);

    // Core Functions - Vault Operations
    function mint(address to, uint256 amount) external;
    function burn(address from, uint256 amount) external;

    // Admin Functions - Vault Management
    function addVault(address vault) external;
    function removeVault(address vault) external;
    function pauseVault(address vault) external;
    function unpauseVault(address vault) external;
    function setVaultCap(address vault, uint256 maxBps) external;
    
    // Admin Functions - Global Controls
    function pause() external;
    function unpause() external;
    
    // View Functions
    function isVault(address account) external view returns (bool);
    function isVaultPaused(address vault) external view returns (bool);
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
}
```

## 3. Events

### VaultAdded(address indexed vault)
**Emitted When**: A new vault is granted minting/burning privileges
**Purpose**: Track vault whitelist changes for monitoring and UI updates
**Indexed**: `vault` - enables efficient filtering by vault address

### VaultRemoved(address indexed vault)
**Emitted When**: A vault's privileges are revoked
**Purpose**: Alert systems that a vault can no longer mint/burn
**Indexed**: `vault` - enables efficient filtering by vault address

### VaultPaused(address indexed vault)
**Emitted When**: A specific vault is temporarily disabled
**Purpose**: Signal temporary suspension without role removal
**Indexed**: `vault` - enables efficient filtering by vault address

### VaultUnpaused(address indexed vault)
**Emitted When**: A paused vault is re-enabled
**Purpose**: Signal vault reactivation
**Indexed**: `vault` - enables efficient filtering by vault address

## 4. Custom Errors

### VaultNotAuthorized(address vault)
**Thrown When**: An address without VAULT_ROLE attempts vault operations
**Parameters**: `vault` - The unauthorized address
### VaultIsPaused(address vault)
**Thrown When**: A paused vault attempts to mint/burn
**Parameters**: `vault` - The paused vault address
### ZeroAddress()
**Thrown When**: address(0) provided where valid address required
**Common Cases**: mint to, burn from, vault registration

### ZeroAmount()
**Thrown When**: Zero amount provided for mint/burn operations
**Purpose**: Prevent wasteful zero-value transactions

### ContractPaused()
**Thrown When**: Operations attempted while contract is globally paused
**Purpose**: Emergency stop mechanism feedback

### UnauthorizedUpgrade()
**Thrown When**: Unauthorized address attempts contract upgrade
**Purpose**: UUPS upgrade protection

## 5. Core Functions - Vault Operations

**Note**: These `mint()` and `burn()` functions are custom IUSDM functions, NOT part of standard ERC20. They allow authorized vaults to create/destroy USDM tokens.

### mint(address to, uint256 amount)
**Purpose**: Create new USDM tokens for vault deposits
**Access**: VAULT_ROLE only, vault not paused, contract not paused
**Parameters**:
- `to`: Recipient of minted tokens
- `amount`: Quantity to mint (18 decimals)
**State Changes**: Increases total supply and recipient balance
**Events**: Transfer(address(0), to, amount)

### burn(address from, uint256 amount)
**Purpose**: Destroy USDM tokens for vault withdrawals
**Access**: VAULT_ROLE only, vault not paused, contract not paused
**Parameters**:
- `from`: Address whose tokens to burn
- `amount`: Quantity to burn (18 decimals)
**State Changes**: Decreases total supply and holder balance
**Events**: Transfer(from, address(0), amount)

## 6. Admin Functions - Vault Management

### addVault(address vault)
**Purpose**: Grant VAULT_ROLE to enable mint/burn operations
**Access**: ADMIN_ROLE only
**Parameters**: `vault` - Address to whitelist
**Validation**: Non-zero address, not already vault
**Events**: VaultAdded(vault), RoleGranted(VAULT_ROLE, vault, msg.sender)

### removeVault(address vault)
**Purpose**: Revoke VAULT_ROLE to disable mint/burn operations
**Access**: ADMIN_ROLE only
**Parameters**: `vault` - Address to remove
**Side Effects**: May clear vault pause status
**Events**: VaultRemoved(vault), RoleRevoked(VAULT_ROLE, vault, msg.sender)

### pauseVault(address vault)
**Purpose**: Temporarily disable specific vault without role removal
**Access**: ADMIN_ROLE only
**Parameters**: `vault` - Vault to pause
**Validation**: Must have VAULT_ROLE, not already paused
**Events**: VaultPaused(vault)

### unpauseVault(address vault)
**Purpose**: Re-enable a paused vault
**Access**: ADMIN_ROLE only
**Parameters**: `vault` - Vault to unpause
**Validation**: Must have VAULT_ROLE, must be paused
**Events**: VaultUnpaused(vault)

## 7. Admin Functions - Global Controls

### pause()
**Purpose**: Emergency stop for all minting/burning operations
**Access**: ADMIN_ROLE only
**Effect**: Prevents all vault operations regardless of individual status
**Events**: Paused(msg.sender)

### unpause()
**Purpose**: Resume normal operations after emergency pause
**Access**: ADMIN_ROLE only
**Validation**: Contract must be paused
**Events**: Unpaused(msg.sender)

## 8. View Functions

### isVault(address account) → bool
**Purpose**: Check if address has VAULT_ROLE
**Returns**: true if address can mint/burn
**Use Case**: Vault validation, UI display

### isVaultPaused(address vault) → bool
**Purpose**: Check individual vault pause status
**Returns**: true if vault is paused
**Use Case**: Pre-transaction validation

### supportsInterface(bytes4 interfaceId) → bool
**Purpose**: ERC165 interface detection
**Returns**: true for supported interfaces
**Supported**: IUSDM, IERC20, IERC7575Share, IERC165

## 9. Access Control Model

```
ADMIN_ROLE:
├── addVault()
├── removeVault()
├── pauseVault()
├── unpauseVault()
├── pause()
├── unpause()
└── _authorizeUpgrade() [internal]

VAULT_ROLE + Not Paused:
├── mint()
└── burn()

Public:
├── isVault() [view]
├── isVaultPaused() [view]
└── supportsInterface() [view]
```

## 10. Integration Guidelines

### For Vault Developers
1. Request VAULT_ROLE from USDM admin
2. Check pause status before operations
3. Handle revert errors gracefully
4. Emit corresponding events in vault contract

### For UI/Frontend
1. Use view functions to check vault status
2. Subscribe to events for real-time updates
3. Validate operations before submission
4. Display pause status to users

### For Monitoring Systems
1. Track VaultAdded/Removed for whitelist changes
2. Monitor mint/burn volumes per vault
3. Alert on pause events
4. Watch for unusual patterns

## 11. Security Considerations

### Permission Model
- Strict role separation between admin and vault operations
- No single vault can compromise entire system
- Admin functions protected by multi-sig

### Pause Mechanism
- Global pause for system-wide emergency
- Per-vault pause for isolated issues
- Pause state doesn't affect ERC20 transfers

### Cross-chain Security
- Relies on LayerZero security model
- Consider implementing rate limits
- Monitor bridge operations closely


## 12. Testing Requirements

### Core Functionality
- [ ] Mint increases supply and balance correctly
- [ ] Burn decreases supply and balance correctly

### Access Control
- [ ] Only VAULT_ROLE can mint/burn
- [ ] Only ADMIN_ROLE can manage vaults
- [ ] Paused vaults cannot operate

### Edge Cases
- [ ] Zero address validation
- [ ] Zero amount validation
- [ ] Pause state combinations
- [ ] Role transition scenarios

## 13. References

- [USDM Contract Specification](../USDM.spec.md)
- [IERC7575Share Specification](./IERC7575Share.spec.md)
- [OpenZeppelin AccessControl](https://docs.openzeppelin.com/contracts/4.x/access-control)
- [LayerZero OFT Standard](https://layerzero.gitbook.io/docs/)