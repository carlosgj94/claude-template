# Contract Specification: [ContractName]

## 1. Overview

### Purpose
[One paragraph describing the contract's purpose and role in the system]

### Key Features
- [ ] Feature 1: [Description]
- [ ] Feature 2: [Description]
- [ ] Feature 3: [Description]

## 2. Architecture

### Contract Hierarchy
```
IContractName (interface)
    ↑
ContractNameStorage (storage layout)
    ↑
ContractName (implementation)
```

### External Dependencies
- **[Contract/Interface Name]**: [Purpose]
- **[Contract/Interface Name]**: [Purpose]

### Inheritance Tree
- `[Contract]` - [Purpose]
- `[Contract]` - [Purpose]

## 3. Interface Definition

```solidity
interface IContractName {
    // Events
    event [EventName](...);
    
    // Custom Errors
    error [ErrorName]();
    error [ErrorNameWithParams](uint256 param);
    
    // Core Functions
    function [functionName](...) external returns (...);
    
    // Admin Functions
    function [adminFunction](...) external;
}
```

## 4. State Variables

### Storage Layout
```solidity
// Slot 0
[type] public [varName];                 // X bytes

// Slot 1
[type] public [varName];                 // X bytes

// Slot 2+
mapping([keyType] => [valueType]) public [mappingName];
```

### Constants
```solidity
uint256 public constant [CONSTANT_NAME] = [value];
```

### Immutables
```solidity
[type] public immutable [IMMUTABLE_NAME];
```

## 5. Functions Specification

### Constructor

```solidity
constructor([params])
```

**Description**: [What the constructor does]

**Parameters**:
- `[param]`: [Description and validation requirements]

**Validation**:
- [Validation rule 1]
- [Validation rule 2]

**State Changes**:
- [Initial state setup]

---

### functionName(param1 type, param2 type) → returnType

**Description**: [What this function does]

**Access Control**: [Public/External/Internal/Private/OnlyOwner]

**Parameters**:
- `param1`: [Description]
- `param2`: [Description]

**Preconditions**:
- [Condition 1]
- [Condition 2]

**State Changes**:
1. [State change 1]
2. [State change 2]

**Postconditions**:
- [What must be true after execution]

**Events**:
- `[EventName](params)`

**Reverts**:
- `[ErrorName]()` - [When this happens]

**CEI Pattern**: [✅/❌] [Checks → Effects → Interactions]

**Gas Estimate**: ~[X] gas

**Security Notes**:
- [Any security considerations]

---

### [Additional functions follow the same template above]

## 6. Security Considerations

### Access Control Matrix

**[functionName]**:
- Owner: [✅/❌]
- User: [✅/❌]
- External Contract: [✅/❌]

### Known Risks & Mitigations

**[Risk Name]** ([Severity]):
- Mitigation: [How it's mitigated]

### Invariants (Must Always Hold)

1. **[Invariant Name]**: [Mathematical or logical condition that must always hold]
2. **[Invariant Name]**: [Mathematical or logical condition that must always hold]

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

## 7. Gas Optimization Notes

### Storage Packing
- [Optimization strategy]

### Function Optimizations
- [Optimization strategy]

### Estimated Gas Costs
- **[Operation]**: ~[X] gas

## 8. Deployment & Verification

### Deployment Checklist
- [ ] Correct compiler version
- [ ] Optimization enabled (200 runs)
- [ ] Constructor parameters validated
- [ ] Initial state verified
- [ ] Ownership transferred to multi-sig
- [ ] Contract verified on Etherscan

### Initialization Sequence
1. [Step 1]
2. [Step 2]

## 9. External Integrations

### Expected Callers
- [Caller type/description]

### Integration Requirements
- [Requirement 1]
- [Requirement 2]

### ABI Stability
Critical functions that must not change:
- `[functionSignature]`

## 10. Upgrade Path (if applicable)

### Upgrade Strategy
- [Strategy description]

### Migration Plan
1. [Step 1]
2. [Step 2]

## 11. Monitoring & Maintenance

### Events to Monitor
- [Event or pattern to watch]

### Key Metrics
- [Metric to track]

### Emergency Contacts
- [Role]: [Contact]