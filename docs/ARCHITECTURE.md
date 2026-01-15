# DOOR Protocol - Architecture Documentation

## Table of Contents

1. [System Overview](#system-overview)
2. [Core Components](#core-components)
3. [Contract Interactions](#contract-interactions)
4. [Waterfall Distribution Mechanism](#waterfall-distribution-mechanism)
5. [Safety Module Design](#safety-module-design)
6. [Oracle System](#oracle-system)
7. [Epoch Management](#epoch-management)
8. [Strategy Architecture](#strategy-architecture)
9. [Access Control](#access-control)
10. [Emergency Procedures](#emergency-procedures)

---

## System Overview

DOOR Protocol implements a structured DeFi product with **waterfall distribution** mechanics. The protocol separates risk and return profiles through a dual-tranche system, where Senior vault holders receive fixed-rate returns with priority claims, and Junior vault holders receive leveraged returns with first-loss responsibility.

### Design Principles

1. **Modularity**: Each component is independently upgradeable and replaceable
2. **Safety First**: Multiple layers of protection for user funds
3. **Transparency**: All operations are on-chain and verifiable
4. **Efficiency**: Gas-optimized using ERC-4626 standard
5. **Composability**: Standard interfaces for easy integration

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        User Layer                                 │
│  ┌────────────────┐              ┌────────────────┐             │
│  │ Senior Investors│              │ Junior Investors│             │
│  └───────┬────────┘              └────────┬───────┘             │
│          │                                 │                      │
└──────────┼─────────────────────────────────┼──────────────────────┘
           │                                 │
┌──────────┼─────────────────────────────────┼──────────────────────┐
│          │       Vault Layer               │                      │
│  ┌───────▼────────┐              ┌─────────▼──────┐             │
│  │ SeniorVault    │◄──────────── │ JuniorVault    │             │
│  │ (ERC-4626)     │   Communicates│ (ERC-4626)     │             │
│  └───────┬────────┘              └─────────┬──────┘             │
│          │                                 │                      │
└──────────┼─────────────────────────────────┼──────────────────────┘
           │                                 │
┌──────────┼─────────────────────────────────┼──────────────────────┐
│          │      Coordination Layer         │                      │
│          │                                 │                      │
│  ┌───────▼─────────────────────────────────▼──────┐             │
│  │                CoreVault                        │             │
│  │         (Central Coordinator)                   │             │
│  └──┬────────┬──────────┬──────────┬──────────┬──┘             │
│     │        │          │          │          │                 │
└─────┼────────┼──────────┼──────────┼──────────┼─────────────────┘
      │        │          │          │          │
┌─────┼────────┼──────────┼──────────┼──────────┼─────────────────┐
│     │        │          │          │          │                 │
│     │  Service Layer    │          │          │                 │
│ ┌───▼────┐ ┌▼────────┐ ┌▼────────┐ ┌▼───────┐ │                │
│ │ Epoch  │ │ Safety  │ │  Rate   │ │Treasury│ │                │
│ │Manager │ │ Module  │ │ Oracle  │ │        │ │                │
│ └────────┘ └─────────┘ └─────────┘ └────────┘ │                │
│                           │                     │                │
└───────────────────────────┼─────────────────────┼────────────────┘
                            │                     │
┌───────────────────────────┼─────────────────────┼────────────────┐
│                           │  Strategy Layer     │                │
│                    ┌──────▼────────┐            │                │
│                    │ VaultStrategy  │            │                │
│                    │  (Pluggable)   │            │                │
│                    └──────┬─────────┘            │                │
│                           │                      │                │
└───────────────────────────┼──────────────────────┼────────────────┘
                            │                      │
┌───────────────────────────┼──────────────────────┼────────────────┐
│                           │ External Layer       │                │
│                    ┌──────▼────────┐     ┌──────▼──────┐        │
│                    │  mETH Staking  │     │ DeFi Pools  │        │
│                    │   (Mantle LST) │     │ (Aave, etc) │        │
│                    └────────────────┘     └─────────────┘        │
└──────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. CoreVault

**Purpose**: Central coordinator for the entire protocol

**Responsibilities**:

- Track total Senior and Junior principal
- Harvest yield from strategies
- Distribute yield via waterfall mechanism
- Synchronize Senior rate with oracle
- Enforce safety thresholds

**Key State Variables**:

```solidity
IERC20 public immutable ASSET;              // Underlying asset (USDC)
ISeniorTranche public immutable SENIOR_VAULT;
IJuniorTranche public immutable JUNIOR_VAULT;
IVaultStrategy public strategy;
IDOORRateOracle public rateOracle;

uint256 public seniorPrincipal;
uint256 public juniorPrincipal;
uint256 public seniorFixedRate;             // Current rate in basis points
uint256 public protocolFeeRate;             // 2% (200 basis points)
```

**Key Functions**:

```solidity
function harvest() external onlyRole(KEEPER_ROLE)
function depositToStrategy(uint256 amount) external onlyRole(KEEPER_ROLE)
function syncSeniorRate() external
function registerSeniorDeposit(uint256 amount) external onlyTranche
function registerJuniorDeposit(uint256 amount) external onlyTranche
```

### 2. SeniorVault (sDOOR)

**Purpose**: Fixed-rate tranche with priority claim on yields

**Inheritance**: `ERC4626`, `AccessControl`, `ReentrancyGuard`

**Key Features**:

- Implements ERC-4626 tokenized vault standard
- Tracks accumulated yield separately from principal
- Maintains last update timestamp for yield calculations
- Protected by Junior capital acting as first-loss buffer

**State Variables**:

```solidity
uint256 public totalYield;              // Accumulated yield not yet withdrawn
uint256 public lastYieldUpdate;         // Last timestamp yield was updated
uint256 public immutable SECONDS_PER_YEAR = 365 days;
```

**Key Functions**:

```solidity
function deposit(uint256 assets, address receiver) external returns (uint256 shares)
function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares)
function addYield(uint256 yieldAmount) external onlyCore
function calculateAccruedYield() public view returns (uint256)
```

**Yield Calculation**:

```solidity
function calculateAccruedYield() public view returns (uint256) {
    if (totalAssets() == 0) return 0;

    uint256 timeElapsed = block.timestamp - lastYieldUpdate;
    uint256 principal = coreVault.seniorPrincipal();
    uint256 rate = coreVault.seniorFixedRate();

    // Annual yield = principal * rate / 10000
    // Accrued yield = annual * timeElapsed / SECONDS_PER_YEAR
    return (principal * rate * timeElapsed) / (10_000 * SECONDS_PER_YEAR);
}
```

### 3. JuniorVault (jDOOR)

**Purpose**: Leveraged tranche with first-loss position

**Inheritance**: `ERC4626`, `AccessControl`, `ReentrancyGuard`

**Key Features**:

- Receives all excess yield after Senior obligations
- Acts as buffer to protect Senior from losses
- Principal can be slashed in case of strategy losses
- Tracks deficit for potential future recovery

**State Variables**:

```solidity
uint256 public accumulatedYield;        // Total yield received
uint256 public principalDeficit;        // Deficit if principal was slashed
```

**Key Functions**:

```solidity
function deposit(uint256 assets, address receiver) external returns (uint256 shares)
function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares)
function addYield(uint256 yieldAmount) external onlyCore
function slashPrincipal(uint256 lossAmount) external onlyCore returns (uint256 slashed)
function calculateLeverage() public view returns (uint256)
```

**Leverage Calculation**:

```solidity
function calculateLeverage() public view returns (uint256) {
    uint256 juniorPrincipal = coreVault.juniorPrincipal();
    if (juniorPrincipal == 0) return 0;

    uint256 totalPrincipal = coreVault.seniorPrincipal() + juniorPrincipal;

    // Leverage = totalPrincipal / juniorPrincipal
    // Returns in 18 decimals (e.g., 5e18 = 5x leverage)
    return (totalPrincipal * 1e18) / juniorPrincipal;
}
```

### 4. EpochManager

**Purpose**: Manage withdrawal windows and early withdrawal penalties

**Epoch Structure**:

```solidity
struct Epoch {
    uint256 id;                     // Epoch identifier
    uint256 startTime;              // Epoch start timestamp
    uint256 endTime;                // Epoch end timestamp
    uint256 totalRequested;         // Total withdrawal requests
    bool finalized;                 // Whether epoch is finalized
}
```

**Withdrawal Queue**:

```solidity
struct WithdrawalRequest {
    uint256 amount;                 // Amount requested
    uint256 epochId;                // Epoch when requested
    bool processed;                 // Whether processed
    address vault;                  // Vault address (Senior or Junior)
}
```

**Key Functions**:

```solidity
function requestWithdrawal(address vault, uint256 amount, address user) external
function processWithdrawal(address vault, address user, uint256 epochId) external
function advanceEpoch() external onlyRole(KEEPER_ROLE)
function calculateEarlyWithdrawalPenalty(uint256 amount) public view returns (uint256)
```

**Early Withdrawal Penalty**:

```solidity
function calculateEarlyWithdrawalPenalty(uint256 amount) public view returns (uint256) {
    // Default: 1% penalty (100 basis points)
    return (amount * earlyWithdrawalPenalty) / 10_000;
}
```

### 5. SafetyModule

**Purpose**: Real-time risk management and safety threshold enforcement

**Safety Levels**:

```solidity
enum SafetyLevel {
    HEALTHY,        // >= 15% junior ratio
    WARNING,        // 10-15% junior ratio
    DANGER,         // 5-10% junior ratio
    CRITICAL        // < 5% junior ratio
}
```

**Safety Metrics**:

```solidity
struct SafetyMetrics {
    uint256 juniorRatio;            // Current junior/total ratio (basis points)
    uint256 leverage;               // Current leverage factor
    SafetyLevel level;              // Current safety level
    uint256 seniorDepositCap;       // Dynamic senior deposit limit
    bool depositsEnabled;           // Whether deposits are allowed
}
```

**Key Functions**:

```solidity
function performHealthCheck() external returns (SafetyLevel)
function updateSafetyLevel() external onlyRole(KEEPER_ROLE)
function calculateJuniorRatio() public view returns (uint256)
function calculateDynamicSeniorCap() public view returns (uint256)
```

**Dynamic Senior Cap Calculation**:

```solidity
function calculateDynamicSeniorCap() public view returns (uint256) {
    uint256 juniorPrincipal = coreVault.juniorPrincipal();
    uint256 minRatio = minJuniorRatio;  // e.g., 500 (5%)

    // If junior ratio must be >= 5%, then senior can be up to:
    // senior / (senior + junior) <= 95%
    // senior <= junior * 19

    return (juniorPrincipal * (10_000 - minRatio)) / minRatio;
}
```

### 6. DOORRateOracle

**Purpose**: Calculate and manage the Decentralized Offered Rate (DOR)

**Rate Source**:

```solidity
struct RateSource {
    uint256 rate;                   // Rate in basis points
    uint256 weight;                 // Weight in calculation
    uint256 lastUpdate;             // Last update timestamp
    bool active;                    // Whether source is active
}
```

**Key Functions**:

```solidity
function updateRate(bytes32 sourceId, uint256 newRate) external onlyRole(ORACLE_ROLE)
function getDOR() external view returns (uint256)
function getSeniorTargetAPY() external view returns (uint256)
function challengeRate(bytes32 sourceId, uint256 challengedRate) external
```

**DOR Calculation**:

```solidity
function getDOR() external view returns (uint256) {
    uint256 totalWeightedRate = 0;
    uint256 totalWeight = 0;

    for (uint256 i = 0; i < activeSources.length; i++) {
        RateSource storage source = sources[activeSources[i]];
        if (!source.active) continue;

        totalWeightedRate += source.rate * source.weight;
        totalWeight += source.weight;
    }

    require(totalWeight > 0, "No active sources");

    return totalWeightedRate / totalWeight;
}
```

**Senior Target APY**:

```solidity
function getSeniorTargetAPY() external view returns (uint256) {
    uint256 dor = this.getDOR();
    return dor + seniorPremium;  // e.g., DOR + 100 (1%)
}
```

### 7. VaultStrategy

**Purpose**: Manage yield generation strategies

**Strategy Interface**:

```solidity
interface IVaultStrategy {
    function allocate(uint256 amount) external returns (uint256 allocated);
    function deallocate(uint256 amount) external returns (uint256 withdrawn);
    function harvest() external returns (uint256 yield);
    function totalAssets() external view returns (uint256);
    function estimatedAPY() external view returns (uint256);
}
```

**Current Implementation**: mETH Staking Strategy

- Stakes USDC → mETH
- Earns staking rewards + Mantle incentives
- Harvests and compounds rewards
- Provides liquidity for withdrawals

---

## Contract Interactions

### Deposit Flow (Senior)

```
User
  │
  ├─► 1. approve(seniorVault, amount)
  │
  ▼
SeniorVault
  │
  ├─► 2. deposit(amount, user)
  │   └─► transferFrom(user, this, amount)
  │
  ├─► 3. registerSeniorDeposit(amount)
  │
  ▼
CoreVault
  │
  ├─► 4. Update seniorPrincipal
  │
  ├─► 5. Check SafetyModule
  │       │
  │       ▼
  │   SafetyModule.performHealthCheck()
  │       │
  │       └─► Ensure deposits allowed
  │
  ├─► 6. depositToStrategy(amount)
  │
  ▼
VaultStrategy
  │
  └─► 7. allocate(amount) → invest in mETH
```

### Harvest Flow

```
Keeper
  │
  ▼
CoreVault.harvest()
  │
  ├─► 1. Call strategy.harvest()
  │       │
  │       ▼
  │   VaultStrategy
  │       │
  │       └─► Collect yield from mETH
  │
  ├─► 2. Calculate protocol fee (2%)
  │       └─► Transfer fee to treasury
  │
  ├─► 3. Calculate senior obligation
  │       │
  │       └─► seniorYield = seniorPrincipal * rate * time / (10000 * year)
  │
  ├─► 4. Distribute senior yield
  │       │
  │       ▼
  │   SeniorVault.addYield(seniorYield)
  │       │
  │       └─► Update totalYield
  │
  ├─► 5. Calculate remaining yield
  │       │
  │       └─► juniorYield = totalYield - protocolFee - seniorYield
  │
  └─► 6. Distribute junior yield
          │
          ▼
      JuniorVault.addYield(juniorYield)
          │
          └─► Update accumulatedYield
```

### Withdrawal Flow (Epoch-Based)

```
User
  │
  ├─► 1. Request withdrawal
  │
  ▼
EpochManager.requestWithdrawal(vault, amount, user)
  │
  ├─► 2. Create WithdrawalRequest
  │       └─► Queue for current epoch
  │
  ├─► 3. Wait for epoch end (7 days)
  │
  ▼
Keeper calls EpochManager.advanceEpoch()
  │
  ├─► 4. Finalize previous epoch
  │
  ▼
User calls EpochManager.processWithdrawal(vault, user, epochId)
  │
  ├─► 5. Verify epoch finalized
  │
  ├─► 6. Call vault.withdraw(amount)
  │       │
  │       ▼
  │   SeniorVault/JuniorVault
  │       │
  │       ├─► Burn shares
  │       ├─► Transfer assets to user
  │       └─► registerWithdrawal() to CoreVault
  │
  └─► 7. Mark request as processed
```

---

## Waterfall Distribution Mechanism

### Distribution Priority

```
┌─────────────────────────────────────────────────────────────┐
│                   Waterfall Distribution                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Total Yield (from strategies)                               │
│         │                                                     │
│         ▼                                                     │
│  ┌─────────────────────────────────────────┐                │
│  │ 1. Protocol Fee (2%)                     │                │
│  │    └─► Treasury                          │                │
│  └─────────────────────────────────────────┘                │
│         │                                                     │
│         ▼                                                     │
│  ┌─────────────────────────────────────────┐                │
│  │ 2. Senior Obligation                     │                │
│  │    = principal × rate × time / (10k×yr)  │                │
│  │    └─► SeniorVault.addYield()           │                │
│  └─────────────────────────────────────────┘                │
│         │                                                     │
│         ▼                                                     │
│  ┌─────────────────────────────────────────┐                │
│  │ 3. Remaining Yield                       │                │
│  │    = total - protocolFee - seniorYield   │                │
│  │    └─► JuniorVault.addYield()           │                │
│  └─────────────────────────────────────────┘                │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Implementation (WaterfallMath Library)

```solidity
library WaterfallMath {
    struct DistributionResult {
        uint256 protocolFee;
        uint256 seniorYield;
        uint256 juniorYield;
    }

    function calculateDistribution(
        uint256 totalYield,
        uint256 seniorPrincipal,
        uint256 seniorRate,
        uint256 timeElapsed,
        uint256 protocolFeeRate
    ) internal pure returns (DistributionResult memory) {
        DistributionResult memory result;

        // 1. Protocol fee
        result.protocolFee = (totalYield * protocolFeeRate) / 10_000;

        // 2. Senior obligation
        result.seniorYield = (seniorPrincipal * seniorRate * timeElapsed)
                            / (10_000 * 365 days);

        // 3. Junior gets remainder
        uint256 distributed = result.protocolFee + result.seniorYield;
        result.juniorYield = totalYield > distributed
                            ? totalYield - distributed
                            : 0;

        return result;
    }
}
```

### Loss Absorption Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Loss Absorption Order                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Strategy Loss Detected                                      │
│         │                                                     │
│         ▼                                                     │
│  ┌─────────────────────────────────────────┐                │
│  │ 1. Junior Accumulated Yield              │                │
│  │    └─► Reduce juniorYield first         │                │
│  └─────────────────────────────────────────┘                │
│         │                                                     │
│         ▼ (if loss > yield)                                  │
│  ┌─────────────────────────────────────────┐                │
│  │ 2. Junior Principal                      │                │
│  │    └─► slashPrincipal(lossAmount)       │                │
│  │    └─► Track principalDeficit           │                │
│  └─────────────────────────────────────────┘                │
│         │                                                     │
│         ▼ (if loss > all junior capital)                     │
│  ┌─────────────────────────────────────────┐                │
│  │ 3. Senior Principal (CRITICAL)           │                │
│  │    └─► Emergency mode activated         │                │
│  │    └─► Protocol-wide pause              │                │
│  └─────────────────────────────────────────┘                │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Safety Module Design

### Health Check Algorithm

```solidity
function performHealthCheck() external returns (SafetyLevel) {
    // 1. Calculate current junior ratio
    uint256 juniorRatio = calculateJuniorRatio();

    // 2. Determine safety level
    SafetyLevel level;
    if (juniorRatio >= healthyThreshold) {          // >= 15%
        level = SafetyLevel.HEALTHY;
    } else if (juniorRatio >= warningThreshold) {   // 10-15%
        level = SafetyLevel.WARNING;
    } else if (juniorRatio >= minJuniorRatio) {     // 5-10%
        level = SafetyLevel.DANGER;
    } else {                                        // < 5%
        level = SafetyLevel.CRITICAL;
    }

    // 3. Update state and apply restrictions
    updateSafetyRestrictions(level);

    emit SafetyLevelChanged(level, juniorRatio);
    return level;
}
```

### Safety Level Actions

| Level        | Junior Ratio | Actions                                                                                |
| ------------ | ------------ | -------------------------------------------------------------------------------------- |
| **HEALTHY**  | ≥ 15%        | • Normal operations<br>• No restrictions<br>• Full deposit capacity                    |
| **WARNING**  | 10-15%       | • Alert keepers<br>• Increase monitoring<br>• Prepare for restrictions                 |
| **DANGER**   | 5-10%        | • Pause senior deposits<br>• Reduce senior rate by 10%<br>• Increase junior incentives |
| **CRITICAL** | < 5%         | • Pause all deposits<br>• Withdrawal-only mode<br>• Emergency governance vote          |

### Dynamic Restrictions

```solidity
function updateSafetyRestrictions(SafetyLevel level) internal {
    if (level == SafetyLevel.HEALTHY) {
        // No restrictions
        seniorDepositCap = calculateDynamicSeniorCap();
        juniorDepositCap = type(uint256).max;
        depositsEnabled = true;

    } else if (level == SafetyLevel.WARNING) {
        // Monitor closely
        emit SafetyWarning("Junior ratio approaching minimum");

    } else if (level == SafetyLevel.DANGER) {
        // Restrict senior deposits
        seniorDepositCap = coreVault.seniorPrincipal(); // No new senior
        emit SafetyAlert("Senior deposits paused");

        // Reduce senior rate to incentivize junior deposits
        uint256 currentRate = coreVault.seniorFixedRate();
        coreVault.setSeniorFixedRate(currentRate * 90 / 100); // -10%

    } else if (level == SafetyLevel.CRITICAL) {
        // Emergency mode
        seniorDepositCap = 0;
        juniorDepositCap = 0;
        depositsEnabled = false;
        emit EmergencyMode("All deposits paused");
    }
}
```

---

## Oracle System

### Multi-Source Rate Aggregation

```
┌─────────────────────────────────────────────────────────────┐
│                  Rate Oracle Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  External Rate Sources                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │ Treehouse│  │  Aave    │  │ Compound │                 │
│  │   TESR   │  │  USDC    │  │   USDC   │                 │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                 │
│       │             │              │                        │
│       ▼             ▼              ▼                        │
│  ┌─────────────────────────────────────┐                   │
│  │      DOORRateOracle                 │                   │
│  │                                      │                   │
│  │  Weighted Average Calculation:       │                   │
│  │  DOR = Σ(rate_i × weight_i) / Σ(w_i)│                   │
│  └─────────────────┬───────────────────┘                   │
│                    │                                        │
│                    ▼                                        │
│           Senior Target APY                                │
│           = DOR + 1% premium                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Rate Update Flow with Challenge Mechanism

```solidity
function updateRate(bytes32 sourceId, uint256 newRate) external onlyRole(ORACLE_ROLE) {
    RateSource storage source = sources[sourceId];
    require(source.active, "Inactive source");

    uint256 currentRate = source.rate;
    uint256 rateChange = newRate > currentRate
        ? newRate - currentRate
        : currentRate - newRate;

    // Large changes require challenge period
    if (rateChange > maxRateChange) {  // e.g., 2% = 200 basis points
        // Create challenge window
        pendingRateUpdates[sourceId] = PendingUpdate({
            newRate: newRate,
            timestamp: block.timestamp,
            finalized: false
        });

        emit RateUpdateChallenged(sourceId, currentRate, newRate);
        return;
    }

    // Small changes applied immediately
    source.rate = newRate;
    source.lastUpdate = block.timestamp;
    emit RateUpdated(sourceId, newRate);
}
```

### Challenge Period

```solidity
function finalizeRateUpdate(bytes32 sourceId) external {
    PendingUpdate storage pending = pendingRateUpdates[sourceId];
    require(!pending.finalized, "Already finalized");
    require(
        block.timestamp >= pending.timestamp + challengePeriod,
        "Challenge period not ended"
    );

    // Apply rate update after challenge period
    sources[sourceId].rate = pending.newRate;
    sources[sourceId].lastUpdate = block.timestamp;
    pending.finalized = true;

    emit RateUpdateFinalized(sourceId, pending.newRate);
}
```

---

## Epoch Management

### Epoch Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                     Epoch Lifecycle                          │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Day 0              Epoch N Starts                           │
│  │                  ┌──────────────┐                        │
│  ├─► Request ───────┤   ACTIVE     │                        │
│  │   Withdrawal     │              │                        │
│  │                  │ Users can:   │                        │
│  │                  │ - Request    │                        │
│  │                  │   withdrawals│                        │
│  │                  │ - Queue up   │                        │
│  │                  └──────┬───────┘                        │
│  │                         │                                 │
│  Day 7            ┌────────▼────────┐                       │
│  │                │ Epoch Advance    │                       │
│  ├─► Keeper ──────┤ (Keeper Role)   │                       │
│  │   calls        │                 │                       │
│  │   advanceEpoch()└────────┬───────┘                       │
│  │                          │                                │
│  │                 ┌────────▼────────┐                      │
│  │                 │  FINALIZED      │                      │
│  ├─► Process ──────┤                 │                      │
│  │   Withdrawals   │ Users can:      │                      │
│  │                 │ - Process queued│                      │
│  │                 │   withdrawals   │                      │
│  │                 └─────────────────┘                      │
│  │                                                           │
│  │                 Epoch N+1 Starts                         │
│  └────────────────────────────────────────────────────────►│
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Early Withdrawal

Users can withdraw before epoch end by paying a penalty:

```solidity
function earlyWithdraw(address vault, uint256 amount) external {
    // Calculate penalty (default 1%)
    uint256 penalty = (amount * earlyWithdrawalPenalty) / 10_000;
    uint256 netAmount = amount - penalty;

    // Transfer penalty to treasury or redistribute to other users
    IERC20(asset).safeTransfer(treasury, penalty);

    // Process withdrawal immediately
    _processWithdrawal(vault, msg.sender, netAmount);

    emit EarlyWithdrawal(msg.sender, vault, amount, penalty);
}
```

---

## Strategy Architecture

### Strategy Interface

All yield strategies must implement:

```solidity
interface IVaultStrategy {
    /// @notice Allocate funds to the strategy
    function allocate(uint256 amount) external returns (uint256 allocated);

    /// @notice Deallocate funds from the strategy
    function deallocate(uint256 amount) external returns (uint256 withdrawn);

    /// @notice Harvest yields and return amount
    function harvest() external returns (uint256 yield);

    /// @notice Total assets under management
    function totalAssets() external view returns (uint256);

    /// @notice Estimated APY (basis points)
    function estimatedAPY() external view returns (uint256);
}
```

### mETH Staking Strategy

Current implementation:

```solidity
contract METHStakingStrategy is IVaultStrategy {
    IERC20 public immutable USDC;
    IERC20 public immutable METH;
    IMantle public immutable MANTLE_STAKING;

    function allocate(uint256 usdcAmount) external override returns (uint256) {
        // 1. Swap USDC to METH
        uint256 methAmount = _swapUSDCtoMETH(usdcAmount);

        // 2. Stake mETH on Mantle
        MANTLE_STAKING.stake(methAmount);

        return methAmount;
    }

    function harvest() external override returns (uint256) {
        // 1. Claim staking rewards
        uint256 rewards = MANTLE_STAKING.claimRewards();

        // 2. Compound or convert to USDC
        uint256 yieldInUSDC = _swapMETHtoUSDC(rewards);

        return yieldInUSDC;
    }
}
```

---

## Access Control

### Role Hierarchy

```
DEFAULT_ADMIN_ROLE
    │
    ├─► KEEPER_ROLE
    │   └─► Automated operations (harvest, health checks, epoch advance)
    │
    ├─► ORACLE_ROLE
    │   └─► Rate updates
    │
    ├─► EMERGENCY_ROLE
    │   └─► Emergency pause and recovery
    │
    └─► STRATEGY_ROLE
        └─► Strategy management (allocate, deallocate)
```

### Role Permissions

| Role                   | Permissions                                                                        |
| ---------------------- | ---------------------------------------------------------------------------------- |
| **DEFAULT_ADMIN_ROLE** | • Grant/revoke roles<br>• Update protocol parameters<br>• Change treasury address  |
| **KEEPER_ROLE**        | • harvest()<br>• depositToStrategy()<br>• performHealthCheck()<br>• advanceEpoch() |
| **ORACLE_ROLE**        | • updateRate()<br>• addRateSource()<br>• removeRateSource()                        |
| **EMERGENCY_ROLE**     | • pause()<br>• unpause()<br>• emergencyWithdraw()                                  |
| **STRATEGY_ROLE**      | • setStrategy()<br>• allocateToStrategy()<br>• deallocateFromStrategy()            |

---

## Emergency Procedures

### Emergency Pause

```solidity
function emergencyPause() external onlyRole(EMERGENCY_ROLE) {
    emergencyMode = true;

    // Pause all deposits
    SENIOR_VAULT.pause();
    JUNIOR_VAULT.pause();

    // Allow withdrawals only
    emit EmergencyPaused(msg.sender);
}
```

### Emergency Withdrawal

```solidity
function emergencyWithdraw() external onlyRole(EMERGENCY_ROLE) {
    require(emergencyMode, "Not in emergency mode");

    // Withdraw all funds from strategy
    uint256 withdrawn = strategy.deallocate(strategy.totalAssets());

    // Hold in CoreVault for manual distribution
    emit EmergencyWithdrawal(withdrawn);
}
```

### Recovery Procedure

1. **Pause Protocol**: `emergencyPause()`
2. **Assess Damage**: Calculate losses and affected users
3. **Withdraw from Strategies**: `emergencyWithdraw()`
4. **Governance Vote**: Community decides on recovery plan
5. **Execute Recovery**: Redistribute funds according to approved plan
6. **Resume Operations**: `unpause()` after fixes

---

## Gas Optimization

### ERC-4626 Benefits

- Share-based accounting (no loops over users)
- Single storage slot updates per deposit/withdraw
- View functions for off-chain calculations

### Storage Packing

```solidity
// Packed into single slot (256 bits)
struct CompactState {
    uint128 amount;         // 128 bits
    uint64 timestamp;       // 64 bits
    uint32 rate;            // 32 bits
    uint16 level;           // 16 bits
    bool active;            // 8 bits
}
```

### Batch Operations

- Multiple withdrawals processed in single transaction
- Batch harvest and distribute in one call

---

## Security Considerations

### Reentrancy Protection

All external calls protected with `ReentrancyGuard`:

```solidity
function deposit(uint256 amount) external nonReentrant {
    // Safe from reentrancy
}
```

### Integer Overflow

- Solidity 0.8.26 built-in protection
- No unchecked blocks unless explicitly safe

### Rate Limiting

- Max rate change: 2% per update
- Challenge period: 24 hours for large changes
- Emergency cooldown: 24 hours between emergency actions

### Multi-Signature Requirements

Critical operations require multi-sig:

- Protocol upgrades (3/5)
- Treasury changes (3/5)
- Emergency actions (2/5)

---

## Upgradeability

### Current Implementation

- Contracts are **non-upgradeable** for security and trust
- Parameters are configurable by governance
- Strategy is replaceable via `setStrategy()`

### Future Considerations

If upgradeability is needed:

- Use OpenZeppelin TransparentUpgradeableProxy
- Implement 48-hour timelock for upgrades
- Require 4/5 multi-sig approval

---

This architecture ensures a secure, efficient, and flexible structured DeFi product with clear separation of concerns and robust risk management.
