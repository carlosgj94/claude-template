# Vault Decimal Conversion Security Research Report

## Executive Summary

This report analyzes the critical security implications of decimal differences in stablecoin vaults, specifically focusing on the challenges of handling USDC/USDT (6 decimals) alongside DAI/USDS (18 decimals) in the same vault system. Decimal conversion errors represent a major vulnerability class that has led to millions in losses through rounding exploits, inflation attacks, and share price manipulation. This report provides mathematical analysis, real-world examples, and concrete implementation strategies for the ERC7575Vault system.

## Table of Contents

1. [The Decimal Landscape in Stablecoins](#the-decimal-landscape-in-stablecoins)
2. [Mathematical Foundations](#mathematical-foundations)
3. [Vulnerability Classes](#vulnerability-classes)
4. [The Inflation Attack Deep Dive](#the-inflation-attack-deep-dive)
5. [Mixed-Decimal Vault Challenges](#mixed-decimal-vault-challenges)
6. [Precision and Rounding Strategies](#precision-and-rounding-strategies)
7. [Implementation Patterns](#implementation-patterns)
8. [ERC7575Vault Specific Solutions](#erc7575vault-specific-solutions)
9. [Testing Strategies](#testing-strategies)
10. [Best Practices and Recommendations](#best-practices-and-recommendations)

## The Decimal Landscape in Stablecoins

### Current Stablecoin Decimals

| Stablecoin | Decimals | Raw Units for $1 | Market Cap | Note |
|------------|----------|------------------|------------|------|
| USDC | 6 | 1,000,000 | $42B | Circle standard |
| USDT | 6 | 1,000,000 | $134B | Tether standard |
| DAI | 18 | 1,000,000,000,000,000,000 | $5B | ERC20 standard |
| USDS | 18 | 1,000,000,000,000,000,000 | $2B | Sky Protocol |
| FRAX | 18 | 1,000,000,000,000,000,000 | $645M | Frax Protocol |
| TUSD | 18 | 1,000,000,000,000,000,000 | $495M | TrueUSD |

### Why The Difference Exists

**Historical Context**:
- USDT originated on Bitcoin's Omni Layer (6 decimals)
- USDC followed USDT's precedent for market compatibility
- DAI and newer stablecoins follow ERC20's 18-decimal standard
- 18 decimals match ETH's wei denomination

**Technical Implications**:
```solidity
// $100 in different representations
uint256 hundredUSDC = 100_000_000;        // 100 * 10^6
uint256 hundredDAI = 100_000_000_000_000_000_000; // 100 * 10^18

// Decimal difference
uint256 decimalGap = 10^12; // 1,000,000,000,000
```

## Mathematical Foundations

### Core Conversion Formulas

The fundamental challenge in mixed-decimal vaults is maintaining precision across conversions:

```solidity
// Share calculation for different decimals
shares = (assets * totalSupply) / totalAssets

// For 6-decimal asset (USDC) with 18-decimal shares (USDM):
// assets = 1_000_000 (1 USDC)
// If shares should equal ~1e18, need scaling factor
```

### Scaling Mathematics

**Decimal Offset Calculation**:
```solidity
// General formula for decimal conversion
targetValue = sourceValue * 10^(targetDecimals - sourceDecimals)

// USDC (6) to Share (18) conversion
shareDecimals = 18;
assetDecimals = 6;
decimalOffset = shareDecimals - assetDecimals; // 12

// Converting 1 USDC to share-decimal space
1_000_000 * 10^12 = 1_000_000_000_000_000_000 // 1e18
```

### Precision Loss in Division

```solidity
// Dangerous: Division before multiplication loses precision
function convertToSharesBad(uint256 assets) returns (uint256) {
    // For small amounts, this truncates to 0
    return (assets / totalAssets) * totalSupply;
}

// Safe: Multiplication before division preserves precision
function convertToSharesGood(uint256 assets) returns (uint256) {
    return (assets * totalSupply) / totalAssets;
}

// Example with real numbers:
// assets = 100 (0.0001 USDC), totalAssets = 1_000_000, totalSupply = 1_000_000_000_000_000_000
// Bad: (100 / 1_000_000) * 1_000_000_000_000_000_000 = 0 * 1_000_000_000_000_000_000 = 0
// Good: (100 * 1_000_000_000_000_000_000) / 1_000_000 = 100_000_000_000_000
```

## Vulnerability Classes

### 1. Rounding Exploitation

**The Attack Pattern**:
```solidity
contract RoundingExploit {
    // Attacker repeatedly deposits amounts that round favorably
    function attack(Vault vault, uint256 iterations) external {
        for (uint256 i = 0; i < iterations; i++) {
            // Deposit amount that rounds up in shares
            uint256 depositAmount = calculateOptimalDeposit(vault);
            vault.deposit(depositAmount);
            
            // Immediately withdraw, potentially getting more back
            vault.withdraw(vault.balanceOf(address(this)));
        }
    }
    
    function calculateOptimalDeposit(Vault vault) internal view returns (uint256) {
        uint256 totalAssets = vault.totalAssets();
        uint256 totalSupply = vault.totalSupply();
        
        // Find amount where rounding favors attacker
        // This exploits: (assets * totalSupply + totalAssets - 1) / totalAssets
        return (totalAssets - 1) / totalSupply + 1;
    }
}
```

**Real Impact Calculation**:
```solidity
// Vault state: totalAssets = 1_000_001 (USDC), totalSupply = 1_000_000_000_000_000_000
// Attacker deposits: 2 units (0.000002 USDC)

// Shares minted (with rounding up):
shares = (2 * 1_000_000_000_000_000_000 + 1_000_001 - 1) / 1_000_001
       = 2_000_000_000_000_000_000 / 1_000_001
       = 1_999_998_000_003 (rounds to 1_999_999_000_000)

// On withdrawal, gets back:
assets = (1_999_999_000_000 * 1_000_001) / 1_000_000_000_000_000_000
       = 2.000001 (profits 0.000001 USDC per iteration)
```

### 2. Decimal Mismatch Overflow

```solidity
// VULNERABLE CODE
contract DecimalMismatchVault {
    uint8 public constant SHARE_DECIMALS = 18;
    mapping(address => uint8) public assetDecimals;
    
    function deposit(address asset, uint256 amount) external {
        uint8 decimals = assetDecimals[asset];
        
        // DANGER: Can overflow for large amounts with 6-decimal assets
        uint256 normalizedAmount = amount * 10**(SHARE_DECIMALS - decimals);
        
        // If amount = 2^256 / 10^12, this overflows
        _mintShares(msg.sender, normalizedAmount);
    }
}

// ATTACK
// Deposit amount = 115792089237316195423570985008687907853269984665640564039457584007913129639935 / 10^12
// This causes overflow in normalization
```

### 3. Share Price Manipulation via Decimal Gaps

```solidity
contract DecimalGapExploit {
    // Exploiting different decimal precision
    function manipulatePrice(Vault vault) external {
        // For 6-decimal asset vault
        // Deposit dust amount that's significant in 6 decimals but not 18
        uint256 dust = 1; // 0.000001 USDC
        
        // In 18-decimal share space, this might round to 0
        uint256 shares = vault.deposit(dust, address(this));
        
        if (shares == 0) {
            // Free donation to vault, inflates share price
            // Next depositor gets fewer shares
        }
    }
}
```

## The Inflation Attack Deep Dive

### Complete Attack Explanation

The inflation attack is a sophisticated exploit that manipulates the share price of a vault to steal from subsequent depositors. Here's a comprehensive breakdown:

#### The Setup
Imagine a brand new vault that has just been deployed. It's completely empty - no deposits have been made, no shares have been issued. The vault accepts USDC (6 decimals) and issues shares to represent ownership.

#### Step 1: The Attacker's Initial Deposit
The attacker deposits the smallest possible amount: 1 wei of USDC (0.000001 USDC).

In a naive implementation:
```solidity
// Vault state: empty
totalAssets = 0
totalSupply = 0

// Attacker deposits 1 wei
deposit = 1 // 0.000001 USDC

// Share calculation (naive):
if (totalSupply == 0) {
    shares = deposit; // First depositor gets 1:1
}
shares = 1

// New state:
totalAssets = 1
totalSupply = 1
sharePrice = 1 wei per share
```

#### Step 2: The Donation Attack
This is the critical manipulation. The attacker sends 10,000 USDC directly to the vault contract - not through the deposit function, but as a direct transfer. This is called a "donation" because the vault receives the assets but doesn't issue any shares for them.

```solidity
// Attacker transfers 10,000 USDC directly to vault
donation = 10_000_000_000 // 10,000 USDC in wei

// New state:
totalAssets = 10_000_000_001 // 10,000.000001 USDC
totalSupply = 1 // Still only 1 share!
sharePrice = 10_000_000_001 wei per share // Massively inflated!
```

#### Step 3: The Victim Arrives
An innocent user wants to deposit 5,000 USDC into the vault:

```solidity
victimDeposit = 5_000_000_000 // 5,000 USDC

// Share calculation:
shares = (victimDeposit * totalSupply) / totalAssets
shares = (5_000_000_000 * 1) / 10_000_000_001
shares = 0.4999... 
shares = 0 // Rounds down to zero!

// The victim gets 0 shares for their 5,000 USDC!
```

#### Step 4: The Theft
The victim's 5,000 USDC is now in the vault, but they have 0 shares. The attacker still has the only share.

```solidity
// Current state:
totalAssets = 15_000_000_001 // Original + victim's deposit
totalSupply = 1 // Still just the attacker's 1 share

// Attacker withdraws:
withdrawal = (attackerShares * totalAssets) / totalSupply
withdrawal = (1 * 15_000_000_001) / 1
withdrawal = 15_000_000_001 // Gets everything!

// Attacker profit:
profit = withdrawal - (initial_deposit + donation)
profit = 15_000_000_001 - (1 + 10_000_000_000)
profit = 5_000_000_000 // Stole the victim's entire deposit!
```

### Why This Works

The attack exploits several vulnerabilities:

1. **Rounding Down**: Solidity rounds down integer division, so small share amounts become 0
2. **External Donations**: The vault accepts direct transfers that inflate the share price
3. **First Depositor Advantage**: The first depositor can manipulate the initial exchange rate
4. **Decimal Precision**: With only 1 share outstanding, the price per share can be inflated dramatically

### Real-World Impact

This isn't theoretical - variations of this attack have been exploited in production:
- The smaller the decimal count of the asset, the easier the attack
- The attack is most effective on new, empty vaults
- Even with 18-decimal assets, the attack is possible with larger donations

### Mathematical Breakdown with Precise Numbers

Let's walk through with exact calculations:

```solidity
// Initial State (Empty Vault)
totalAssets = 0
totalSupply = 0
assetDecimals = 6 (USDC)

// Step 1: Attacker's First Deposit
attackerDeposit = 1 // 1 unit = 0.000001 USDC
shares = 1 // Gets 1 share (1:1 for first depositor)

// Step 2: Attacker Donates Large Amount
donation = 10_000_000_000 // 10,000 USDC
totalAssets = 10_000_000_001 // 10,000.000001 USDC
totalSupply = 1 // Still just 1 share
// Price = 10,000,000,001 USDC per share!

// Step 3: Victim Deposits
victimDeposit = 20_000_000_000 // 20,000 USDC

// Shares calculation for victim:
victimShares = (20_000_000_000 * 1) / 10_000_000_001
             = 20_000_000_000 / 10_000_000_001
             = 1.999... 
             = 1 // Rounds down to 1 share

// Step 4: Current State
totalAssets = 30_000_000_001 // 30,000.000001 USDC
totalSupply = 2 // 2 shares total

// Step 5: Both Withdraw
attackerAssets = (1 * 30_000_000_001) / 2 = 15_000_000_000 // ~15,000 USDC
victimAssets = (1 * 30_000_000_001) / 2 = 15_000_000_000 // ~15,000 USDC

// Attacker's net:
attackerNet = 15_000_000_000 - 10_000_000_001 = 4_999_999_999 profit
// Victim's net:
victimNet = 15_000_000_000 - 20_000_000_000 = -5_000_000_000 loss
```

### Decimal-Specific Attack Vectors

**6-Decimal Asset Attack**:
```solidity
function attackSixDecimal() external {
    // More effective because smaller decimal space
    // 1 unit = 0.000001 USD (larger economic value)
    
    // Deposit 1 unit
    vault.deposit(1);
    
    // Donate 1_000_000 units ($1)
    token.transfer(address(vault), 1_000_000);
    
    // Share price now inflated by 1_000_000x
}
```

**18-Decimal Asset Attack**:
```solidity
function attackEighteenDecimal() external {
    // Less effective because larger decimal space
    // 1 unit = 0.000000000000000001 USD (negligible value)
    
    // Need much larger donation for same effect
    vault.deposit(1);
    
    // Must donate 1_000_000_000_000_000_000 units ($1)
    token.transfer(address(vault), 1_000_000_000_000_000_000);
}
```

## Mixed-Decimal Vault Challenges

### Challenge 1: Unified Share Representation

When a vault accepts multiple assets with different decimals:

```solidity
contract MultiAssetVault {
    // USDM has 18 decimals
    uint8 constant SHARE_DECIMALS = 18;
    
    // Different assets
    IERC20 public usdc; // 6 decimals
    IERC20 public dai;  // 18 decimals
    
    function depositUSDC(uint256 amount) external {
        // Must scale from 6 to 18 decimals
        uint256 shares = (amount * 10**12 * totalSupply) / totalAssetsUSDC;
        _mint(msg.sender, shares);
    }
    
    function depositDAI(uint256 amount) external {
        // Already 18 decimals, no scaling needed
        uint256 shares = (amount * totalSupply) / totalAssetsDAI;
        _mint(msg.sender, shares);
    }
}
```

### Challenge 2: Cross-Asset Value Comparison

```solidity
contract ValueComparisonChallenge {
    // How to compare $100 USDC vs $100 DAI?
    
    function getTotalValueUSD() public view returns (uint256) {
        uint256 usdcBalance = usdc.balanceOf(address(this)); // 6 decimals
        uint256 daiBalance = dai.balanceOf(address(this));   // 18 decimals
        
        // Must normalize to common decimal base
        uint256 usdcNormalized = usdcBalance * 10**12; // Scale to 18
        uint256 daiNormalized = daiBalance;             // Already 18
        
        // Assuming 1:1 price (simplified)
        return usdcNormalized + daiNormalized;
    }
}
```

### Challenge 3: Rounding Direction Consistency

```solidity
contract RoundingConsistency {
    // Different rounding needed for different operations
    
    function convertToShares(uint256 assets, bool isDeposit) internal view returns (uint256) {
        if (isDeposit) {
            // Round DOWN for deposits (conservative for protocol)
            return (assets * totalSupply) / totalAssets;
        } else {
            // Round UP for withdrawals (conservative for protocol)
            return (assets * totalSupply + totalAssets - 1) / totalAssets;
        }
    }
}
```

## Precision and Rounding Strategies

### The mulDiv Pattern

OpenZeppelin's solution for maintaining precision:

```solidity
library Math {
    function mulDiv(
        uint256 x,
        uint256 y,
        uint256 denominator,
        Rounding rounding
    ) internal pure returns (uint256) {
        uint256 result = (x * y) / denominator;
        
        if (rounding == Rounding.Up && (x * y) % denominator > 0) {
            result += 1;
        }
        
        return result;
    }
}

// Usage in vault
function _convertToShares(uint256 assets, Math.Rounding rounding) internal view returns (uint256) {
    return assets.mulDiv(totalSupply() + 10 ** _decimalsOffset(), totalAssets() + 1, rounding);
}
```

### Decimal Offset Strategy

```solidity
contract DecimalOffsetVault {
    uint8 public immutable assetDecimals;
    uint8 public constant SHARE_DECIMALS = 18;
    uint8 public immutable decimalOffset;
    
    constructor(address _asset) {
        assetDecimals = IERC20Metadata(_asset).decimals();
        decimalOffset = SHARE_DECIMALS - assetDecimals;
    }
    
    function _convertToShares(uint256 assets) internal view returns (uint256) {
        // Scale assets to share decimal space
        uint256 scaledAssets = assets * 10**decimalOffset;
        
        if (totalSupply == 0) {
            return scaledAssets;
        }
        
        return (scaledAssets * totalSupply) / _totalAssets();
    }
    
    function _totalAssets() internal view returns (uint256) {
        // Return assets in share decimal space
        return asset.balanceOf(address(this)) * 10**decimalOffset;
    }
}
```

### Virtual Shares and Assets

```solidity
contract VirtualOffsetVault {
    // Virtual amounts to prevent inflation attack
    uint256 private constant VIRTUAL_SHARES = 10**3;
    uint256 private constant VIRTUAL_ASSETS_MULTIPLIER = 10;
    
    function _convertToShares(uint256 assets) internal view returns (uint256) {
        uint256 virtualAssets = 10**(decimals() - assetDecimals + 3);
        
        return assets.mulDiv(
            totalSupply() + VIRTUAL_SHARES,
            totalAssets() + virtualAssets,
            Math.Rounding.Down
        );
    }
}
```

## Implementation Patterns

### Pattern 1: Normalized Internal Accounting

```solidity
contract NormalizedVault {
    uint256 private constant NORMALIZED_DECIMALS = 18;
    mapping(address => uint8) public assetDecimals;
    mapping(address => uint256) public normalizedBalances;
    
    function deposit(address asset, uint256 amount) external {
        uint8 decimals = assetDecimals[asset];
        uint256 normalized = normalize(amount, decimals);
        
        // All internal math uses normalized (18 decimal) values
        uint256 shares = calculateShares(normalized);
        
        // Transfer actual amount
        IERC20(asset).transferFrom(msg.sender, address(this), amount);
        
        // Store normalized
        normalizedBalances[asset] += normalized;
        _mint(msg.sender, shares);
    }
    
    function normalize(uint256 amount, uint8 decimals) internal pure returns (uint256) {
        if (decimals == NORMALIZED_DECIMALS) return amount;
        if (decimals < NORMALIZED_DECIMALS) {
            return amount * 10**(NORMALIZED_DECIMALS - decimals);
        }
        return amount / 10**(decimals - NORMALIZED_DECIMALS);
    }
}
```

### Pattern 2: Asset-Specific Vaults

```solidity
contract AssetSpecificVault {
    uint8 public immutable assetDecimals;
    uint256 private immutable SCALE_FACTOR;
    
    constructor(address _asset) {
        assetDecimals = IERC20Metadata(_asset).decimals();
        SCALE_FACTOR = 10**(18 - assetDecimals);
    }
    
    function deposit(uint256 assets) external returns (uint256 shares) {
        // Scale to 18 decimals for share calculation
        uint256 scaledAssets = assets * SCALE_FACTOR;
        
        if (totalSupply() == 0) {
            // First deposit: mint virtual shares to prevent inflation
            _mint(address(1), 1000 * SCALE_FACTOR);
            shares = scaledAssets - 1000 * SCALE_FACTOR;
        } else {
            shares = (scaledAssets * totalSupply()) / totalScaledAssets();
        }
        
        asset.transferFrom(msg.sender, address(this), assets);
        _mint(msg.sender, shares);
    }
    
    function totalScaledAssets() public view returns (uint256) {
        return asset.balanceOf(address(this)) * SCALE_FACTOR;
    }
}
```

### Pattern 3: Fixed-Point Math Library

```solidity
library FixedPoint {
    uint256 constant SCALE = 1e18;
    
    struct Unsigned {
        uint256 rawValue;
    }
    
    function fromUnscaledUint(uint256 value) internal pure returns (Unsigned memory) {
        return Unsigned(value * SCALE);
    }
    
    function fromScaledUint(uint256 value, uint8 decimals) internal pure returns (Unsigned memory) {
        if (decimals == 18) return Unsigned(value);
        if (decimals < 18) return Unsigned(value * 10**(18 - decimals));
        return Unsigned(value / 10**(decimals - 18));
    }
    
    function mul(Unsigned memory a, Unsigned memory b) internal pure returns (Unsigned memory) {
        return Unsigned((a.rawValue * b.rawValue) / SCALE);
    }
    
    function div(Unsigned memory a, Unsigned memory b) internal pure returns (Unsigned memory) {
        return Unsigned((a.rawValue * SCALE) / b.rawValue);
    }
}
```

## ERC7575Vault Specific Solutions

### Solution 1: Decimal-Aware Configuration

```solidity
contract ERC7575Vault {
    IUSDM public immutable usdm; // 18 decimals
    IERC20 public immutable asset; // 6 or 18 decimals
    uint8 public immutable assetDecimals;
    uint256 private immutable DECIMAL_FACTOR;
    
    // Minimum amounts to prevent dust attacks
    uint256 public immutable MIN_DEPOSIT;
    uint256 public immutable MIN_SHARES;
    
    constructor(address _asset, address _usdm) {
        asset = IERC20(_asset);
        usdm = IUSDM(_usdm);
        assetDecimals = IERC20Metadata(_asset).decimals();
        
        // Calculate decimal conversion factor
        if (assetDecimals <= 18) {
            DECIMAL_FACTOR = 10**(18 - assetDecimals);
        } else {
            DECIMAL_FACTOR = 1;
        }
        
        // Set minimums based on decimals
        MIN_DEPOSIT = 10**(assetDecimals > 10 ? assetDecimals - 10 : 0);
        MIN_SHARES = 10**10; // Minimum 10^10 shares (0.00000001 USDM)
    }
    
    function deposit(uint256 assets, address receiver) external returns (uint256 shares) {
        require(assets >= MIN_DEPOSIT, "Deposit too small");
        
        // Calculate shares with decimal adjustment
        shares = _calculateShares(assets);
        require(shares >= MIN_SHARES, "Shares too small");
        
        // Transfer assets
        asset.safeTransferFrom(msg.sender, address(this), assets);
        
        // Mint USDM (18 decimals)
        usdm.mint(receiver, shares);
        
        emit Deposit(msg.sender, receiver, assets, shares);
    }
    
    function _calculateShares(uint256 assets) internal view returns (uint256) {
        uint256 totalAssets = _getTotalAssets();
        uint256 totalShares = usdm.totalSupply();
        
        if (totalShares == 0) {
            // First deposit: apply decimal factor and virtual offset
            return assets * DECIMAL_FACTOR + 10**10;
        }
        
        // Scale assets to 18 decimals, then calculate shares
        uint256 scaledAssets = assets * DECIMAL_FACTOR;
        
        // Use mulDiv for precision
        return Math.mulDiv(
            scaledAssets,
            totalShares + 10**10, // Virtual shares offset
            totalAssets * DECIMAL_FACTOR + 1, // Virtual assets offset
            Math.Rounding.Down
        );
    }
}
```

### Solution 2: Oracle Price Normalization

```solidity
contract OracleNormalizedVault {
    // All prices normalized to 18 decimals
    uint256 constant PRICE_DECIMALS = 18;
    
    function getAssetPrice(address asset) public view returns (uint256) {
        uint8 assetDecimals = IERC20Metadata(asset).decimals();
        uint256 rawPrice = oracle.getPrice(asset); // Returns price per token
        
        // Normalize price to 18 decimals
        if (assetDecimals == 6) {
            // USDC: price per 10^6 units
            // Need price per 10^18 units
            return rawPrice * 10**12;
        } else if (assetDecimals == 18) {
            // DAI: already per 10^18 units
            return rawPrice;
        } else {
            // Generic normalization
            return rawPrice * 10**(18 - assetDecimals);
        }
    }
    
    function calculateSharesWithPrice(address asset, uint256 amount) public view returns (uint256) {
        uint256 normalizedPrice = getAssetPrice(asset);
        uint8 assetDecimals = IERC20Metadata(asset).decimals();
        
        // Convert amount to 18-decimal value
        uint256 value = amount * normalizedPrice / 10**assetDecimals;
        
        // Now calculate shares based on normalized value
        return value * totalSupply() / totalValue();
    }
}
```

### Solution 3: Batch Processing with Decimal Safety

```solidity
contract BatchProcessingVault {
    struct RedemptionRequest {
        address owner;
        uint256 shares; // Always 18 decimals (USDM)
        uint256 assets; // In asset decimals (6 or 18)
        bool processed;
    }
    
    function processRedemptions(uint256[] calldata requestIds) external {
        uint256 totalAssetsNeeded = 0;
        uint256 totalSharesToBurn = 0;
        
        // First pass: calculate totals with proper scaling
        for (uint256 i = 0; i < requestIds.length; i++) {
            RedemptionRequest memory req = requests[requestIds[i]];
            
            // Shares are always 18 decimals
            totalSharesToBurn += req.shares;
            
            // Assets in native decimals
            uint256 assets = _convertToAssets(req.shares);
            totalAssetsNeeded += assets;
            
            requests[requestIds[i]].assets = assets;
        }
        
        // Ensure sufficient liquidity
        require(asset.balanceOf(address(this)) >= totalAssetsNeeded, "Insufficient liquidity");
        
        // Second pass: execute redemptions
        for (uint256 i = 0; i < requestIds.length; i++) {
            RedemptionRequest storage req = requests[requestIds[i]];
            
            // Burn USDM shares (18 decimals)
            usdm.burn(address(this), req.shares);
            
            // Transfer assets (6 or 18 decimals)
            asset.safeTransfer(req.owner, req.assets);
            
            req.processed = true;
        }
    }
    
    function _convertToAssets(uint256 shares) internal view returns (uint256) {
        uint256 totalShares = usdm.totalSupply();
        uint256 totalAssets = asset.balanceOf(address(this));
        
        if (assetDecimals < 18) {
            // Shares are 18 decimals, assets are 6 decimals
            // Must scale down the result
            uint256 scaledAssets = shares.mulDiv(
                totalAssets,
                totalShares,
                Math.Rounding.Down
            );
            return scaledAssets; // Already in asset decimals
        } else {
            // Both 18 decimals, no scaling needed
            return shares.mulDiv(
                totalAssets,
                totalShares,
                Math.Rounding.Down
            );
        }
    }
}
```

## Testing Strategies

### Decimal-Specific Test Cases

```solidity
contract DecimalTestSuite {
    ERC7575Vault public usdcVault; // 6 decimals
    ERC7575Vault public daiVault;  // 18 decimals
    
    function testDepositPrecision() public {
        // Test with amounts that stress decimal conversion
        uint256[] memory testAmounts = new uint256[](5);
        testAmounts[0] = 1;           // Minimum unit
        testAmounts[1] = 999;         // Just under 0.001 USDC
        testAmounts[2] = 1000;        // 0.001 USDC
        testAmounts[3] = 999999;      // Just under 1 USDC
        testAmounts[4] = 1000000;     // Exactly 1 USDC
        
        for (uint256 i = 0; i < testAmounts.length; i++) {
            uint256 shares = usdcVault.previewDeposit(testAmounts[i]);
            
            // Verify shares calculation is correct
            uint256 expectedShares = testAmounts[i] * 10**12; // Scale to 18 decimals
            
            // Allow for rounding difference of 1
            assertApproxEqAbs(shares, expectedShares, 1);
        }
    }
    
    function testRoundingConsistency() public {
        // Deposit and immediate withdrawal should not profit
        uint256 initialBalance = usdc.balanceOf(address(this));
        
        uint256 depositAmount = 1000001; // 1.000001 USDC
        uint256 shares = usdcVault.deposit(depositAmount, address(this));
        
        uint256 withdrawnAmount = usdcVault.redeem(shares, address(this), address(this));
        
        // Should get back same or less (never more)
        assertLe(withdrawnAmount, depositAmount);
        
        uint256 finalBalance = usdc.balanceOf(address(this));
        assertLe(finalBalance, initialBalance);
    }
}
```

### Fuzzing for Decimal Edge Cases

```solidity
contract DecimalFuzzTest {
    function testFuzzDecimalConversion(
        uint256 amount,
        uint8 decimals
    ) public {
        decimals = uint8(bound(decimals, 1, 36));
        amount = bound(amount, 1, 10**(decimals > 18 ? 18 : decimals));
        
        // Deploy vault with specific decimal asset
        MockToken token = new MockToken(decimals);
        ERC7575Vault vault = new ERC7575Vault(address(token), address(usdm));
        
        // Test deposit doesn't revert
        token.mint(address(this), amount);
        token.approve(address(vault), amount);
        
        uint256 shares = vault.deposit(amount, address(this));
        
        // Verify shares are reasonable
        if (decimals <= 18) {
            uint256 expectedMin = amount * 10**(18 - decimals) / 2;
            assertGe(shares, expectedMin, "Shares too low");
        }
    }
    
    function testFuzzRoundingAttack(
        uint256 attackAmount,
        uint256 victimAmount
    ) public {
        attackAmount = bound(attackAmount, 1, 1000);
        victimAmount = bound(victimAmount, 1000000, 1000000000);
        
        // Setup attacker donation attack
        usdc.mint(attacker, attackAmount + victimAmount);
        
        vm.startPrank(attacker);
        uint256 attackShares = vault.deposit(attackAmount, attacker);
        
        // Donate to inflate price
        usdc.transfer(address(vault), victimAmount);
        
        vm.stopPrank();
        
        // Victim deposits
        usdc.mint(victim, victimAmount);
        vm.startPrank(victim);
        uint256 victimShares = vault.deposit(victimAmount, victim);
        vm.stopPrank();
        
        // Verify victim didn't lose significant value
        uint256 victimValue = vault.convertToAssets(victimShares);
        uint256 loss = victimAmount > victimValue ? victimAmount - victimValue : 0;
        
        // Loss should be minimal (< 0.01%)
        assertLt(loss * 10000 / victimAmount, 1, "Victim loss too high");
    }
}
```

### Invariant Testing

```solidity
contract DecimalInvariantTest {
    function invariant_totalValueConsistent() public {
        uint256 totalShares = usdm.totalSupply();
        uint256 totalAssets = vault.totalAssets();
        
        if (totalShares > 0 && totalAssets > 0) {
            // Share price should be approximately 1:1 for stablecoins
            uint256 sharePrice = totalAssets * 10**12 / totalShares;
            
            // Allow 1% deviation maximum
            assertGt(sharePrice, 0.99e18);
            assertLt(sharePrice, 1.01e18);
        }
    }
    
    function invariant_noValueCreation() public {
        // Sum of all user shares value <= total assets
        uint256 totalUserValue = 0;
        
        for (uint256 i = 0; i < users.length; i++) {
            uint256 userShares = vault.balanceOf(users[i]);
            uint256 userValue = vault.convertToAssets(userShares);
            totalUserValue += userValue;
        }
        
        assertLe(totalUserValue, vault.totalAssets());
    }
}
```

## Best Practices and Recommendations

### 1. Decimal Handling Checklist

- [ ] **Always query decimals dynamically** - Never hardcode decimal assumptions
- [ ] **Use decimal offset patterns** - Calculate once, use everywhere
- [ ] **Implement minimum amounts** - Prevent dust attacks
- [ ] **Test with all decimal combinations** - 6, 8, 18, and edge cases
- [ ] **Use mulDiv for precision** - Avoid intermediate truncation
- [ ] **Round conservatively** - Down for deposits, up for withdrawals
- [ ] **Normalize internal accounting** - Use consistent decimal base internally

### 2. Implementation Guidelines

```solidity
contract BestPracticeVault {
    // 1. Store decimal information
    uint8 public immutable assetDecimals;
    uint8 public constant SHARE_DECIMALS = 18;
    uint256 private immutable DECIMAL_OFFSET;
    
    // 2. Set meaningful minimums
    uint256 public immutable MIN_DEPOSIT;
    uint256 public immutable DUST_THRESHOLD;
    
    // 3. Virtual amounts for inflation resistance
    uint256 private constant VIRTUAL_SHARES = 10**10;
    uint256 private immutable VIRTUAL_ASSETS;
    
    constructor(address _asset) {
        // Dynamic decimal detection
        assetDecimals = IERC20Metadata(_asset).decimals();
        require(assetDecimals <= 36, "Decimals too high");
        
        // Calculate offsets
        if (assetDecimals < SHARE_DECIMALS) {
            DECIMAL_OFFSET = 10**(SHARE_DECIMALS - assetDecimals);
        } else {
            DECIMAL_OFFSET = 1;
        }
        
        // Set minimums based on decimals
        MIN_DEPOSIT = 10**(assetDecimals / 2); // Reasonable minimum
        DUST_THRESHOLD = 10**(assetDecimals > 6 ? assetDecimals - 6 : 0);
        
        // Virtual assets scaled to asset decimals
        VIRTUAL_ASSETS = 10**(assetDecimals > 3 ? assetDecimals - 3 : 0);
    }
    
    // 4. Safe conversion functions
    function _toShareDecimals(uint256 amount) internal view returns (uint256) {
        if (assetDecimals < SHARE_DECIMALS) {
            return amount * DECIMAL_OFFSET;
        } else if (assetDecimals > SHARE_DECIMALS) {
            return amount / 10**(assetDecimals - SHARE_DECIMALS);
        }
        return amount;
    }
    
    function _toAssetDecimals(uint256 amount) internal view returns (uint256) {
        if (assetDecimals < SHARE_DECIMALS) {
            return amount / DECIMAL_OFFSET;
        } else if (assetDecimals > SHARE_DECIMALS) {
            return amount * 10**(assetDecimals - SHARE_DECIMALS);
        }
        return amount;
    }
    
    // 5. Rounding direction helpers
    function _roundUp(uint256 value, uint256 divisor) internal pure returns (uint256) {
        return (value + divisor - 1) / divisor;
    }
    
    function _roundDown(uint256 value, uint256 divisor) internal pure returns (uint256) {
        return value / divisor;
    }
}
```

### 3. Security Measures

**Prevent Inflation Attacks**:
```solidity
// Dead shares on first deposit
if (totalSupply() == 0) {
    uint256 deadShares = 10**(SHARE_DECIMALS / 2);
    _mint(address(0xdead), deadShares);
    shares = assets * DECIMAL_OFFSET - deadShares;
}
```

**Enforce Minimums**:
```solidity
require(assets >= MIN_DEPOSIT, "Below minimum");
require(shares >= MIN_SHARES, "Shares dust");
```

**Validate Conversions**:
```solidity
// Verify roundtrip consistency
uint256 reconverted = convertToAssets(convertToShares(amount));
require(reconverted <= amount, "Conversion amplification");
```

### 4. Testing Requirements

**Essential Test Coverage**:
1. Deposit/withdraw with 1 unit (minimum)
2. Deposit/withdraw with maximum uint256 (overflow)
3. First deposit scenarios (inflation attack)
4. Rapid small deposits (rounding accumulation)
5. Mixed decimal operations (if multi-asset)
6. Price manipulation attempts
7. Donation attacks
8. Cross-vault decimal interactions

### 5. Audit Focus Areas

**Critical Review Points**:
- Decimal offset calculations
- Rounding direction consistency
- Virtual share/asset implementation
- Minimum threshold enforcement
- Division before multiplication patterns
- Overflow possibilities in scaling
- Share price calculation precision
- Cross-function decimal consistency

## Conclusion

Decimal conversion vulnerabilities represent a critical attack surface in vault implementations, particularly for stablecoin vaults handling mixed 6 and 18 decimal assets. The ERC7575Vault system must implement comprehensive decimal safety measures:

1. **Dynamic decimal detection** - Never assume, always verify
2. **Consistent internal representation** - Normalize to 18 decimals internally
3. **Virtual offsets** - Prevent inflation attacks with virtual shares/assets
4. **Minimum thresholds** - Block dust and rounding exploits
5. **Conservative rounding** - Always favor the protocol
6. **Comprehensive testing** - Cover all decimal combinations and edge cases

The complexity of decimal handling in DeFi cannot be overstated. A single misplaced decimal conversion can lead to catastrophic losses. The mathematical precision required, combined with Solidity's integer-only arithmetic, demands careful implementation and thorough testing.

For the USDM protocol specifically, with USDM at 18 decimals interfacing with 6-decimal USDC/USDT and 18-decimal DAI/USDS, the decimal conversion layer becomes a critical security boundary. Every interaction between different decimal spaces must be carefully validated, tested, and audited.

The cost of implementing these safety measures is minimal compared to the potential losses from decimal-related exploits. As demonstrated by numerous real-world attacks, decimal vulnerabilities are not theoretical—they are actively exploited and have resulted in millions in losses.