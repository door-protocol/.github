# DOOR Protocol - API Reference

## Table of Contents

1. [Core Contracts](#core-contracts)
2. [Tranche Vaults](#tranche-vaults)
3. [Management Contracts](#management-contracts)
4. [Integration Examples](#integration-examples)
5. [Events Reference](#events-reference)
6. [Error Reference](#error-reference)

---

## Core Contracts

### CoreVault

Central coordinator for DOOR Protocol.

**Contract Address**: See [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md)

#### Read Functions

##### `seniorPrincipal()`

Returns total Senior principal registered.

```solidity
function seniorPrincipal() external view returns (uint256)
```

**Returns**: Total Senior principal in USDC (6 decimals)

**Example**:

```typescript
const seniorPrincipal = await coreVault.read.seniorPrincipal();
console.log(`Senior Principal: ${seniorPrincipal / 1e6} USDC`);
```

##### `juniorPrincipal()`

Returns total Junior principal registered.

```solidity
function juniorPrincipal() external view returns (uint256)
```

**Returns**: Total Junior principal in USDC (6 decimals)

##### `seniorFixedRate()`

Returns current Senior fixed rate.

```solidity
function seniorFixedRate() external view returns (uint256)
```

**Returns**: Rate in basis points (e.g., 550 = 5.5%)

##### `protocolFeeRate()`

Returns protocol fee rate.

```solidity
function protocolFeeRate() external view returns (uint256)
```

**Returns**: Fee rate in basis points (e.g., 200 = 2%)

#### Write Functions

##### `harvest()`

Harvests yield from strategy and distributes via waterfall.

```solidity
function harvest() external onlyRole(KEEPER_ROLE)
```

**Access**: KEEPER_ROLE only

**Events**: `YieldHarvested`, `YieldDistributed`

**Example**:

```typescript
const tx = await coreVault.write.harvest();
await tx.wait();
```

##### `depositToStrategy(uint256 amount)`

Deposits funds to yield strategy.

```solidity
function depositToStrategy(uint256 amount) external onlyRole(KEEPER_ROLE)
```

**Parameters**:

- `amount`: Amount to deposit (USDC, 6 decimals)

**Access**: KEEPER_ROLE only

##### `syncSeniorRate()`

Synchronizes Senior rate with oracle.

```solidity
function syncSeniorRate() external
```

**Access**: Public (anyone can call)

**Example**:

```typescript
await coreVault.write.syncSeniorRate();
```

---

## Tranche Vaults

### SeniorVault (sDOOR)

Fixed-rate tranche implementing ERC-4626.

**Contract Address**: See [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md)

#### ERC-4626 Functions

##### `deposit(uint256 assets, address receiver)`

Deposits USDC and mints sDOOR shares.

```solidity
function deposit(uint256 assets, address receiver) external returns (uint256 shares)
```

**Parameters**:

- `assets`: Amount of USDC to deposit (6 decimals)
- `receiver`: Address to receive shares

**Returns**: Number of shares minted (18 decimals)

**Example**:

```typescript
// 1. Approve USDC
await usdc.write.approve([seniorVault.address, 10000n * 10n ** 6n]);

// 2. Deposit
const shares = await seniorVault.write.deposit([
  10000n * 10n ** 6n,
  userAddress,
]);
console.log(`Minted ${shares} sDOOR shares`);
```

##### `withdraw(uint256 assets, address receiver, address owner)`

Burns shares and withdraws USDC.

```solidity
function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares)
```

**Parameters**:

- `assets`: Amount of USDC to withdraw (6 decimals)
- `receiver`: Address to receive USDC
- `owner`: Owner of the shares

**Returns**: Number of shares burned

**Example**:

```typescript
const shares = await seniorVault.write.withdraw([
  5000n * 10n ** 6n,
  userAddress,
  userAddress,
]);
```

##### `redeem(uint256 shares, address receiver, address owner)`

Burns exact number of shares and withdraws corresponding USDC.

```solidity
function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assets)
```

**Parameters**:

- `shares`: Number of shares to burn (18 decimals)
- `receiver`: Address to receive USDC
- `owner`: Owner of the shares

**Returns**: Amount of USDC withdrawn

##### `totalAssets()`

Returns total assets held by vault.

```solidity
function totalAssets() external view returns (uint256)
```

**Returns**: Total USDC in vault (6 decimals)

##### `convertToShares(uint256 assets)`

Calculates shares for given assets.

```solidity
function convertToShares(uint256 assets) external view returns (uint256 shares)
```

**Example**:

```typescript
const shares = await seniorVault.read.convertToShares([1000n * 10n ** 6n]);
console.log(`1000 USDC = ${shares} shares`);
```

##### `convertToAssets(uint256 shares)`

Calculates assets for given shares.

```solidity
function convertToAssets(uint256 shares) external view returns (uint256 assets)
```

#### DOOR-Specific Functions

##### `currentAPY()`

Returns current APY.

```solidity
function currentAPY() external view returns (uint256)
```

**Returns**: APY in basis points (e.g., 550 = 5.5%)

**Example**:

```typescript
const apy = await seniorVault.read.currentAPY();
console.log(`Current APY: ${apy / 100}%`);
```

##### `calculateAccruedYield()`

Calculates accrued yield since last update.

```solidity
function calculateAccruedYield() external view returns (uint256)
```

**Returns**: Accrued yield in USDC (6 decimals)

### JuniorVault (jDOOR)

Leveraged tranche implementing ERC-4626.

**Contract Address**: See [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md)

#### ERC-4626 Functions

Same as SeniorVault:

- `deposit(uint256, address)`
- `withdraw(uint256, address, address)`
- `redeem(uint256, address, address)`
- `totalAssets()`
- `convertToShares(uint256)`
- `convertToAssets(uint256)`

#### DOOR-Specific Functions

##### `calculateLeverage()`

Returns current leverage factor.

```solidity
function calculateLeverage() external view returns (uint256)
```

**Returns**: Leverage factor in 18 decimals (e.g., 5e18 = 5x)

**Example**:

```typescript
const leverage = await juniorVault.read.calculateLeverage();
console.log(`Current Leverage: ${leverage / 10n ** 18n}x`);
```

##### `estimateAPY()`

Estimates Junior APY based on current conditions.

```solidity
function estimateAPY() external view returns (uint256)
```

**Returns**: Estimated APY in basis points

**Example**:

```typescript
const estimatedAPY = await juniorVault.read.estimateAPY();
console.log(`Estimated APY: ${estimatedAPY / 100}%`);
```

##### `principalDeficit()`

Returns principal deficit (if any losses occurred).

```solidity
function principalDeficit() external view returns (uint256)
```

**Returns**: Deficit amount in USDC (6 decimals)

---

## Management Contracts

### EpochManager

Manages withdrawal windows and epochs.

#### Read Functions

##### `getCurrentEpoch()`

Returns current epoch information.

```solidity
function getCurrentEpoch() external view returns (
    uint256 id,
    uint256 startTime,
    uint256 endTime,
    bool finalized
)
```

**Returns**:

- `id`: Current epoch ID
- `startTime`: Epoch start timestamp
- `endTime`: Epoch end timestamp
- `finalized`: Whether epoch is finalized

**Example**:

```typescript
const [id, startTime, endTime, finalized] =
  await epochManager.read.getCurrentEpoch();
console.log(
  `Epoch ${id}: ${new Date(Number(startTime) * 1000)} - ${new Date(
    Number(endTime) * 1000
  )}`
);
```

##### `getWithdrawalRequest(address vault, address user, uint256 epochId)`

Returns withdrawal request details.

```solidity
function getWithdrawalRequest(address vault, address user, uint256 epochId) external view returns (
    uint256 amount,
    uint256 epochId,
    bool processed
)
```

#### Write Functions

##### `requestWithdrawal(address vault, uint256 amount, address user)`

Requests withdrawal for next epoch.

```solidity
function requestWithdrawal(address vault, uint256 amount, address user) external
```

**Parameters**:

- `vault`: Vault address (SeniorVault or JuniorVault)
- `amount`: Amount to withdraw (USDC, 6 decimals)
- `user`: User address

**Example**:

```typescript
await epochManager.write.requestWithdrawal([
  seniorVault.address,
  5000n * 10n ** 6n,
  userAddress,
]);
```

##### `processWithdrawal(address vault, address user, uint256 epochId)`

Processes withdrawal request after epoch ends.

```solidity
function processWithdrawal(address vault, address user, uint256 epochId) external
```

**Example**:

```typescript
const [currentEpochId] = await epochManager.read.getCurrentEpoch();
await epochManager.write.processWithdrawal([
  seniorVault.address,
  userAddress,
  currentEpochId - 1n, // Previous epoch
]);
```

##### `advanceEpoch()`

Advances to next epoch (keeper only).

```solidity
function advanceEpoch() external onlyRole(KEEPER_ROLE)
```

### SafetyModule

Manages risk and safety levels.

#### Read Functions

##### `performHealthCheck()`

Performs health check and returns safety level.

```solidity
function performHealthCheck() external view returns (SafetyLevel)
```

**Returns**: Safety level (0=HEALTHY, 1=WARNING, 2=DANGER, 3=CRITICAL)

**Example**:

```typescript
const level = await safetyModule.read.performHealthCheck();
const levels = ['HEALTHY', 'WARNING', 'DANGER', 'CRITICAL'];
console.log(`Safety Level: ${levels[level]}`);
```

##### `calculateJuniorRatio()`

Calculates current junior ratio.

```solidity
function calculateJuniorRatio() external view returns (uint256)
```

**Returns**: Ratio in basis points (e.g., 2500 = 25%)

**Example**:

```typescript
const ratio = await safetyModule.read.calculateJuniorRatio();
console.log(`Junior Ratio: ${ratio / 100}%`);
```

##### `getSafetyMetrics()`

Returns comprehensive safety metrics.

```solidity
function getSafetyMetrics() external view returns (
    uint256 juniorRatio,
    uint256 leverage,
    SafetyLevel level,
    uint256 seniorDepositCap,
    bool depositsEnabled
)
```

**Example**:

```typescript
const [juniorRatio, leverage, level, cap, enabled] =
  await safetyModule.read.getSafetyMetrics();
console.log({
  juniorRatio: juniorRatio / 100,
  leverage: leverage / 10n ** 18n,
  level,
  seniorDepositCap: cap / 10n ** 6n,
  depositsEnabled: enabled,
});
```

### DOORRateOracle

Manages decentralized offered rate.

#### Read Functions

##### `getDOR()`

Returns current Decentralized Offered Rate.

```solidity
function getDOR() external view returns (uint256)
```

**Returns**: DOR in basis points

**Example**:

```typescript
const dor = await rateOracle.read.getDOR();
console.log(`DOR: ${dor / 100}%`);
```

##### `getSeniorTargetAPY()`

Returns Senior target APY (DOR + premium).

```solidity
function getSeniorTargetAPY() external view returns (uint256)
```

**Returns**: Target APY in basis points

**Example**:

```typescript
const targetAPY = await rateOracle.read.getSeniorTargetAPY();
console.log(`Senior Target APY: ${targetAPY / 100}%`);
```

#### Write Functions

##### `updateRate(bytes32 sourceId, uint256 newRate)`

Updates rate for a source (oracle role only).

```solidity
function updateRate(bytes32 sourceId, uint256 newRate) external onlyRole(ORACLE_ROLE)
```

**Parameters**:

- `sourceId`: Source identifier (e.g., keccak256("TESR"))
- `newRate`: New rate in basis points

---

## Integration Examples

### Frontend Integration (React + Viem)

```typescript
import { createPublicClient, createWalletClient, http } from 'viem';
import { mantleSepoliaTestnet } from 'viem/chains';
import SeniorVaultABI from './abis/SeniorVault.json';
import JuniorVaultABI from './abis/JuniorVault.json';

// Initialize clients
const publicClient = createPublicClient({
  chain: mantleSepoliaTestnet,
  transport: http('https://rpc.sepolia.mantle.xyz'),
});

const walletClient = createWalletClient({
  chain: mantleSepoliaTestnet,
  transport: window.ethereum,
});

// Contract addresses
const SENIOR_VAULT = '0x34BC889a143870bBd8538EAe6421cA4c62e84bc3';
const USDC = '0x9a54bad93a00bf1232d4e636f5e53055dc0b8238';

// Read vault stats
async function getVaultStats() {
  const [totalAssets, currentAPY] = await Promise.all([
    publicClient.readContract({
      address: SENIOR_VAULT,
      abi: SeniorVaultABI,
      functionName: 'totalAssets',
    }),
    publicClient.readContract({
      address: SENIOR_VAULT,
      abi: SeniorVaultABI,
      functionName: 'currentAPY',
    }),
  ]);

  return {
    tvl: Number(totalAssets) / 1e6,
    apy: Number(currentAPY) / 100,
  };
}

// Deposit to Senior Vault
async function depositSenior(amount: bigint, userAddress: `0x${string}`) {
  // 1. Approve USDC
  const approveHash = await walletClient.writeContract({
    address: USDC,
    abi: [
      {
        name: 'approve',
        type: 'function',
        inputs: [
          { name: 'spender', type: 'address' },
          { name: 'amount', type: 'uint256' },
        ],
        outputs: [{ type: 'bool' }],
      },
    ],
    functionName: 'approve',
    args: [SENIOR_VAULT, amount],
  });

  await publicClient.waitForTransactionReceipt({ hash: approveHash });

  // 2. Deposit
  const depositHash = await walletClient.writeContract({
    address: SENIOR_VAULT,
    abi: SeniorVaultABI,
    functionName: 'deposit',
    args: [amount, userAddress],
  });

  const receipt = await publicClient.waitForTransactionReceipt({
    hash: depositHash,
  });
  return receipt;
}

// Request withdrawal
async function requestWithdrawal(amount: bigint, userAddress: `0x${string}`) {
  const EPOCH_MANAGER = '0x2956e44668E4026D499D46Ad7eCB1312EA8484aa';

  const hash = await walletClient.writeContract({
    address: EPOCH_MANAGER,
    abi: [
      {
        name: 'requestWithdrawal',
        type: 'function',
        inputs: [
          { name: 'vault', type: 'address' },
          { name: 'amount', type: 'uint256' },
          { name: 'user', type: 'address' },
        ],
      },
    ],
    functionName: 'requestWithdrawal',
    args: [SENIOR_VAULT, amount, userAddress],
  });

  return await publicClient.waitForTransactionReceipt({ hash });
}
```

### Backend Integration (Node.js + Ethers)

```typescript
import { ethers } from 'ethers';

// Setup provider
const provider = new ethers.JsonRpcProvider('https://rpc.sepolia.mantle.xyz');
const wallet = new ethers.Wallet(process.env.PRIVATE_KEY!, provider);

// Contract instances
const seniorVault = new ethers.Contract(
  '0x34BC889a143870bBd8538EAe6421cA4c62e84bc3',
  SeniorVaultABI,
  wallet
);

// Keeper: Harvest yields
async function harvest() {
  const coreVault = new ethers.Contract(
    CORE_VAULT_ADDRESS,
    CoreVaultABI,
    wallet
  );

  try {
    const tx = await coreVault.harvest();
    const receipt = await tx.wait();

    console.log(`Harvested! Gas used: ${receipt.gasUsed}`);

    // Parse events
    for (const log of receipt.logs) {
      try {
        const parsed = coreVault.interface.parseLog(log);
        if (parsed?.name === 'YieldDistributed') {
          console.log({
            protocolFee: ethers.formatUnits(parsed.args.protocolFee, 6),
            seniorYield: ethers.formatUnits(parsed.args.seniorYield, 6),
            juniorYield: ethers.formatUnits(parsed.args.juniorYield, 6),
          });
        }
      } catch {}
    }
  } catch (error) {
    console.error('Harvest failed:', error);
  }
}

// Monitor safety level
async function monitorSafety() {
  const safetyModule = new ethers.Contract(
    SAFETY_MODULE_ADDRESS,
    SafetyModuleABI,
    provider
  );

  const [juniorRatio, level] = await Promise.all([
    safetyModule.calculateJuniorRatio(),
    safetyModule.performHealthCheck(),
  ]);

  const levels = ['HEALTHY', 'WARNING', 'DANGER', 'CRITICAL'];

  if (level > 0) {
    console.warn(
      `⚠️ Safety Alert: ${levels[level]} (Junior Ratio: ${juniorRatio / 100}%)`
    );
    // Send alert to monitoring service
  }
}

// Run keeper operations
setInterval(async () => {
  await harvest();
  await monitorSafety();
}, 60 * 60 * 1000); // Every hour
```

### Smart Contract Integration

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IERC4626} from "@openzeppelin/contracts/interfaces/IERC4626.sol";

contract DOORIntegration {
    IERC20 public immutable USDC;
    IERC4626 public immutable SENIOR_VAULT;

    constructor(address _usdc, address _seniorVault) {
        USDC = IERC20(_usdc);
        SENIOR_VAULT = IERC4626(_seniorVault);
    }

    /// @notice Deposit USDC to DOOR Senior Vault
    function depositToDOOR(uint256 amount) external {
        // Transfer USDC from user
        USDC.transferFrom(msg.sender, address(this), amount);

        // Approve Senior Vault
        USDC.approve(address(SENIOR_VAULT), amount);

        // Deposit and receive sDOOR shares
        uint256 shares = SENIOR_VAULT.deposit(amount, msg.sender);

        emit Deposited(msg.sender, amount, shares);
    }

    /// @notice Withdraw USDC from DOOR Senior Vault
    function withdrawFromDOOR(uint256 shares) external {
        // Transfer shares from user to this contract
        IERC20(address(SENIOR_VAULT)).transferFrom(msg.sender, address(this), shares);

        // Redeem shares for USDC
        uint256 assets = SENIOR_VAULT.redeem(shares, msg.sender, address(this));

        emit Withdrawn(msg.sender, assets, shares);
    }

    /// @notice Get user's balance in DOOR
    function getBalance(address user) external view returns (uint256 shares, uint256 assets) {
        shares = SENIOR_VAULT.balanceOf(user);
        assets = SENIOR_VAULT.convertToAssets(shares);
    }

    event Deposited(address indexed user, uint256 amount, uint256 shares);
    event Withdrawn(address indexed user, uint256 assets, uint256 shares);
}
```

---

## Events Reference

### CoreVault Events

```solidity
event YieldHarvested(uint256 totalYield, uint256 timestamp);
event YieldDistributed(uint256 protocolFee, uint256 seniorYield, uint256 juniorYield);
event SeniorDeposit(address indexed user, uint256 amount);
event JuniorDeposit(address indexed user, uint256 amount);
event SeniorWithdrawal(address indexed user, uint256 amount);
event JuniorWithdrawal(address indexed user, uint256 amount);
event SeniorRateUpdated(uint256 oldRate, uint256 newRate);
event ProtocolFeeRateUpdated(uint256 oldRate, uint256 newRate);
event StrategyUpdated(address indexed oldStrategy, address indexed newStrategy);
event EmergencyMode(bool enabled);
```

### EpochManager Events

```solidity
event WithdrawalRequested(address indexed user, address indexed vault, uint256 amount, uint256 epochId);
event WithdrawalProcessed(address indexed user, address indexed vault, uint256 amount, uint256 epochId);
event EpochAdvanced(uint256 oldEpochId, uint256 newEpochId);
event EarlyWithdrawal(address indexed user, address indexed vault, uint256 amount, uint256 penalty);
```

### SafetyModule Events

```solidity
event SafetyLevelChanged(SafetyLevel newLevel, uint256 juniorRatio);
event SafetyWarning(string message);
event SafetyAlert(string message);
event EmergencyModeActivated(string reason);
event DepositCapUpdated(uint256 newSeniorCap);
```

### RateOracle Events

```solidity
event RateUpdated(bytes32 indexed sourceId, uint256 newRate);
event RateUpdateChallenged(bytes32 indexed sourceId, uint256 oldRate, uint256 newRate);
event RateUpdateFinalized(bytes32 indexed sourceId, uint256 finalRate);
event RateSourceAdded(bytes32 indexed sourceId, uint256 initialRate, uint256 weight);
event RateSourceRemoved(bytes32 indexed sourceId);
```

---

## Error Reference

### Common Errors

```solidity
error NotTranche();                     // Caller is not a tranche vault
error AlreadyInitialized();             // Contract already initialized
error NotInitialized();                 // Contract not yet initialized
error EmergencyModeActive();            // Operation blocked due to emergency
error ZeroAddress();                    // Zero address provided
error InvalidRate();                    // Invalid rate value
error InvalidFeeRate();                 // Invalid fee rate
error InsufficientBalance();            // Insufficient balance
error DepositsDisabled();               // Deposits currently disabled
error WithdrawalNotAllowed();           // Withdrawal not allowed yet
error EpochNotFinalized();              // Epoch not finalized
error AlreadyProcessed();               // Request already processed
error InvalidAmount();                  // Invalid amount provided
error UnauthorizedCaller();             // Caller not authorized
error RateLimitExceeded();              // Rate change limit exceeded
error ChallengePeriodActive();          // Challenge period still active
error StaleRateSource();                // Rate source outdated
```

---

## Rate Limits and Constraints

| Parameter                    | Limit         | Reason                                   |
| ---------------------------- | ------------- | ---------------------------------------- |
| **Max Rate Change**          | 2% per update | Prevent oracle manipulation              |
| **Challenge Period**         | 24 hours      | Community review time                    |
| **Min Junior Ratio**         | 5%            | Maintain senior protection               |
| **Epoch Duration**           | 7 days        | Balance liquidity and capital efficiency |
| **Early Withdrawal Penalty** | 1-5%          | Discourage frequent withdrawals          |

---

For more detailed examples and use cases, visit our [GitHub repository](https://github.com/door-protocol/contracts) or join our [Discord community](https://discord.gg/doorprotocol).
