# Contract Specification: USDM

## 1. Overview

### Purpose
USDM main purpose is to be a synthetic asset backed by other stablecoins vaults. To accomplish that, USDM serves as the Share contract in the ERC7575 Multi asset vault standard, in which other vaults (that have been whitelisted) can mint/burn from. This minting and burning basically is done when depositing or redeeming.
To manage these vaults, USDM has a permission management system which is quite critical for the liveness of this contract. The owner should be able to add vaults to the whitelisting and to remove them as well. 


### Key Features

- Serves as the Share contract in the ERC7575 Multi-asset vault standard
- It’s a LayerZero OFT so it’s cross chain compatible from the get go
- It’s an ERC20
- Has a robust permission management system for whitelisting new vaults into the multi asset vault
- Has a function to mint (can be called only from permissioned vaults) and another function to mint to a different chain.
- When minting should allow to mint to another address


## 2. Architecture

### Contract Hierarchy

```
USDM Contract Structure:

Implements Interfaces:
├── IUSDM (custom USDM-specific functions)
├── IERC7575Share (multi-asset vault standard)
├── IERC20 (via OFTUpgradeable → ERC20Upgradeable)
├── IAccessControl (via AccessControlUpgradeable)
└── IERC165 (via ERC165Upgradeable)

Inherits From:
├── OFTUpgradeable (LayerZero cross-chain + ERC20 token functionality)
│   ├── OFTCoreUpgradeable (cross-chain messaging logic)
│   └── ERC20Upgradeable (upgradeable token implementation)
├── AccessControlUpgradeable (role management)
├── PausableUpgradeable (emergency pause)
├── UUPSUpgradeable (upgrade pattern)
└── ERC165Upgradeable (interface detection)
```

### External Dependencies

- **LayerZero OFTUpgradeable**: Upgradeable Omnichain Fungible Token that combines cross-chain functionality with ERC20 token implementation
  - Note: Since LayerZero doesn't provide official OFTUpgradeable, use community implementations (e.g., FraxFinance/LayerZero-v2-upgradeable or dinaricrypto/oft-upgradeable)
  - Includes both OFTCoreUpgradeable (cross-chain logic) and ERC20Upgradeable (token functionality)
- **OpenZeppelin AccessControlUpgradeable**: Role-based permission management for vault whitelisting and admin functions
- **OpenZeppelin PausableUpgradeable**: Emergency pause functionality for the entire contract
- **OpenZeppelin UUPSUpgradeable**: UUPS proxy pattern for upgradeability
- **OpenZeppelin ERC165Upgradeable**: Interface detection support
- **[IERC7575Share][1]**: Multi-asset vault share token interface for ERC7575 standard compliance

### Inheritance Tree

- `OFTUpgradeable` - Upgradeable Omnichain Fungible Token combining:
  - `OFTCoreUpgradeable` - LayerZero cross-chain messaging and transfer logic
  - `ERC20Upgradeable` - Base ERC20 token functionality (upgradeable)
- `AccessControlUpgradeable` - Role-based access control for vault and admin management
- `PausableUpgradeable` - Emergency pause mechanism
- `UUPSUpgradeable` - UUPS proxy upgrade pattern
- `ERC165Upgradeable` - Standard interface detection

## 3. Interface Definition

USDM implements multiple interfaces to provide its full functionality:

### Core Interfaces

- **[IUSDM][2]**: Custom interface defining vault operations, admin functions, and view methods specific to USDM
- **[IERC7575Share][3]**: Multi-asset vault share token standard, enabling USDM to serve as the share token for multiple vaults
- **IERC20**: Standard ERC20 token interface (via ERC20Upgradeable)
- **IAccessControl**: Role-based access control (via AccessControlUpgradeable)
- **IERC165**: Standard interface detection for compatibility checks

### Interface Summary

The IUSDM interface provides:
- **Vault Operations**: `mint()`, `burn()` (custom functions, not part of ERC20)
- **Vault Management**: `addVault()`, `removeVault()`, `pauseVault()`, `unpauseVault()`
- **Global Controls**: `pause()`, `unpause()`
- **View Functions**: `isVault()`, `isVaultPaused()`, `supportsInterface()`

Additional USDM functions (not in IUSDM interface):
- **Cross-chain**: `mintCrossChain()` - Mints and bridges tokens via LayerZero

The IERC7575Share interface adds:
- **Vault Registry**: `vault(address asset)` for discovering vaults by asset
- **Update Events**: `VaultUpdate` event for tracking vault changes

For complete interface specifications, refer to the linked documentation files.

## 4. State Variables

### Storage Layout

```solidity
// USDM-specific storage variables
mapping(address => bool) public vaultPaused;        // Per-vault pause status
mapping(address => uint256) public vaultNetMinted;  // Net minted (minted - burned) per vault
mapping(address => uint256) public vaultMaxBps;     // Max basis points of totalSupply (4500 = 45%)
uint256 public globalTotalSupply;                   // Total supply across ALL chains (unaffected by bridging)

// Storage gap for future upgrades (UUPS pattern)
uint256[46] private __gap;                          // Reserve storage slots for future variables
```

### Constants

```solidity
bytes32 public constant VAULT_ROLE = keccak256("VAULT_ROLE");
bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
```

### Immutables

```solidity
// Note: No immutables in upgradeable contracts
// LayerZero endpoint is stored in OFT contract's storage
```

### Vault Cap System

Per the whitepaper requirements (sections 4.1, 5.2):
- Each vault has a maximum percentage of total USDM supply it can mint
- Default cap: 45% (4500 basis points) per vault
- Caps prevent single-vault concentration risk and limit blast radius
- Over-cap vaults cannot mint new USDM but can still burn
- Preferential redemption from over-cap vaults (future implementation)
- Circuit breaker triggers if multiple vaults exceed caps

**Critical: Cross-Chain Supply Coherence**

The vault cap system uses `globalTotalSupply` instead of `totalSupply()` to maintain consistent caps across bridging:

```solidity
// Cap enforcement formula:
vaultNetMinted[vault] <= globalTotalSupply * vaultMaxBps[vault] / 10000
```

Why `globalTotalSupply` is necessary:
- `totalSupply()` changes when users bridge tokens (burns on source, mints on destination)
- Using `totalSupply()` would cause vault caps to fluctuate with bridging activity
- Example: If 50% of tokens bridge to another chain, vault caps would incorrectly halve
- `globalTotalSupply` tracks total tokens created by vaults across ALL chains
- Remains stable regardless of cross-chain token movement

## 5. Functions Specification

### Constructor

```solidity
constructor() {
    _disableInitializers();
}
```

**Description**: Disables initializers to prevent implementation contract initialization

---

### initialize(string memory name, string memory symbol, address \_lzEndpoint, address admin)

```solidity
function initialize(string memory name, string memory symbol, address _lzEndpoint, address admin) public initializer
```

**Description**: Initializes the USDM token proxy with ERC20 metadata, LayerZero endpoint, and initial admin

**Access Control**: Can only be called once during proxy deployment

**Parameters**:

- `name`: Token name (e.g., "USD Synthetic")
- `symbol`: Token symbol (e.g., "USDM")
- `_lzEndpoint`: LayerZero endpoint address for cross-chain functionality
- `admin`: Initial admin address who will manage vaults

**Validation**:

- `_lzEndpoint != address(0)` - LayerZero endpoint required
- `admin != address(0)` - Admin address required
- Can only be called once (enforced by initializer modifier)

**State Changes**:

- Initializes OFTUpgradeable components:
  - `__ERC20_init(name, symbol)` - Token metadata
  - `__OFT_init(_lzEndpoint)` - LayerZero endpoint configuration
- Initializes AccessControlUpgradeable
- Initializes PausableUpgradeable
- Initializes UUPSUpgradeable
- Sets `globalTotalSupply = 0` - Initial global supply across all chains
- Grants DEFAULT\_ADMIN\_ROLE to admin
- Grants ADMIN\_ROLE to admin

**Note**: The exact initialization pattern depends on the OFTUpgradeable implementation used (FraxFinance or dinaricrypto)

---

### mint(address to, uint256 amount)

**Description**: Mints new USDM tokens to the specified address. Only callable by whitelisted vaults.

**Access Control**: External, requires VAULT\_ROLE and vault not paused

**Parameters**:

- `to`: Address to receive the minted tokens
- `amount`: Amount of USDM to mint

**Preconditions**:

- Contract not paused
- Caller has VAULT\_ROLE
- Caller vault not individually paused
- `to != address(0)`
- `amount > 0`
- `vaultNetMinted[msg.sender] + amount <= (globalTotalSupply + amount) * vaultMaxBps[msg.sender] / 10000` (cap check)

**State Changes**:

1. Increases globalTotalSupply by amount (tracks total across all chains)
2. Increases totalSupply by amount (via \_mint)
3. Increases balance of `to` by amount
4. Updates vaultNetMinted[msg.sender] += amount

**Postconditions**:

- Balance of `to` increased by amount
- Total supply increased by amount

**Events**:

- `Transfer(address(0), to, amount)` (from ERC20)

**Reverts**:

- `VaultNotAuthorized(msg.sender)` - Caller lacks VAULT\_ROLE
- `VaultIsPaused(msg.sender)` - Vault is paused
- `ContractPaused()` - Contract globally paused
- `ZeroAddress()` - to is zero address
- `ZeroAmount()` - amount is zero
- `VaultCapExceeded(uint256 requested, uint256 available)` - Would exceed vault's percentage cap

**CEI Pattern**: ✅ Checks → Effects → No interactions
---

### burn(address from, uint256 amount)

**Description**: Burns USDM tokens from the specified address. Only callable by whitelisted vaults. Users wanting to redeem don’t do it calling it through here but directly interact with the vault they want to redeem from, that vault will call USDM to burn the users tokens.

**Access Control**: External, requires VAULT\_ROLE and vault not paused

**Parameters**:

- `from`: Address to burn tokens from
- `amount`: Amount of USDM to burn

**Preconditions**:

- Contract not paused
- Caller has VAULT\_ROLE
- Caller vault not individually paused
- `from != address(0)`
- `amount > 0`
- `from` has sufficient balance

**State Changes**:

1. Decreases globalTotalSupply by amount (vault burns reduce global supply)
2. Decreases totalSupply by amount (via \_burn)
3. Decreases balance of `from` by amount
4. Updates vaultNetMinted[msg.sender] -= amount

**Postconditions**:

- Balance of `from` decreased by amount
- Total supply decreased by amount

**Events**:

- `Transfer(from, address(0), amount)` (from ERC20)

**Reverts**:

- `VaultNotAuthorized(msg.sender)` - Caller lacks VAULT\_ROLE
- `VaultIsPaused(msg.sender)` - Vault is paused
- `ContractPaused()` - Contract globally paused
- `ZeroAddress()` - from is zero address
- `ZeroAmount()` - amount is zero
- ERC20 insufficient balance error

**CEI Pattern**: ✅ Checks → Effects → No interactions
---

### mintCrossChain(address to, uint256 amount, uint16 dstChainId, bytes calldata adapterParams) payable

**Description**: Mints new USDM tokens and immediately bridges them to another chain via LayerZero. This function combines vault minting with cross-chain transfer in a single operation.

**Access Control**: External, requires VAULT\_ROLE and vault not paused

**Parameters**:

- `to`: Recipient address on destination chain
- `amount`: Amount of USDM to mint and bridge
- `dstChainId`: Destination chain ID in LayerZero network
- `adapterParams`: LayerZero adapter parameters for gas and messaging options

**Preconditions**:

- Contract not paused
- Caller has VAULT\_ROLE
- Caller vault not individually paused
- `to != address(0)`
- `amount > 0`
- Valid destination chain ID with configured trusted remote
- `vaultNetMinted[msg.sender] + amount <= (globalTotalSupply + amount) * vaultMaxBps[msg.sender] / 10000` (cap check)
- Sufficient native token sent for LayerZero fees

**State Changes**:

1. Increases globalTotalSupply by amount (new tokens created)
2. Updates vaultNetMinted[msg.sender] += amount
3. Mints tokens to this contract (increases totalSupply)
4. Initiates cross-chain transfer via OFT's sendFrom
5. Burns tokens on source chain (decreases totalSupply, handled by OFT)
6. Destination chain mints equivalent tokens (increases totalSupply there)

**Postconditions**:

- Vault's net minted amount increased
- Tokens minted and transferred to destination chain
- LayerZero message sent for cross-chain minting

**Events**:

- `Transfer(address(0), address(this), amount)` (initial mint)
- `Transfer(address(this), address(0), amount)` (burn for cross-chain)
- `SendToChain(dstChainId, address(this), to, amount)` (OFT event)

**Reverts**:

- `VaultNotAuthorized(msg.sender)` - Caller lacks VAULT\_ROLE
- `VaultIsPaused(msg.sender)` - Vault is paused
- `ContractPaused()` - Contract globally paused
- `VaultCapExceeded(requested, available)` - Would exceed vault's percentage cap
- `InsufficientFee()` - Insufficient native token for LayerZero fees
- LayerZero specific errors (invalid destination, no trusted path, etc.)

**CEI Pattern**: ✅ Checks → Effects → LayerZero interaction

**Note**: This function enforces vault caps for the initial minting, ensuring vaults cannot exceed their allocated percentage even when minting cross-chain
---

### addVault(address vault)

**Description**: Grants VAULT\_ROLE to a new vault address

**Access Control**: External, requires ADMIN\_ROLE

**Parameters**:

- `vault`: Address of the vault to whitelist

**Preconditions**:

- Caller has ADMIN\_ROLE
- `vault != address(0)`
- Vault doesn't already have VAULT\_ROLE

**State Changes**:

1. Grants VAULT\_ROLE to vault address

**Events**:

- `VaultAdded(vault)`
- `RoleGranted(VAULT_ROLE, vault, msg.sender)` (from AccessControl)

**Reverts**:

- `ZeroAddress()` - vault is zero address
- AccessControl unauthorized error

**CEI Pattern**: ✅ Checks → Effects → No interactions
---

### removeVault(address vault)

**Description**: Revokes VAULT\_ROLE from a vault address

**Access Control**: External, requires ADMIN\_ROLE

**Parameters**:

- `vault`: Address of the vault to remove

**Preconditions**:

- Caller has ADMIN\_ROLE
- Vault has VAULT\_ROLE

**State Changes**:

1. Revokes VAULT\_ROLE from vault address
2. Optionally clear vaultPaused status

**Events**:

- `VaultRemoved(vault)`
- `RoleRevoked(VAULT_ROLE, vault, msg.sender)` (from AccessControl)

**Reverts**:

- AccessControl unauthorized error

**CEI Pattern**: ✅ Checks → Effects → No interactions
---

### pauseVault(address vault)

**Description**: Pauses a specific vault without removing its role

**Access Control**: External, requires ADMIN\_ROLE

**Parameters**:

- `vault`: Address of the vault to pause

**Preconditions**:

- Caller has ADMIN\_ROLE
- Vault has VAULT\_ROLE
- Vault not already paused

**State Changes**:

1. Sets vaultPaused[vault] = true

**Events**:

- `VaultPaused(vault)`

**Reverts**:

- `VaultNotAuthorized(vault)` - Vault doesn't have VAULT\_ROLE
- AccessControl unauthorized error

**CEI Pattern**: ✅ Checks → Effects → No interactions
---

### unpauseVault(address vault)

**Description**: Unpauses a specific vault

**Access Control**: External, requires ADMIN\_ROLE

**Parameters**:

- `vault`: Address of the vault to unpause

**Preconditions**:

- Caller has ADMIN\_ROLE
- Vault has VAULT\_ROLE
- Vault is currently paused

**State Changes**:

1. Sets vaultPaused[vault] = false

**Events**:

- `VaultUnpaused(vault)`

**Reverts**:

- `VaultNotAuthorized(vault)` - Vault doesn't have VAULT\_ROLE
- AccessControl unauthorized error

**CEI Pattern**: ✅ Checks → Effects → No interactions
---

### setVaultCap(address vault, uint256 maxBps)

**Description**: Sets the maximum percentage (in basis points) of total supply a vault can mint

**Access Control**: External, requires ADMIN\\\_ROLE

**Parameters**:

- `vault`: Address of the vault to configure
- `maxBps`: Maximum basis points (e.g., 4500 = 45%)

**Preconditions**:

- Caller has ADMIN\\\_ROLE
- Vault has VAULT\\\_ROLE
- `maxBps <= 10000` (cannot exceed 100%)

**State Changes**:

1. Sets vaultMaxBps[vault] = maxBps

**Events**:

- `VaultCapUpdated(address indexed vault, uint256 maxBps)`

**Reverts**:

- `VaultNotAuthorized(vault)` - Vault doesn't have VAULT\\\_ROLE
- `InvalidCap(uint256 maxBps)` - Cap exceeds 10000 (100%)
- AccessControl unauthorized error

**CEI Pattern**: ✅ Checks → Effects → No interactions
---

### pause()

**Description**: Pauses all mint and burn operations globally

**Access Control**: External, requires ADMIN\_ROLE

**Preconditions**:

- Caller has ADMIN\_ROLE
- Contract not already paused

**State Changes**:

1. Sets global pause state to true

**Events**:

- `Paused(msg.sender)` (from Pausable)

**Reverts**:

- AccessControl unauthorized error
- Pausable already paused error

**CEI Pattern**: ✅ Checks → Effects → No interactions
---

### unpause()

**Description**: Unpauses all operations globally

**Access Control**: External, requires ADMIN\_ROLE

**Preconditions**:

- Caller has ADMIN\_ROLE
- Contract is currently paused

**State Changes**:

1. Sets global pause state to false

**Events**:

- `Unpaused(msg.sender)` (from Pausable)

**Reverts**:

- AccessControl unauthorized error
- Pausable not paused error

**CEI Pattern**: ✅ Checks → Effects → No interactions
---

### supportsInterface(bytes4 interfaceId) → bool

**Description**: Returns true if contract implements the queried interface

**Access Control**: External view

**Parameters**:

- `interfaceId`: Interface identifier to check

**Returns**: Boolean indicating interface support

**Supported Interfaces**:

- [IUSDM][4] - Custom USDM interface
- [IERC7575Share][5] - Multi-asset vault share
- IERC20 - Standard token interface
- IAccessControl - Role-based access
- IERC165 - Interface detection
---

### Cross-Chain Transfer Functions (Inherited from OFTUpgradeable)

#### sendFrom(address from, uint16 dstChainId, bytes32 toAddress, uint256 amount, CallParams callParams)

**Description**: Sends tokens from the current chain to a destination chain via LayerZero

**Access Control**: External, requires token approval or ownership

**Parameters**:
- `from`: Address to debit tokens from
- `dstChainId`: Destination chain ID in LayerZero network
- `toAddress`: Recipient address on destination chain (as bytes32)
- `amount`: Amount of tokens to send
- `callParams`: LayerZero messaging parameters (refund address, zro payment, adapter params)

**State Changes**:
1. Burns tokens on source chain
2. Sends LayerZero message to destination chain
3. Destination chain mints equivalent tokens

**Events**:
- `SendToChain(uint16 indexed dstChainId, address indexed from, bytes32 indexed toAddress, uint256 amount)`
- `Transfer(from, address(0), amount)` (burn event)

**Note**: Cross-chain transfers bypass vault caps as they're moving existing supply, not creating new tokens

---

#### quoteSend(uint16 dstChainId, bytes32 toAddress, uint256 amount, bool useZro, bytes calldata adapterParams) → (uint256 nativeFee, uint256 zroFee)

**Description**: Estimates the fee for a cross-chain transfer

**Access Control**: External view

**Parameters**:
- `dstChainId`: Destination chain ID
- `toAddress`: Recipient address on destination chain
- `amount`: Amount to send
- `useZro`: Whether to pay fees in ZRO token
- `adapterParams`: LayerZero adapter parameters

**Returns**: 
- `nativeFee`: Fee in native token (ETH, BNB, etc.)
- `zroFee`: Fee in ZRO token (if useZro is true)

---

### Cross-Chain Internal Functions (OFT Overrides)

#### \_debitFrom(address from, uint16 dstChainId, bytes32 toAddress, uint256 amount) → uint256

**Description**: Internal function called when sending tokens to another chain. Burns tokens on the source chain without affecting global supply tracking.

**Access Control**: Internal, called by OFT's sendFrom

**Parameters**:
- `from`: Address whose tokens are being sent
- `dstChainId`: Destination chain ID
- `toAddress`: Recipient on destination chain
- `amount`: Amount to send

**State Changes**:
1. Burns tokens via `_burn(from, amount)` - decreases totalSupply
2. **Does NOT** change globalTotalSupply (tokens still exist, just on another chain)

**Critical**: This function must NOT update globalTotalSupply or vaultNetMinted because it's moving existing tokens, not creating/destroying them.

---

#### \_creditTo(uint16 srcChainId, address to, uint256 amount) → uint256

**Description**: Internal function called when receiving tokens from another chain. Mints tokens on the destination chain without affecting global supply tracking.

**Access Control**: Internal, called by OFT's lzReceive

**Parameters**:
- `srcChainId`: Source chain ID
- `to`: Recipient address on this chain
- `amount`: Amount received

**State Changes**:
1. Mints tokens via `_mint(to, amount)` - increases totalSupply
2. **Does NOT** change globalTotalSupply (not new tokens, just arriving from another chain)

**Critical**: This function must NOT update globalTotalSupply or vaultNetMinted because it's receiving existing tokens that were already counted.

---

#### lzReceive(uint16 srcChainId, bytes srcAddress, uint64 nonce, bytes payload)

**Description**: Entry point for LayerZero Endpoint when receiving cross-chain messages. Inherited from OFT, typically not overridden.

**Access Control**: External, only callable by LayerZero Endpoint

**Flow**: 
```
LayerZero Endpoint → lzReceive() → _lzReceive() → _creditTo() → _mint()
```

**Note**: The separation between vault operations (which update globalTotalSupply) and bridge operations (which don't) is crucial for maintaining accurate supply tracking across chains.

---

### \_authorizeUpgrade(address newImplementation)

**Description**: Authorizes an upgrade to a new implementation contract (UUPS pattern)

**Access Control**: Internal, only callable by ADMIN\_ROLE

**Parameters**:

- `newImplementation`: Address of the new implementation contract

**Preconditions**:

- Caller must have ADMIN\_ROLE
- `newImplementation != address(0)`
- New implementation must be a valid contract

**State Changes**:

- None (authorization only, actual upgrade happens in UUPSUpgradeable)

**Reverts**:

- `UnauthorizedUpgrade()` - Caller lacks ADMIN\_ROLE
- `ZeroAddress()` - newImplementation is zero address

**CEI Pattern**: ✅ Checks only
**Security Notes**:

- Critical function that controls contract upgrades
- Should be protected by multi-sig in production
- Consider adding timelock for upgrade delay

## 6. Security Considerations

### Access Control Matrix

| Function             | ADMIN\\\_ROLE | VAULT\\\_ROLE | Public Users | When Paused |
| -------------------- | ------------- | ------------- | ------------ | ----------- |
| mint                 | ❌             | ✅             | ❌            | ❌           |
| burn                 | ❌             | ✅             | ❌            | ❌           |
| mintCrossChain       | ❌             | ✅             | ❌            | ❌           |
| addVault             | ✅             | ❌             | ❌            | ✅           |
| removeVault          | ✅             | ❌             | ❌            | ✅           |
| pauseVault           | ✅             | ❌             | ❌            | ✅           |
| unpauseVault         | ✅             | ❌             | ❌            | ✅           |
| setVaultCap          | ✅             | ❌             | ❌            | ✅           |
| pause                | ✅             | ❌             | ❌            | ✅           |
| unpause              | ✅             | ❌             | ❌            | N/A         |
| \\\_authorizeUpgrade | ✅             | ❌             | ❌            | ✅           |
| transfer             | ✅             | ✅             | ✅            | ✅           |
| approve              | ✅             | ✅             | ✅            | ✅           |
| LayerZero functions  | ✅             | ❌             | ❌            | ✅           |

### Known Risks & Mitigations

**Multiple Vaults with Same Underlying** (Medium):
- Risk: Two vaults could use the same underlying asset (e.g., USDC), potentially leading to double-accounting
- Mitigation: Track net minted per vault; percentage caps apply regardless of underlying asset; consider implementing global limits per underlying asset type

**Vault Cap Breach** (Medium):
- Risk: Rapid totalSupply changes could temporarily allow over-cap minting
- Mitigation: Cap check performed atomically within mint transaction; vaultNetMinted tracks actual exposure

**Centralized Vault Management** (Medium):
- Risk: Admin can add/remove vaults at will
- Mitigation: Use multi-sig for admin role; implement timelock for critical changes in future versions

**Cross-chain Bridge Risk** (High):
- Risk: LayerZero bridge exploits could affect token supply
- Mitigation: Implement mint/burn limits per vault; monitor bridge operations

**Vault Compromise** (High):
- Risk: Compromised vault could mint unlimited USDM
- Mitigation: Per-vault pause mechanism; percentage-based caps (45% default) limit exposure per vault; net minted tracking ensures accurate accounting

**Upgrade Risk** (High):
- Risk: Malicious or buggy upgrade could compromise entire protocol
- Mitigation: Multi-sig required for upgrades; timelock delay recommended; thorough testing on testnet

**Storage Collision** (Medium):
- Risk: Incorrect upgrade could corrupt storage layout
- Mitigation: Storage gaps reserved; careful review of storage layout changes; use upgrade safety tools

**Cross-Chain Supply Coherence** (High):
- Risk: Using `totalSupply()` for vault caps would break with cross-chain transfers
- Mitigation: `globalTotalSupply` tracks total minted across all chains, unaffected by bridging
- Critical: Bridge operations (`_debitFrom`, `_creditTo`) must NEVER modify `globalTotalSupply`
- Monitoring: Ensure `sum(vaultNetMinted) == globalTotalSupply` invariant holds

### Invariants (Must Always Hold)

1. **Total Supply Consistency**: `totalSupply == sum(all user balances)` (per chain)
2. **Vault Authorization**: Only addresses with VAULT\_ROLE can mint/burn
3. **Non-negative Balances**: All user balances >= 0
4. **Pause State Hierarchy**: If contract is paused, no vault can mint/burn regardless of individual vault pause state
5. **Role Consistency**: An address cannot be a vault if it doesn't have VAULT\_ROLE
6. **Global Supply Tracking**: `sum(vaultNetMinted) == globalTotalSupply` (NOT totalSupply)
7. **Cross-Chain Supply Bound**: `totalSupply <= globalTotalSupply` (equality only if no bridging occurred)
8. **Vault Cap Compliance**: For each vault: `vaultNetMinted[vault] <= globalTotalSupply * vaultMaxBps[vault] / 10000`
9. **Cap Sum Constraint**: `sum(vaultMaxBps) <= 30000` (max 300% to allow flexibility)
10. **Bridge Neutrality**: `globalTotalSupply` only changes via vault mint/burn operations, never via cross-chain transfers

### Audit Checklist

- [ ] No use of `delegatecall` to untrusted contracts
- [ ] No use of `tx.origin` for authorization
- [ ] All external calls happen after state changes
- [ ] Critical functions have reentrancy protection
- [ ] Integer operations checked for overflow/underflow
- [ ] No unbounded loops over user-supplied data
- [ ] Events emitted for all state-changing operations
- [ ] Access control on all admin functions
- [ ] Pause mechanism tested and working
- [ ] Storage gaps for upgradeability (if applicable)

## 7. External Integrations

### Expected Callers

- **ERC7575 Vaults**: Primary callers for mint/burn operations
- **Admin Multi-sig**: For vault management and emergency functions
- **LayerZero Relayer**: For cross-chain message delivery

### Integration Requirements

- Vaults must implement ERC7575 standard correctly
- Vaults must handle mint/burn reverts gracefully
- Vaults should check USDM pause state before attempting operations
- Cross-chain integrations must account for LayerZero message fees

### ABI Stability

Critical functions that must not change:

- `mint(address,uint256)`
- `burn(address,uint256)`
- `transfer(address,uint256)`
- `approve(address,uint256)`
- `balanceOf(address)`
- `totalSupply()`

## 8. Notes

### Multiple Vaults with Same Underlying
Having multiple vaults with the same underlying asset is allowed but should be monitored. Each vault may have different strategies or risk profiles, justifying separate implementations. Track total exposure per underlying asset type if risk limits are needed.

### Interface Design (IUSDM)
The IUSDM interface is recommended as it:
- Provides a clean API definition separate from implementation
- Makes integration easier for other contracts
- Enables better testing through mocking
- Allows future alternative implementations

The interface should only include USDM-specific functions (mint, burn, vault management), not inherited ERC20/ERC165 functions which have their own interfaces.

### OFTUpgradeable Implementation Note
Since LayerZero doesn't provide an official OFTUpgradeable, USDM should use a community implementation:
- **FraxFinance/LayerZero-v2-upgradeable**: Production-tested by Frax Protocol
- **dinaricrypto/oft-upgradeable**: OpenZeppelin v5 compatible implementation

The chosen implementation must properly combine OFTCoreUpgradeable with ERC20Upgradeable using initializer patterns instead of constructors.

## 9. LayerZero Configuration

### Configuration Functions
The OFTUpgradeable inheritance provides critical bridge management functions that must be restricted to ADMIN\_ROLE:

#### setTrustedRemote(uint16 chainId, bytes path)
**Purpose**: Establishes trusted communication path with USDM on another chain
**Access**: Override to require ADMIN\_ROLE
**Usage**: Must be called for each chain where USDM is deployed

#### setMinDstGas(uint16 dstChainId, uint16 packetType, uint256 minGas)
**Purpose**: Sets minimum gas requirements for destination chain execution
**Access**: Override to require ADMIN\_ROLE
**Usage**: Prevents out-of-gas failures on destination chains

#### setConfig(uint16 version, uint16 chainId, uint configType, bytes config)
**Purpose**: Configure LayerZero messaging library settings
**Access**: Override to require ADMIN\_ROLE
**Usage**: Set Oracle, Relayer, and other protocol configurations

#### setPrecrime(address precrime)
**Purpose**: Optional pre-crime security module for additional validation
**Access**: Override to require ADMIN\_ROLE
**Usage**: Adds extra security layer for cross-chain transfers

### Deployment Configuration

1. **Initialize on Each Chain**:
   - Deploy USDM with same name/symbol on each chain
   - Configure LayerZero endpoint for that chain
   - Set up access control roles

2. **Establish Trust Relationships**:
   ```solidity
   // On Ethereum (chainId: 101)
   usdm.setTrustedRemote(102, abi.encodePacked(arbitrumUSDM, ethereumUSDM));
   
   // On Arbitrum (chainId: 102)  
   usdm.setTrustedRemote(101, abi.encodePacked(ethereumUSDM, arbitrumUSDM));
   ```

3. **Configure Gas Settings**:
   - Set appropriate minDstGas for each destination
   - Account for different gas costs across chains

4. **Security Configuration**:
   - Configure DVNs (Decentralized Verifier Networks)
   - Set appropriate confirmation requirements
   - Consider implementing rate limits

### Cross-Chain Considerations

- **Supply Coherence**: Total supply across all chains must equal sum of all minted tokens
- **Vault Caps**: Only apply on initial minting chain, not on cross-chain transfers
- **Fee Management**: Users must provide native tokens for LayerZero fees
- **Message Ordering**: LayerZero ensures ordered delivery per source-destination pair

[1]:	./interfaces/IERC7575Share.spec.md
[2]:	./interfaces/IUSDM.spec.md
[3]:	./interfaces/IERC7575Share.spec.md
[4]:	./interfaces/IUSDM.spec.md
[5]:	./interfaces/IERC7575Share.spec.md