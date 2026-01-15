# DOOR Protocol - Deployment Guide

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Environment Setup](#environment-setup)
3. [Local Development](#local-development)
4. [Testnet Deployment](#testnet-deployment)
5. [Mainnet Deployment](#mainnet-deployment)
6. [Post-Deployment](#post-deployment)
7. [Verification](#verification)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software

- **Node.js**: >= 18.0.0
- **npm** or **yarn**: >= 9.0.0
- **Foundry**: Latest version (forge, cast, anvil)
- **Git**: Latest version

### Install Foundry

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

### Verify Installation

```bash
node --version  # Should be >= 18.0.0
forge --version  # Should display Foundry version
cast --version
anvil --version
```

---

## Environment Setup

### 1. Clone Repository

```bash
git clone https://github.com/door-protocol/contracts.git
cd contracts
```

### 2. Install Dependencies

```bash
# Install npm dependencies
npm install

# Install Foundry dependencies
forge install
```

### 3. Configure Environment Variables

```bash
# Copy example environment file
cp .env.example .env
```

### 4. Edit `.env` File

```env
# Deployer Private Key (NEVER commit this!)
PRIVATE_KEY=0xyour_private_key_here_without_0x_prefix

# RPC URLs
MANTLE_TESTNET_RPC_URL=https://rpc.sepolia.mantle.xyz
MANTLE_MAINNET_RPC_URL=https://rpc.mantle.xyz

# Block Explorers (for verification)
MANTLESCAN_API_KEY=your_mantlescan_api_key_here

# Optional: Etherscan API keys for other networks
ETHERSCAN_API_KEY=your_etherscan_api_key

# Gas Settings (optional)
GAS_PRICE_GWEI=1
GAS_LIMIT=8000000
```

### 5. Obtain Testnet MNT

Visit [Mantle Sepolia Faucet](https://faucet.sepolia.mantle.xyz/) to get testnet MNT for gas fees.

---

## Local Development

### Start Local Node (Anvil)

```bash
# Terminal 1: Start Anvil (Foundry's local node)
npm run anvil
```

Anvil will start on `http://localhost:8545` with 10 test accounts pre-funded with ETH.

### Deploy to Local Node

#### Option 1: Deploy with Mock Tokens (Viem)

```bash
# Terminal 2
npm run deploy:local:mock
```

#### Option 2: Deploy with Mock Tokens (Ethers)

```bash
npm run deploy:local:mock:ethers
```

#### Option 3: Deploy with Forge

```bash
forge script scripts/deploy/DeployDoor.s.sol:DeployDoor \
  --rpc-url http://localhost:8545 \
  --broadcast \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

### Interact with Local Deployment

```bash
# Get deployed addresses
cat deployments/local/latest.json

# Example: Check Senior Vault balance
cast call <SENIOR_VAULT_ADDRESS> "totalAssets()(uint256)" --rpc-url http://localhost:8545
```

---

## Testnet Deployment

### Mantle Sepolia Testnet

**Network Details**:

- Chain ID: 5003
- RPC: https://rpc.sepolia.mantle.xyz
- Explorer: https://explorer.sepolia.mantle.xyz

### Deployment Options

#### Option 1: Using Real Testnet Tokens (Recommended)

**Real Token Addresses (Mantle Sepolia)**:

```
USDC: 0x9a54bad93a00bf1232d4e636f5e53055dc0b8238
mETH: 0x4Ade8aAa0143526393EcadA836224EF21aBC6ac6
```

Get testnet tokens:

- **USDC**: [Mantle Faucet](https://faucet.sepolia.mantle.xyz/)
- **mETH**: Contact Mantle team or bridge testnet ETH

**Deploy**:

```bash
# Using Viem
npm run deploy:testnet

# Or using Ethers
npm run deploy:testnet:ethers
```

#### Option 2: Using Mock Tokens

Deploy with mock USDC and mETH contracts:

```bash
# Using Viem
npm run deploy:testnet:mock

# Or using Ethers
npm run deploy:testnet:mock:ethers

# Or using Forge
npm run deploy:forge:testnet
```

### Deployment Script Flow

The deployment script will:

1. Deploy mock tokens (if mock mode)
2. Deploy CoreVault
3. Deploy SeniorVault and JuniorVault
4. Deploy EpochManager
5. Deploy SafetyModule
6. Deploy DOORRateOracle
7. Deploy VaultStrategy
8. Initialize all contracts
9. Configure roles and parameters
10. Save deployment addresses to `deployments/testnet/latest.json`

### Monitor Deployment

```bash
# Watch deployment in real-time
tail -f deployments/testnet/latest.log

# Check deployment status
cat deployments/testnet/latest.json
```

### Example Deployment Output

**Latest Mock Deployment (2026-01-15)**:

```json
{
  "network": "mantleTestnet",
  "chainId": 5003,
  "timestamp": "2026-01-15T12:37:29.857Z",
  "deployer": "0xb09b4152d37a05a2f2d73e1f5010014e6afafc39",
  "treasury": "0xb09b4152d37a05a2f2d73e1f5010014e6afafc39",
  "contracts": {
    "MockUSDC": "0xa9fd59bf5009da2d002a474309ca38a8d8686f6a",
    "MockMETH": "0xac8fc1d5593ada635c5569e35534bfab1ab2fedc",
    "SeniorVault": "0x03f4903c3fcf0cb23bee2c11531afb8a1307ce91",
    "JuniorVault": "0x694c667c3b7ba5620c68fe1cc3b308eed26afc6e",
    "CoreVault": "0x8d3ed9a02d3f1e05f68a306037edaf9a54a16105",
    "EpochManager": "0xdc0f912aa970f2a89381985a8e0ea3128e754748",
    "SafetyModule": "0xab5fd152973f5430991df6c5b74a5559ffa0d189",
    "DOORRateOracle": "0xe76e27759b2416ec7c9ddf8ed7a58e61030876a4",
    "VaultStrategy": "0xdd84c599f3b9a12d7f8e583539f11a3e1d9224df",
    "MockYieldStrategy": "0x403e548ec79ade195db7e7abaa0eb203bbaa1db0"
  }
}
```

**Production Deployment (Using Real Tokens)**:

```json
{
  "network": "mantleTestnet",
  "chainId": 5003,
  "deployer": "0xb09b4152D37a05a2f2D73e1f5010014e6aFAFC39",
  "contracts": {
    "SeniorVault": "0x34BC889a143870bBd8538EAe6421cA4c62e84bc3",
    "JuniorVault": "0x8E1A6A3Ba7c5cb4d416Da7Fd376b2BC75227022e",
    "CoreVault": "0x1601Aa4aE97b999cEd4bbaCF0D4B52f29554846F",
    "EpochManager": "0x2956e44668E4026D499D46Ad7eCB1312EA8484aa",
    "SafetyModule": "0xA08fF559C4Fc41FEf01D26744394dD2d2aa74E55",
    "DOORRateOracle": "0x8888F236f9ec2B3aD0c07080ba5Ebc1241F70d71",
    "VaultStrategy": "0x92273a6629A87094E4A2525a7AcDE00eD3f025D3",
    "MockYieldStrategy": "0x0C3701a4d3F95af12Ed830caD9082aF896D92De9"
  },
  "tokens": {
    "USDC": "0x9a54bad93a00bf1232d4e636f5e53055dc0b8238",
    "mETH": "0x4Ade8aAa0143526393EcadA836224EF21aBC6ac6"
  }
}
```

---

## Mainnet Deployment

### Pre-Deployment Checklist

- [ ] **Security Audit Completed**
- [ ] **All tests passing** (`npm run test:forge`)
- [ ] **Testnet deployment successful**
- [ ] **Parameters reviewed and finalized**
- [ ] **Multi-sig wallet setup** (Gnosis Safe)
- [ ] **Emergency procedures documented**
- [ ] **Team training completed**
- [ ] **Insurance coverage arranged** (optional but recommended)

### Mainnet Configuration

**Network Details**:

- Chain ID: 5000
- RPC: https://rpc.mantle.xyz
- Explorer: https://explorer.mantle.xyz

**Token Addresses (Mainnet)**:

```
USDC: TBD
mETH: 0xdEAddEaDdeadDEadDEADDEAddEADDEAddead1111
```

### Deployment Steps

#### 1. Prepare Deployment Account

```bash
# Ensure deployer has sufficient MNT for gas
cast balance $DEPLOYER_ADDRESS --rpc-url $MANTLE_MAINNET_RPC_URL
```

#### 2. Review Deployment Parameters

Edit `scripts/deploy/config/mainnet.json`:

```json
{
  "network": "mantle-mainnet",
  "tokens": {
    "usdc": "0xUSDC_MAINNET_ADDRESS",
    "meth": "0xMETH_MAINNET_ADDRESS"
  },
  "parameters": {
    "seniorFixedRate": 550,
    "protocolFeeRate": 200,
    "minJuniorRatio": 500,
    "epochDuration": 604800,
    "earlyWithdrawalPenalty": 100,
    "maxRateChange": 200,
    "challengePeriod": 86400
  },
  "roles": {
    "admin": "0xMULTISIG_ADDRESS",
    "keeper": "0xKEEPER_ADDRESS",
    "oracle": "0xORACLE_ADDRESS",
    "emergency": "0xEMERGENCY_MULTISIG_ADDRESS",
    "treasury": "0xTREASURY_ADDRESS"
  }
}
```

#### 3. Deploy to Mainnet

```bash
# Dry run first (simulate without broadcasting)
forge script scripts/deploy/DeployDoor.s.sol:DeployDoor \
  --rpc-url $MANTLE_MAINNET_RPC_URL \
  --private-key $PRIVATE_KEY \
  --verify \
  --etherscan-api-key $MANTLESCAN_API_KEY

# If dry run succeeds, deploy for real
forge script scripts/deploy/DeployDoor.s.sol:DeployDoor \
  --rpc-url $MANTLE_MAINNET_RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast \
  --verify \
  --etherscan-api-key $MANTLESCAN_API_KEY \
  --slow
```

**Parameters Explanation**:

- `--broadcast`: Actually submit transactions
- `--verify`: Automatically verify on block explorer
- `--slow`: Wait for transaction confirmations (safer)

#### 4. Transfer Ownership to Multi-Sig

```bash
# Transfer admin role from deployer to Gnosis Safe
cast send $CORE_VAULT_ADDRESS \
  "grantRole(bytes32,address)" \
  $(cast --from-utf8 "DEFAULT_ADMIN_ROLE") \
  $MULTISIG_ADDRESS \
  --rpc-url $MANTLE_MAINNET_RPC_URL \
  --private-key $PRIVATE_KEY

# Renounce deployer's admin role
cast send $CORE_VAULT_ADDRESS \
  "renounceRole(bytes32,address)" \
  $(cast --from-utf8 "DEFAULT_ADMIN_ROLE") \
  $DEPLOYER_ADDRESS \
  --rpc-url $MANTLE_MAINNET_RPC_URL \
  --private-key $PRIVATE_KEY
```

---

## Post-Deployment

### 1. Verify All Contracts

```bash
# Verify CoreVault
npx hardhat verify --network mantle-mainnet $CORE_VAULT_ADDRESS \
  $USDC_ADDRESS $SENIOR_VAULT $JUNIOR_VAULT

# Verify SeniorVault
npx hardhat verify --network mantle-mainnet $SENIOR_VAULT_ADDRESS \
  $CORE_VAULT $USDC_ADDRESS "DOOR Senior Vault" "sDOOR"

# Verify JuniorVault
npx hardhat verify --network mantle-mainnet $JUNIOR_VAULT_ADDRESS \
  $CORE_VAULT $USDC_ADDRESS "DOOR Junior Vault" "jDOOR"

# Continue for other contracts...
```

### 2. Configure Initial Parameters

```bash
# Set Senior Fixed Rate (5.5% = 550 basis points)
cast send $CORE_VAULT_ADDRESS \
  "setSeniorFixedRate(uint256)" 550 \
  --rpc-url $MANTLE_MAINNET_RPC_URL \
  --private-key $PRIVATE_KEY

# Set Protocol Fee Rate (2% = 200 basis points)
cast send $CORE_VAULT_ADDRESS \
  "setProtocolFeeRate(uint256)" 200 \
  --rpc-url $MANTLE_MAINNET_RPC_URL \
  --private-key $PRIVATE_KEY
```

### 3. Initialize Rate Oracle

```bash
# Add first rate source (e.g., Treehouse TESR)
cast send $RATE_ORACLE_ADDRESS \
  "addRateSource(bytes32,uint256,uint256)" \
  $(cast --from-utf8 "TESR") \
  400 \
  100 \
  --rpc-url $MANTLE_MAINNET_RPC_URL \
  --private-key $PRIVATE_KEY
```

### 4. Setup Keepers

```bash
# Grant KEEPER_ROLE to keeper address
cast send $CORE_VAULT_ADDRESS \
  "grantRole(bytes32,address)" \
  $(cast keccak "KEEPER_ROLE") \
  $KEEPER_ADDRESS \
  --rpc-url $MANTLE_MAINNET_RPC_URL \
  --private-key $PRIVATE_KEY
```

### 5. Initial Liquidity Bootstrap

```bash
# Seed Senior Vault with initial liquidity
cast send $SENIOR_VAULT_ADDRESS \
  "deposit(uint256,address)" \
  1000000000 \
  $DEPLOYER_ADDRESS \
  --rpc-url $MANTLE_MAINNET_RPC_URL \
  --private-key $PRIVATE_KEY

# Seed Junior Vault
cast send $JUNIOR_VAULT_ADDRESS \
  "deposit(uint256,address)" \
  250000000 \
  $DEPLOYER_ADDRESS \
  --rpc-url $MANTLE_MAINNET_RPC_URL \
  --private-key $PRIVATE_KEY
```

### 6. Setup Monitoring

Configure monitoring tools:

- **Tenderly**: Contract monitoring and alerting
- **OpenZeppelin Defender**: Automated operations and security
- **Dune Analytics**: On-chain analytics dashboard

---

## Verification

### Manual Verification

#### 1. Check Contract Deployment

```bash
# Verify contracts exist on-chain
cast code $CORE_VAULT_ADDRESS --rpc-url $MANTLE_MAINNET_RPC_URL

# Should return bytecode (not empty)
```

#### 2. Check Initialization

```bash
# Check if CoreVault is initialized
cast call $CORE_VAULT_ADDRESS \
  "initialized()(bool)" \
  --rpc-url $MANTLE_MAINNET_RPC_URL

# Should return "true"
```

#### 3. Verify Roles

```bash
# Check admin role
cast call $CORE_VAULT_ADDRESS \
  "hasRole(bytes32,address)(bool)" \
  $(cast --from-utf8 "DEFAULT_ADMIN_ROLE") \
  $MULTISIG_ADDRESS \
  --rpc-url $MANTLE_MAINNET_RPC_URL

# Should return "true"
```

#### 4. Verify Parameters

```bash
# Check senior fixed rate
cast call $CORE_VAULT_ADDRESS \
  "seniorFixedRate()(uint256)" \
  --rpc-url $MANTLE_MAINNET_RPC_URL

# Should return configured rate (e.g., 550 for 5.5%)
```

### Automated Verification Script

```bash
npm run verify:deployment:mainnet
```

This script will check:

- ✅ All contracts deployed
- ✅ Contracts verified on explorer
- ✅ Roles correctly assigned
- ✅ Parameters correctly set
- ✅ Safety thresholds configured
- ✅ Oracle sources added

---

## Troubleshooting

### Common Issues

#### Issue 1: Transaction Reverted During Deployment

**Symptoms**: `Transaction reverted: execution reverted`

**Solutions**:

```bash
# 1. Check gas price
cast gas-price --rpc-url $RPC_URL

# 2. Increase gas limit in deployment script
export GAS_LIMIT=10000000

# 3. Check deployer balance
cast balance $DEPLOYER_ADDRESS --rpc-url $RPC_URL
```

#### Issue 2: Contract Verification Failed

**Symptoms**: `Error: Unable to verify contract`

**Solutions**:

```bash
# 1. Wait a few minutes and retry
sleep 60
npx hardhat verify --network mantle-mainnet $ADDRESS

# 2. Manually verify on Mantlescan
# - Go to https://explorer.mantle.xyz/address/$ADDRESS
# - Click "Contract" tab → "Verify and Publish"
# - Upload flattened source code

# Flatten contract
forge flatten src/core/CoreVault.sol > CoreVault_flat.sol
```

#### Issue 3: Rate Oracle Not Updating

**Symptoms**: `getDOR()` returns 0

**Solutions**:

```bash
# 1. Check if rate sources added
cast call $RATE_ORACLE_ADDRESS \
  "getActiveSourceCount()(uint256)" \
  --rpc-url $RPC_URL

# 2. Add rate source if missing
cast send $RATE_ORACLE_ADDRESS \
  "addRateSource(bytes32,uint256,uint256)" \
  $SOURCE_ID $RATE $WEIGHT \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY

# 3. Update rate manually
cast send $RATE_ORACLE_ADDRESS \
  "updateRate(bytes32,uint256)" \
  $SOURCE_ID $NEW_RATE \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY
```

#### Issue 4: Safety Module Triggers Immediately

**Symptoms**: Deposits paused right after deployment

**Solutions**:

```bash
# 1. Check junior ratio
cast call $SAFETY_MODULE_ADDRESS \
  "calculateJuniorRatio()(uint256)" \
  --rpc-url $RPC_URL

# 2. Ensure junior deposits before senior
# Deposit to Junior first to establish buffer
cast send $JUNIOR_VAULT_ADDRESS \
  "deposit(uint256,address)" \
  $JUNIOR_AMOUNT $USER \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY

# 3. Adjust safety thresholds temporarily
cast send $SAFETY_MODULE_ADDRESS \
  "setMinJuniorRatio(uint256)" \
  100 \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY
```

### Getting Help

- **Discord**: [discord.gg/doorprotocol](https://discord.gg/doorprotocol)
- **GitHub Issues**: [github.com/door-protocol/contracts/issues](https://github.com/door-protocol/contracts/issues)
- **Email**: andy3638@naver.com

---

## Deployment Checklist

### Pre-Deployment

- [ ] All tests passing (`npm run test:forge`)
- [ ] Gas optimization review completed
- [ ] Security audit completed (for mainnet)
- [ ] Environment variables configured
- [ ] Deployer account funded
- [ ] Multi-sig wallet setup (for mainnet)
- [ ] Parameters finalized and reviewed
- [ ] Deployment script tested on testnet

### During Deployment

- [ ] Deploy all contracts
- [ ] Initialize contracts
- [ ] Configure roles
- [ ] Set parameters
- [ ] Verify contracts on explorer
- [ ] Transfer ownership to multi-sig (mainnet)

### Post-Deployment

- [ ] Verify all contracts on explorer
- [ ] Check initialization status
- [ ] Verify roles assigned correctly
- [ ] Test basic operations (deposit, withdraw)
- [ ] Setup monitoring and alerts
- [ ] Document deployed addresses
- [ ] Announce to community

---

**Deployment Completed!** 🎉

For ongoing operations and maintenance, see the [Operations Guide](./OPERATIONS.md).
