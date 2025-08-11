# Contract Specification: ERC7575Vault

## 1. Overview

### Purpose
ERC7575Vault is a multi-instance vault contract that accepts stablecoin deposits (e.g., USDC, USDT, USDS) and manages them as collateral for the USDM synthetic dollar system. Unlike traditional ERC4626 vaults that issue their own shares, ERC7575Vault integrates with USDM as the external share token, following the ERC7575 multi-asset vault standard. The vault implements a hybrid operational model: synchronous deposits with immediate USDM minting and asynchronous redemptions with a request/claim pattern to manage liquidity efficiently.

### Key Features

- **Multi-instance deployment**: One contract code, deployed separately for each collateral asset
- **External share token**: Uses USDM as the share token via mint/burn operations
- **Hybrid sync/async model**: Instant deposits, queued redemptions
- **Price-aware operations**: Integrates Chainlink oracles with TWAP validation (TBD on what TWAP to use and integrate with)
- **Yield generation**: Deploys capital to whitelisted strategies (Morpho, Aave, Sky)
- **Circuit breakers**: Automatic pause on price deviations or oracle failures
- **Cap enforcement**: Respects per-vault limits set in USDM contract

## 2. Architecture

### Contract Hierarchy

```
ERC7575Vault Contract Structure:

Implements Interfaces:
├── IERC7575Vault (custom vault operations)
├── IERC4626 (standard vault interface)
├── IERC7540 (async redemption operations)
├── IAccessControl (role management)
└── IERC165 (interface detection)

Inherits From:
├── ERC4626Upgradeable (base vault functionality)
├── AccessControlUpgradeable (role management)
├── PausableUpgradeable (emergency pause)
├── ReentrancyGuardUpgradeable (reentrancy protection)
├── UUPSUpgradeable (upgrade pattern)
└── ERC165Upgradeable (interface detection)
```

### External Dependencies

- **USDM Contract**: The share token that this vault mints/burns
  - Must have VAULT\_ROLE granted by USDM to call mint/burn
  - Vault operations respect USDM's global pause state
  - Cap enforcement checked via USDM's vaultMaxBps

- **OpenZeppelin Upgradeable Contracts**:
  - ERC4626Upgradeable: Base vault implementation
  - AccessControlUpgradeable: Role-based permissions
  - PausableUpgradeable: Emergency pause functionality
  - ReentrancyGuardUpgradeable: Reentrancy protection
  - UUPSUpgradeable: Upgrade pattern

- **Oracle Dependencies**:
  - Chainlink Price Feed: Primary price source for collateral/USD
  - DEX TWAP Oracle: Secondary validation source (e.g., Curve pool)

- **Strategy Interfaces**:
  - IStrategy: Abstract interface for yield strategies
  - Morpho, Aave, Sky protocol interfaces

### Inheritance Tree

- `ERC4626Upgradeable` - Standard vault operations (modified for external shares)
- `AccessControlUpgradeable` - Role management for keepers and admins
- `PausableUpgradeable` - Emergency pause mechanism
- `ReentrancyGuardUpgradeable` - Protection against reentrancy attacks
- `UUPSUpgradeable` - UUPS proxy upgrade pattern
- `ERC165Upgradeable` - Interface detection support

## 3. Interface Definition

### Core Interfaces

```solidity
interface IERC7575Vault {
    // Synchronous Deposit Operations
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function mint(uint256 shares, address receiver) external returns (uint256 assets);
    
    // Asynchronous Redemption Operations
    function requestRedeem(uint256 shares, address owner) external returns (uint256 requestId);
    function claimRedeem(uint256 requestId) external returns (uint256 assets);
    function pendingRedeemRequest(uint256 requestId) external view returns (uint256 shares, address owner, uint256 claimableAssets);
    
    // Processing Functions
    function processRedemptions(uint256 maxRequests) external;
    function currentEpoch() external view returns (uint256);
    
    // Strategy Management
    function deployToStrategy(address strategy, uint256 amount) external;
    function withdrawFromStrategy(address strategy, uint256 amount) external;
    function harvest() external returns (uint256 yield);
    
    // View Functions
    function totalIdle() external view returns (uint256);
    function totalDeployed() external view returns (uint256);
    function getPricePerShare() external view returns (uint256);
}
```

### Events

```solidity
event RedemptionRequested(address indexed owner, uint256 indexed requestId, uint256 shares, uint256 epoch);
event RedemptionProcessed(uint256 indexed requestId, uint256 assets);
event RedemptionClaimed(address indexed owner, uint256 indexed requestId, uint256 assets);
event StrategyDeployment(address indexed strategy, uint256 amount);
event StrategyWithdrawal(address indexed strategy, uint256 amount);
event HarvestCompleted(uint256 yield, uint256 totalAssets);
event PriceDeviationDetected(uint256 chainlinkPrice, uint256 twapPrice, uint256 deviation);
```

## 4. State Variables

### Storage Layout

```solidity
// Core Vault State
IUSDM public usdm;                          // USDM share token contract
IERC20 public asset;                        // Underlying collateral asset (USDC, USDT, etc.)
uint256 public totalIdle;                   // Assets held in vault, not deployed
uint256 public totalDeployed;               // Assets deployed to strategies

// Oracle Configuration
AggregatorV3Interface public chainlinkFeed; // Chainlink price feed
address public twapOracle;                  // Secondary price source (DEX pool)
uint256 public maxPriceDeviation;           // Max allowed deviation (basis points)
uint256 public priceUpdateThreshold;        // Max staleness for price feeds (seconds)

// Async Redemption State
struct RedemptionRequest {
    address owner;                          // Who requested the redemption
    uint256 shares;                         // Amount of shares to redeem
    uint256 epoch;                          // When request was made
    uint256 claimableAssets;               // Assets available after processing
    bool processed;                         // Whether request has been processed
    bool claimed;                           // Whether assets have been claimed
}

mapping(uint256 => RedemptionRequest) public redemptionRequests;
uint256 public nextRequestId;               // Counter for request IDs
uint256 public currentEpoch;                // Current processing epoch
uint256 public lastProcessedRequestId;      // Last request that was processed

// Queue Management
uint256[] public pendingRequestQueue;       // Queue of unprocessed request IDs
mapping(address => uint256[]) public userRequests; // Requests per user

// Strategy Management
mapping(address => uint256) public strategyAllocations; // Amount per strategy
address[] public activeStrategies;          // List of active strategies
mapping(address => bool) public whitelistedStrategies; // Allowed strategies

// Price Bands (from whitepaper section 4.2)
uint256 public constant PRICE_STABLE = 998;     // 0.998 (scaled by 1000)
uint256 public constant PRICE_DISCOUNT = 995;   // 0.995
uint256 public constant PRICE_SWAP = 985;       // 0.985
uint256 public constant PRICE_PAUSE = 985;      // Circuit breaker threshold

// Access Control Roles
bytes32 public constant KEEPER_ROLE = keccak256("KEEPER_ROLE");
bytes32 public constant STRATEGY_MANAGER_ROLE = keccak256("STRATEGY_MANAGER_ROLE");
bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");

// Storage gap for upgrades
uint256[50] private __gap;
```

### Constants

```solidity
uint256 public constant BASIS_POINTS = 10000;
uint256 public constant DECIMALS_SCALE = 1e18;
uint256 public constant MIN_DEPOSIT = 1e6;      // Minimum deposit (in asset decimals)
uint256 public constant MAX_REDEMPTION_QUEUE = 1000; // Max pending redemptions
uint256 public constant PROCESSING_BATCH_SIZE = 50;  // Redemptions per batch

// Decimal management for cross-decimal operations
uint8 public immutable assetDecimals;           // Detected from asset (6 for USDC/USDT, 18 for DAI)
uint8 public constant SHARE_DECIMALS = 18;      // USDM is always 18 decimals
uint256 public immutable DECIMAL_OFFSET;        // 10^(18 - assetDecimals)

// Virtual amounts for inflation attack prevention
uint256 public constant VIRTUAL_SHARES = 10**10; // Virtual share offset
uint256 public immutable VIRTUAL_ASSETS;        // Calculated based on asset decimals
```

## 5. Functions Specification

### Constructor & Initialization

```solidity
constructor() {
    _disableInitializers();
}
```

**Description**: Disables initializers to prevent implementation contract initialization

---

### initialize(address _asset, address _usdm, address _chainlinkFeed, address _admin)

```solidity
function initialize(
    address _asset,
    address _usdm, 
    address _chainlinkFeed,
    address _admin
) public initializer
```

**Description**: Initializes the vault proxy with asset configuration and oracle setup

**Access Control**: Can only be called once during proxy deployment

**Parameters**:
- `_asset`: The ERC20 collateral asset this vault will accept
- `_usdm`: USDM contract address for minting/burning shares
- `_chainlinkFeed`: Chainlink price feed for asset/USD
- `_admin`: Initial admin address

**State Changes**:
1. Sets asset and USDM contract references
2. Configures oracle feed
3. Initializes access control roles
4. Sets initial parameters (price deviation, thresholds)
5. **Calculates decimal offsets**:
   ```solidity
   assetDecimals = IERC20Metadata(_asset).decimals();
   require(assetDecimals <= 36, "Decimals too high");
   
   // Calculate decimal conversion factor
   if (assetDecimals < SHARE_DECIMALS) {
       DECIMAL_OFFSET = 10**(SHARE_DECIMALS - assetDecimals);
   } else {
       DECIMAL_OFFSET = 1;
   }
   
   // Set virtual assets based on asset decimals
   VIRTUAL_ASSETS = 10**(assetDecimals > 3 ? assetDecimals - 3 : 0);
   ```

**Validation**:
- All addresses must be non-zero
- Asset must be valid ERC20
- USDM must support IERC7575Share interface
- Asset decimals must be <= 36

---

### Synchronous Deposit Functions

#### deposit(uint256 assets, address receiver) → uint256 shares

```solidity
function deposit(uint256 assets, address receiver) 
    public 
    override 
    nonReentrant 
    whenNotPaused 
    returns (uint256 shares)
```

**Description**: Deposits assets and immediately mints USDM to receiver

**Access Control**: Public, anyone can deposit

**Parameters**:
- `assets`: Amount of collateral to deposit
- `receiver`: Address to receive USDM shares

**Preconditions**:
- Contract not paused
- USDM not paused
- `assets >= MIN_DEPOSIT`
- Oracle price within acceptable range
- Vault not exceeding cap in USDM

**State Changes**:
1. Transfer assets from msg.sender to vault
2. Update totalIdle
3. Call USDM.mint(receiver, shares)
4. Update internal accounting

**Price Logic**:
```solidity
uint256 price = getOraclePrice();
if (price >= PRICE_STABLE) {
    shares = assets; // 1:1 conversion
} else if (price >= PRICE_DISCOUNT) {
    shares = assets * price / 1000; // Discounted rate
} else if (price >= PRICE_SWAP) {
    // Swap to healthier asset first (future implementation)
    revert("Swap path not yet implemented");
} else {
    revert("Price below circuit breaker threshold");
}
```

**Events**:
- `Deposit(msg.sender, receiver, assets, shares)`

**Reverts**:
- `InsufficientDeposit()` - Below minimum
- `VaultCapExceeded()` - Would exceed USDM vault cap
- `PriceBelowThreshold()` - Oracle price too low
- `OracleStale()` - Price feed outdated

---

### Asynchronous Redemption Functions

#### requestRedeem(uint256 shares, address owner) → uint256 requestId

```solidity
function requestRedeem(uint256 shares, address owner) 
    external 
    nonReentrant 
    returns (uint256 requestId)
```

**Description**: Queues a redemption request for async processing

**Access Control**: Public, but owner must have approved shares

**Parameters**:
- `shares`: Amount of USDM to redeem
- `owner`: Address that owns the shares

**Preconditions**:
- `shares > 0`
- Owner has sufficient USDM balance
- Queue not at maximum capacity

**State Changes**:
1. Create new RedemptionRequest
2. Add to pendingRequestQueue
3. Transfer USDM from owner to vault (held in escrow)
4. Increment nextRequestId

**Events**:
- `RedemptionRequested(owner, requestId, shares, currentEpoch)`

**Returns**: Unique request ID for tracking

---

#### processRedemptions(uint256 maxRequests)

```solidity
function processRedemptions(uint256 maxRequests) 
    external 
    onlyRole(KEEPER_ROLE)
    nonReentrant
```

**Description**: Processes pending redemption requests in FIFO order

**Access Control**: KEEPER\_ROLE only

**Parameters**:
- `maxRequests`: Maximum number of requests to process

**Logic**:
1. Iterate through pendingRequestQueue
2. For each request:
   3. Calculate assets based on current price
   4. Ensure sufficient liquidity (withdraw from strategies if needed)
   5. Burn USDM shares via USDM.burn()
   6. Mark request as processed with claimableAssets
3. Update epoch if all pending processed

**Price-based Processing**:
```solidity
uint256 price = getOraclePrice();
if (price >= PRICE_STABLE) {
    // 1:1 redemption
    assets = shares;
} else if (price >= PRICE_DISCOUNT) {
    // Single asset redemption at current price
    assets = shares * 1000 / price;
} else if (price >= PRICE_SWAP) {
    // Proportional basket redemption
    // (Implementation depends on multi-vault coordination)
} else {
    // Cannot process - circuit breaker
    return;
}
```

**Events**:
- `RedemptionProcessed(requestId, assets)` for each processed

---

#### claimRedeem(uint256 requestId) → uint256 assets

```solidity
function claimRedeem(uint256 requestId) 
    external 
    nonReentrant 
    returns (uint256 assets)
```

**Description**: Claims processed redemption, transferring assets to owner

**Access Control**: Public, but must be request owner

**Preconditions**:
- Request exists and is processed
- Caller is request owner
- Not already claimed

**State Changes**:
1. Mark request as claimed
2. Transfer claimableAssets to owner
3. Update totalIdle

**Events**:
- `RedemptionClaimed(owner, requestId, assets)`

---

### Strategy Management Functions

#### deployToStrategy(address strategy, uint256 amount)

```solidity
function deployToStrategy(address strategy, uint256 amount) 
    external 
    onlyRole(STRATEGY_MANAGER_ROLE)
```

**Description**: Deploys idle assets to a yield strategy

**Access Control**: STRATEGY\_MANAGER\_ROLE only

**Validation**:
- Strategy is whitelisted
- Sufficient idle balance
- Strategy accepts deposits

**State Changes**:
1. Decrease totalIdle
2. Increase totalDeployed
3. Update strategyAllocations
4. Transfer assets to strategy

---

#### harvest() → uint256 yield

```solidity
function harvest() 
    external 
    onlyRole(KEEPER_ROLE) 
    returns (uint256 yield)
```

**Description**: Harvests yield from all strategies and compounds

**Logic**:
1. Iterate through active strategies
2. Call harvest on each
3. Calculate total yield generated
4. Update totalDeployed with compounded amount

**Events**:
- `HarvestCompleted(yield, totalAssets())`

---

### Oracle Functions

#### getOraclePrice() → uint256

```solidity
function getOraclePrice() public view returns (uint256)
```

**Description**: Gets validated price from oracles

**Logic**:
1. Fetch Chainlink price
2. Fetch TWAP price
3. Validate deviation within threshold
4. Check staleness
5. Return price (scaled to 1000 = $1.00)

**Reverts**:
- `OracleStale()` - Feed older than threshold
- `PriceDeviationTooHigh()` - Chainlink/TWAP mismatch
- `InvalidPrice()` - Price is 0 or clearly wrong

---

### View Functions

#### totalAssets() → uint256

```solidity
function totalAssets() public view override returns (uint256)
```

**Returns**: `totalIdle + totalDeployed`

---

#### getPricePerShare() → uint256

```solidity
function getPricePerShare() public view returns (uint256)
```

**Returns**: Current price per USDM share in asset terms

---

### Admin Functions

#### pause() / unpause()

```solidity
function pause() external onlyRole(ADMIN_ROLE)
function unpause() external onlyRole(ADMIN_ROLE)
```

**Description**: Emergency pause mechanism

---

#### setOracleConfig(address _chainlinkFeed, address _twapOracle, uint256 \_maxDeviation)

```solidity
function setOracleConfig(
    address _chainlinkFeed,
    address _twapOracle,
    uint256 _maxDeviation
) external onlyRole(ADMIN_ROLE)
```

**Description**: Updates oracle configuration

---

### Upgrade Function

#### \_authorizeUpgrade(address newImplementation)

```solidity
function _authorizeUpgrade(address newImplementation) 
    internal 
    override 
    onlyRole(ADMIN_ROLE)
```

**Description**: Authorizes upgrades (UUPS pattern)

---

### Internal Hooks Framework

The vault implements an optional hooks system for extensibility:

```solidity
interface IVaultHooks {
    function beforeDeposit(address user, uint256 assets) external returns (bool);
    function afterDeposit(address user, uint256 shares) external;
    function beforeWithdraw(address user, uint256 shares) external returns (bool);
    function afterWithdraw(address user, uint256 assets) external;
    function beforeRedeem(address user, uint256 shares) external returns (bool);
    function afterRedeem(address user, uint256 assets) external;
}
```

**Usage Pattern**:
```solidity
IVaultHooks public hooks; // Optional hooks contract

function deposit(uint256 assets, address receiver) external returns (uint256 shares) {
    // Pre-deposit hook
    if (address(hooks) != address(0)) {
        require(hooks.beforeDeposit(msg.sender, assets), "Hook rejected deposit");
    }
    
    // Core deposit logic
    shares = _deposit(assets, receiver);
    
    // Post-deposit hook
    if (address(hooks) != address(0)) {
        hooks.afterDeposit(msg.sender, shares);
    }
}
```

**Potential Use Cases**:
- KYC/AML checks
- Custom fee logic
- Loyalty rewards
- Deposit limits per user
- Integration with external protocols

---

### Batch Operations

For gas efficiency and improved UX, the vault supports batch operations:

#### batchDeposit(address[] calldata receivers, uint256[] calldata amounts) → uint256[] shares

```solidity
function batchDeposit(
    address[] calldata receivers,
    uint256[] calldata amounts
) external nonReentrant returns (uint256[] memory shares)
```

**Description**: Processes multiple deposits in a single transaction

**Parameters**:
- `receivers`: Array of addresses to receive shares
- `amounts`: Array of asset amounts to deposit

**Logic**:
1. Validate arrays have same length
2. Calculate total assets needed
3. Transfer total from msg.sender once
4. Process each deposit internally
5. Return array of shares minted

**Gas Optimization**: Single transfer for all deposits

---

#### batchProcessRedemptions(uint256[] calldata requestIds)

```solidity
function batchProcessRedemptions(
    uint256[] calldata requestIds
) external onlyRole(KEEPER_ROLE)
```

**Description**: Process multiple redemption requests efficiently

---

### Emergency Withdrawal Pattern (Under Evaluation)

**Note**: This pattern is under evaluation for potential inclusion. It provides emergency exit capabilities but requires careful consideration of economic implications.

```solidity
bool public emergencyShutdown;

modifier whenNotShutdown() {
    require(!emergencyShutdown, "Vault in emergency shutdown");
    _;
}

function triggerEmergencyShutdown() external onlyRole(ADMIN_ROLE) {
    emergencyShutdown = true;
    emit EmergencyShutdown(block.timestamp);
}

function emergencyWithdraw() external nonReentrant {
    require(emergencyShutdown, "Not in emergency");
    
    uint256 shares = usdm.balanceOf(msg.sender);
    require(shares > 0, "No shares");
    
    // Calculate proportional share of all assets
    uint256 totalShares = usdm.totalSupply();
    uint256 totalAssets = totalAssets();
    uint256 userAssets = (shares * totalAssets) / totalShares;
    
    // Burn user's USDM
    usdm.burn(msg.sender, shares);
    
    // Withdraw proportionally from strategies if needed
    if (userAssets > totalIdle) {
        _emergencyWithdrawFromStrategies(userAssets - totalIdle);
    }
    
    // Transfer assets to user
    asset.safeTransfer(msg.sender, userAssets);
    
    emit EmergencyWithdrawal(msg.sender, shares, userAssets);
}
```

**Considerations**:
- May cause losses if strategies cannot exit immediately
- Could trigger bank run dynamics
- Requires coordination across multiple vaults
- Should only be used in extreme circumstances

## 6. Security Considerations

### Oracle Risks (Critical Focus)

**Price Manipulation**:
- Risk: Attacker manipulates Chainlink or TWAP feeds to mint/redeem at favorable rates
- Mitigation: 
  - Dual oracle system (Chainlink + TWAP) with deviation checks
  - Maximum deviation threshold (default 0.5%)
  - Circuit breaker on extreme deviations

**Oracle Staleness**:
- Risk: Using outdated prices during market volatility
- Mitigation:
  - Heartbeat checks (max 10 minutes staleness)
  - Automatic pause if feeds go stale
  - Manual fallback via admin intervention

**Oracle Failure**:
- Risk: Complete oracle failure prevents operations
- Mitigation:
  - Graceful degradation to pause state
  - Admin can switch to backup oracles
  - No operations proceed without valid prices

**Flash Loan Attacks**:
- Risk: Manipulating TWAP via flash loans
- Mitigation:
  - 30-minute TWAP window resists short-term manipulation
  - Cross-validation with Chainlink
  - Minimum deposit amounts

### Access Control Matrix

| Function           | ADMIN\_ROLE | KEEPER\_ROLE | STRATEGY\_MANAGER | Public | When Paused |
| ------------------ | ----------- | ------------ | ----------------- | ------ | ----------- |
| deposit            | ❌           | ❌            | ❌                 | ✅      | ❌           |
| requestRedeem      | ❌           | ❌            | ❌                 | ✅      | ❌           |
| processRedemptions | ❌           | ✅            | ❌                 | ❌      | ❌           |
| claimRedeem        | ❌           | ❌            | ❌                 | ✅      | ✅           |
| deployToStrategy   | ❌           | ❌            | ✅                 | ❌      | ❌           |
| harvest            | ❌           | ✅            | ❌                 | ❌      | ❌           |
| pause/unpause      | ✅           | ❌            | ❌                 | ❌      | N/A         |
| setOracleConfig    | ✅           | ❌            | ❌                 | ❌      | ✅           |

### Known Risks & Mitigations

**Liquidity Risk**:
- Risk: Insufficient liquidity for redemptions
- Mitigation: 
  - Async redemption queue
  - Can withdraw from strategies during processing
  - Reserve buffer maintenance

**Strategy Risk**:
- Risk: Loss of funds in yield strategies
- Mitigation:
  - Whitelisted strategies only
  - Regular harvesting and monitoring
  - First-loss absorbed by yield buffer

**Reentrancy**:
- Risk: Reentrancy in deposit/redeem flows
- Mitigation: ReentrancyGuard on all state-changing functions

**Vault Cap Enforcement**:
- Risk: Exceeding USDM's per-vault caps
- Mitigation: Check with USDM before minting

**Cross-Vault Coordination**:
- Risk: Proportional redemptions need multi-vault coordination
- Mitigation: Keeper-coordinated processing across vaults

## 7. Mathematical Formulas

### Share Price Calculation

**With Virtual Offsets and Decimal Adjustment**:
```solidity
sharePrice = (totalAssets * DECIMAL_OFFSET + VIRTUAL_ASSETS) * 10^18 
           / (totalSupply + VIRTUAL_SHARES)
```

### Deposit Share Calculation

```solidity
// For 6-decimal assets (USDC/USDT)
shares = (assets * DECIMAL_OFFSET) * (totalSupply + VIRTUAL_SHARES) 
       / (totalAssets * DECIMAL_OFFSET + VIRTUAL_ASSETS)

// For 18-decimal assets (DAI/USDS)
shares = assets * (totalSupply + VIRTUAL_SHARES) 
       / (totalAssets + VIRTUAL_ASSETS)
```

### Redemption Asset Calculation

```solidity
// Adjusting for decimals
assets = shares * (totalAssets * DECIMAL_OFFSET + VIRTUAL_ASSETS) 
       / ((totalSupply + VIRTUAL_SHARES) * DECIMAL_OFFSET)
```

### Yield Calculation (APY)

```solidity
// Calculate yield generated over period
yieldGenerated = currentTotalAssets - previousTotalAssets - netDeposits

// APY calculation (annualized)
timeElapsed = block.timestamp - lastHarvestTime
APY = (yieldGenerated * SECONDS_PER_YEAR * 10000) 
    / (previousTotalAssets * timeElapsed)  // In basis points

// Performance fee on yield
performanceFee = yieldGenerated * performanceFeeRate / 10000
netYield = yieldGenerated - performanceFee
```

### Slippage Calculation for Redemptions

```solidity
// Price-based slippage
oraclePrice = getOraclePrice() // Scaled to 1000 = $1.00
slippage = (1000 - oraclePrice) * 100 / 1000 // Percentage

// Redemption with slippage
assetsRedeemed = shares * (1000 - slippage) / 1000
```

## 8. Invariants

These conditions must ALWAYS hold:

1. **Asset Conservation**: `totalAssets() == totalIdle + totalDeployed`
2. **Share Backing**: All minted USDM is backed by deposited assets
3. **Request Integrity**: Processed requests have claimableAssets set
4. **Queue Order**: Redemptions processed in FIFO order
5. **Price Bounds**: Operations only proceed within defined price bands
6. **Oracle Validation**: No operations with stale or deviant prices
7. **Cap Compliance**: Vault respects USDM's vaultMaxBps limit
8. **Strategy Consistency**: `sum(strategyAllocations) == totalDeployed`
9. **No Negative Balances**: All balances >= 0
10. **Request State Machine**: Requests go pending → processed → claimed
11. **Decimal Consistency**: `shares * 10^assetDecimals == assets * 10^18` (approximately)
12. **Virtual Offset Integrity**: Virtual shares and assets never decrease

## 9. External Integrations

### USDM Integration
- Must be granted VAULT\_ROLE by USDM admin
- Calls USDM.mint() for deposits
- Calls USDM.burn() for redemptions
- Respects USDM pause state and caps

### Oracle Integration
- Chainlink: AggregatorV3Interface for price feeds
- DEX: Pool interface for TWAP calculation
- Must handle feed deprecation gracefully

### Strategy Integration
- Implements IStrategy interface
- Morpho, Aave, Sky protocols
- Must handle partial withdrawals
- Yield reporting and compounding

### Expected Callers
- End users: Deposit and request redemptions
- Keepers: Process redemptions and harvest
- Strategy managers: Rebalance deployments
- Admin: Emergency functions and upgrades

## 10. Events & Monitoring

Critical events to monitor:
- Large deposits/redemptions
- Oracle price deviations
- Strategy losses
- Queue buildup
- Processing delays
- Circuit breaker triggers

## 11. Notes

### Asynchronous Design Rationale
The async redemption model prevents liquidity crunches and allows orderly unwinding of strategy positions. This is especially important during market stress when many users might redeem simultaneously.

### Oracle Security Emphasis
Given the critical nature of price feeds in determining mint/redeem rates, oracle security is paramount. The dual-oracle system with deviation checks provides defense-in-depth against manipulation.

### Multi-Instance Considerations
Each deployed instance (USDC vault, USDT vault, etc.) operates independently but must coordinate during system-wide events like proportional redemptions or rebalancing.

### Future Enhancements
- Cross-chain deposit/redeem support (see below)
- Automated strategy rebalancing
- Dynamic fee mechanisms
- Advanced queue prioritization

### Cross-Chain Deposits (Future Research)

**LayerZero Hub-and-Spoke Architecture**:

This approach would enable deposits from any L2 while maintaining accounting on mainnet:

```solidity
// Main Vault on Ethereum (Hub)
contract MainVault is ERC7575Vault {
    mapping(uint16 => address) public remoteVaults; // chainId => vault address
    ILayerZeroEndpoint public lzEndpoint;
    
    function receiveDeposit(
        uint16 srcChainId,
        bytes memory srcAddress,
        uint64 nonce,
        bytes memory payload
    ) external {
        require(msg.sender == address(lzEndpoint), "Invalid caller");
        require(remoteVaults[srcChainId] == address(srcAddress), "Unknown source");
        
        (address user, uint256 amount, address asset) = abi.decode(
            payload, 
            (address, uint256, address)
        );
        
        // Mint shares for cross-chain depositor
        uint256 shares = _calculateShares(amount);
        usdm.mint(user, shares);
        
        emit CrossChainDeposit(srcChainId, user, amount, shares);
    }
}

// Satellite Vault on L2s (Spokes)
contract L2Vault {
    ILayerZeroEndpoint public lzEndpoint;
    uint16 public mainChainId;
    address public mainVault;
    
    function deposit(uint256 amount) external payable {
        // Collect assets on L2
        asset.transferFrom(msg.sender, address(this), amount);
        
        // Bridge assets to mainnet (using native bridge or LayerZero OFT)
        _bridgeAssets(amount);
        
        // Send message to mainnet vault
        bytes memory payload = abi.encode(msg.sender, amount, address(asset));
        
        lzEndpoint.send{value: msg.value}(
            mainChainId,                    // destination chainId
            abi.encodePacked(mainVault),    // destination address
            payload,                         // deposit info
            payable(msg.sender),             // refund address
            address(0),                      // zroPaymentAddress
            bytes("")                        // adapterParams
        );
    }
}
```

**Benefits**:
- Users can deposit from any supported L2
- All accounting remains on mainnet
- Single source of truth for USDM supply
- Reduces mainnet gas costs for users

**Challenges**:
- Cross-chain message reliability
- Bridge security risks
- Latency in deposit confirmation
- Handling failed messages

**Alternative Approaches**:
- **Wormhole**: Similar architecture with different messaging protocol
- **Hyperlane**: Focus on modular interchain communication
- **Native Bridges**: Use official L2 bridges but more complex integration

This would be explored in detail for V2 implementation after core vault functionality is proven.