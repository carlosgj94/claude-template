# CLAUDE.md - Smart Contract Development

## Core Commands

```bash
# Build & Test (run after EVERY change)
forge build
forge test -vvv

# Extended validation
forge coverage
forge test --gas-report
slither . --exclude external-function
```

## Workflow (USE TodoWrite TOOL)

### 1. Plan & Spec FIRST (STOP for approval)

Create detailed spec in `specs/contracts/{ContractName}.spec.md`:

```markdown
# Contract: {ContractName}

## Purpose
Single sentence describing the contract's core function

## State Variables
```solidity
address public owner;           // Contract owner
uint256 public totalSupply;     // Total token supply
mapping(address => uint256) public balances;  // User balances
```

## Functions

### functionName(param1 type, param2 type) → returnType
- **Visibility**: public/external/internal/private
- **Modifiers**: onlyOwner, nonReentrant
- **Parameters**:
  - `param1`: Description and constraints
  - `param2`: Description and constraints
- **Returns**: What it returns and when
- **Reverts**:
  - `ErrorName()`: When this happens
  - `OtherError(value)`: When that happens
- **Events**: 
  - `EventName(indexed address user, uint256 value)`
- **CEI Pattern**: Check → Effect → Interaction order
- **Gas**: ~50,000 (approximate)

## Invariants (MUST HOLD)
1. `sum(all balances) == totalSupply`
2. `balance[user] <= totalSupply`
3. `owner != address(0)`

## Security Considerations
- Reentrancy: Protected via checks-effects-interactions
- Access Control: Owner-only functions use modifier
- Integer Overflow: Using Solidity 0.8+ built-in checks
```

Then use TodoWrite to create task list and WAIT FOR APPROVAL.

### 2. Implement (WITH TESTING)

```bash
# For EACH todo item:
1. Mark as in_progress in TodoWrite
2. Write/modify code
3. IMMEDIATELY run:
   forge build
   forge test -vvv
   forge test --match-test testSpecificFunction -vvv  # For specific test
4. If tests pass, mark as completed
5. If tests fail, fix before moving on
```

**Test commands by context:**
```bash
# After contract changes
forge build && forge test

# After specific function
forge test --match-contract ContractNameTest
forge test --match-test testFunctionName

# After security-critical changes
forge test && slither .

# Check gas optimization
forge snapshot
forge test --gas-report
```

### 3. Test Failure Protocol

When a test fails, determine the cause:

**A. Test Issue (fix immediately):**
- Wrong expected values in assertions
- Incorrect test setup
- Missing test state initialization
- Wrong function parameters in test

**B. Contract Logic Issue (STOP & ASK):**
- Function doesn't match spec behavior
- Unexpected reverts in valid scenarios
- State not updating as designed
- Security vulnerability detected

**Decision Tree:**
```
Test Fails
├── Is the CONTRACT behavior correct according to spec?
│   ├── YES → Fix the TEST
│   ├── NO → STOP: "Test failing due to contract logic issue: [describe]. Should I fix the contract or is this intended behavior?"
│   └── UNSURE → STOP: "Test failing. Unclear if contract logic or test issue: [describe]. Please confirm expected behavior."
```

**Example escalations:**
- "The withdraw function reverts on valid amounts. This seems like a logic error. Should the validation be `>=` instead of `>`?"
- "Transfer test expects event but none emitted. Is the Transfer event missing from the contract implementation?"
- "Invariant violation detected: total supply doesn't match sum of balances after mint. Please confirm the minting logic."

## Code Rules

```solidity
// Custom errors with context
error TransferToZeroAddress();
error InsufficientBalance(uint256 available, uint256 required);
error Unauthorized(address caller);

// CEI Pattern (Checks-Effects-Interactions)
function withdraw(uint256 amount) external {
    // 1. Checks
    if (amount == 0) revert ZeroAmount();
    if (balances[msg.sender] < amount) 
        revert InsufficientBalance(balances[msg.sender], amount);
    
    // 2. Effects (state changes)
    balances[msg.sender] -= amount;
    
    // 3. Interactions (external calls)
    (bool success,) = msg.sender.call{value: amount}("");
    if (!success) revert TransferFailed();
}

// Always use SafeERC20
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
using SafeERC20 for IERC20;
```

## Security Checklist

Every function needs:
- [ ] Input validation (zero checks, bounds)
- [ ] Access control (who can call?)
- [ ] State changes BEFORE external calls
- [ ] Events for ALL state changes
- [ ] Reentrancy protection if needed
- [ ] No unbounded loops

## Spec Sync Rules

**CRITICAL**: Spec must ALWAYS match implementation
- Before coding: Write spec
- After coding: Verify spec matches
- If mismatch: STOP and ask which is correct

## File Structure

```
contracts/     # Solidity files
test/         # Foundry tests (mirror contracts structure)
script/       # Deploy scripts
specs/        # Contract specifications (REQUIRED)
```

## Emergency Protocol

If critical vulnerability found:
1. STOP immediately
2. Document in comment
3. DO NOT attempt fix
4. Wait for approval