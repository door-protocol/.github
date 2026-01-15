# DOOR Protocol - Complete Project Documentation

> **Decentralized Offered Rate Protocol on Mantle Network**
> A production-ready structured DeFi product implementing waterfall distribution for risk-adjusted yield generation.

For a comprehensive overview of our vision, market opportunity, and roadmap, please refer to our **[One-Pager Pitch](https://github.com/door-protocol/one-paper/blob/main/README.md)**.

---

## 🎯 Project Overview

DOOR Protocol is a sophisticated dual-tranche vault system deployed on Mantle Network that provides:

- **Senior Vault (sDOOR)**: Fixed-rate returns (5-8% APY) with priority claim on yields
- **Junior Vault (jDOOR)**: Leveraged returns (15-30% APY) with first-loss protection for seniors
- **Waterfall Distribution**: Automated yield allocation following strict priority ordering
- **Dynamic Safety Module**: Real-time risk management with configurable thresholds
- **Oracle-Based Rates**: Decentralized rate calculation from multiple sources

### Key Innovation

DOOR Protocol implements a **risk-tranching mechanism** that allows investors to choose their risk/return profile while maintaining capital efficiency through a unified yield strategy. Junior capital acts as a buffer to protect senior investors, while receiving amplified returns in exchange.

---

## 📁 Repository Structure

This monorepo contains all components of the DOOR Protocol ecosystem:

```
production/
├── contract/              # Smart contracts (Solidity)
│   ├── src/              # Contract source code
│   │   ├── core/         # CoreVault - central coordinator
│   │   ├── tranches/     # SeniorVault & JuniorVault (ERC-4626)
│   │   ├── epoch/        # EpochManager - withdrawal windows
│   │   ├── safety/       # SafetyModule - risk management
│   │   ├── oracle/       # DOORRateOracle - rate calculation
│   │   ├── strategy/     # VaultStrategy - yield generation
│   │   └── libraries/    # Math and utility libraries
│   ├── test/             # Foundry tests (142 tests, 100% pass)
│   ├── scripts/          # Deployment and utility scripts
│   └── README.md         # Contract-specific documentation
│
├── frontend/             # Next.js web application
│   ├── src/
│   │   ├── app/          # Next.js app router pages
│   │   ├── components/   # React components
│   │   ├── lib/          # Utilities and configurations
│   │   └── hooks/        # Custom React hooks
│   └── README.md         # Frontend-specific documentation
│
├── .github/              # Project-wide documentation
│   ├── docs/             # Comprehensive technical documentation
│   │   ├── README.md             # Product overview
│   │   ├── ARCHITECTURE.md       # System design & technical details
│   │   ├── API_REFERENCE.md      # Contract interfaces & integration
│   │   └── DEPLOYMENT_GUIDE.md   # Deployment instructions
│   └── README.md         # This file - project navigation
│
└── md/                   # Development documentation
    ├── hackathon/        # Hackathon submission materials
    └── dev/              # Development notes
```

---

## 🚀 Quick Start

### For Users

**Web Application**: [https://door-protocol-frontend.vercel.app](https://door-protocol-frontend.vercel.app)

1. Connect your wallet (MetaMask, WalletConnect, etc.)
2. Switch to Mantle Sepolia Testnet (Chain ID: 5003)
3. Choose your vault:
   - **Senior Vault**: Low risk, fixed returns (~8% APY)
   - **Junior Vault**: High risk, leveraged returns (~25% APY)
4. Deposit USDC and start earning

### For Developers

#### Smart Contracts

```bash
# Navigate to contracts directory
cd contract/

# Install dependencies
npm install
forge install

# Run tests
forge test

# Deploy to testnet
npm run deploy:testnet

# See full documentation
cat README.md
```

**Key Contract Addresses (Mantle Sepolia)**:

- CoreVault: `0x1601Aa4aE97b999cEd4bbaCF0D4B52f29554846F`
- SeniorVault: `0x34BC889a143870bBd8538EAe6421cA4c62e84bc3`
- JuniorVault: `0x8E1A6A3Ba7c5cb4d416Da7Fd376b2BC75227022e`

[See all addresses →](./docs/DEPLOYMENT_GUIDE.md#deployment-output)

#### Frontend Application

```bash
# Navigate to frontend directory
cd frontend/

# Install dependencies
npm install

# Set up environment variables
cp .env.local.example .env.local
# Edit .env.local with your configuration

# Run development server
npm run dev

# Open http://localhost:3000

# See full documentation
cat README.md
```

---

## 📚 Documentation

### Core Documentation

Located in [`.github/docs/`](./docs/):

| Document                                              | Description                                                    | Audience    |
| ----------------------------------------------------- | -------------------------------------------------------------- | ----------- |
| [**README.md**](./docs/README.md)                     | Product overview, features, and use cases                      | Everyone    |
| [**ARCHITECTURE.md**](./docs/ARCHITECTURE.md)         | System design, contract interactions, technical specifications | Developers  |
| [**API_REFERENCE.md**](./docs/API_REFERENCE.md)       | Contract interfaces, integration examples, event reference     | Integrators |
| [**DEPLOYMENT_GUIDE.md**](./docs/DEPLOYMENT_GUIDE.md) | Deployment instructions, network configs, troubleshooting      | DevOps      |

### Component Documentation

- **Smart Contracts**: [`/contract/README.md`](../contract/README.md)

  - Contract architecture
  - Testing guide
  - Development workflow
  - Security considerations

- **Frontend**: [`/frontend/README.md`](../frontend/README.md)
  - Application structure
  - Component library
  - State management
  - Deployment on Vercel

---

## 🏗️ Architecture Overview

### System Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     DOOR Protocol Flow                       │
└─────────────────────────────────────────────────────────────┘

Users
  │
  ├─► SeniorVault (Fixed Rate)
  │     └─► deposit() → registerSeniorDeposit()
  │
  └─► JuniorVault (Leveraged)
        └─► deposit() → registerJuniorDeposit()
                │
                ▼
          CoreVault (Coordinator)
                │
                ├─► depositToStrategy()
                │         │
                │         ▼
                │   VaultStrategy (mETH Staking)
                │         │
                │         └─► Mantle Network Yield
                │
                ├─► harvest() [Every 24h]
                │     │
                │     └─► Waterfall Distribution:
                │           1. Protocol Fee (2%)
                │           2. Senior Obligation (Fixed Rate)
                │           3. Remaining → Junior
                │
                └─► SafetyModule.performHealthCheck()
                      └─► Enforce minimum junior ratio
```

### Waterfall Distribution

```
Total Yield (12% APY on $100k)
    │
    ├─► Protocol Fee (2%)
    │   └─► $2,400 → Treasury
    │
    ├─► Senior Yield (8% on $80k)
    │   └─► $6,400 → SeniorVault
    │
    └─► Junior Yield (Remainder)
        └─► $3,200 → JuniorVault
            (16% APY on $20k Junior Capital)
```

[Learn more about architecture →](./docs/ARCHITECTURE.md)

---

## 🔐 Security

### Audit Status

- ✅ **Internal Review**: Completed
- ⏳ **External Audit**: Pending (recommended before mainnet)
- 📋 **Bug Bounty**: To be announced

### Security Features

1. **Access Control**: OpenZeppelin role-based permissions
2. **Reentrancy Guards**: All external calls protected
3. **Rate Limiting**: Max 2% rate changes, 24h challenge period
4. **Emergency Controls**: Pause mechanisms and emergency withdrawals
5. **Testing**: 142 tests, 100% pass rate, comprehensive coverage

### Known Limitations

- Protocol has not undergone external security audit
- Experimental software - use at your own risk
- Strategy risk: mETH staking carries smart contract and market risks
- Testnet only - not production ready

[See security details →](./docs/ARCHITECTURE.md#security-considerations)

---

## 🧪 Testing

### Smart Contracts

```bash
cd contract/

# Run all tests
forge test

# Run with verbosity
forge test -vvv

# Run specific test
forge test --match-test testWaterfallDistribution

# Generate coverage report
forge coverage
```

**Test Results**:

```
Test result: ok. 142 passed; 0 failed; 0 skipped; finished in 2.31s
Ran 8 test suites: 142 tests passed, 0 failed, 0 skipped
```

### Frontend

```bash
cd frontend/

# Run type checking
npm run type-check

# Run linting
npm run lint

# Build for production
npm run build
```

---

## 🌐 Deployment

### Networks

| Network            | Status      | Chain ID | RPC URL                        |
| ------------------ | ----------- | -------- | ------------------------------ |
| **Mantle Sepolia** | ✅ Deployed | 5003     | https://rpc.sepolia.mantle.xyz |
| **Mantle Mainnet** | ⏳ Planned  | 5000     | https://rpc.mantle.xyz         |

### Deployed Contracts (Testnet)

See full deployment addresses in [DEPLOYMENT_GUIDE.md](./docs/DEPLOYMENT_GUIDE.md#deployment-output)

### Web Application

- **Production**: [https://door-protocol-frontend.vercel.app](https://door-protocol-frontend.vercel.app)
- **Status**: ✅ Live on Mantle Sepolia Testnet

---

## 🛠️ Technology Stack

### Smart Contracts

- **Language**: Solidity 0.8.26
- **Framework**: Foundry + Hardhat
- **Standards**: ERC-4626 (Tokenized Vaults)
- **Libraries**: OpenZeppelin Contracts v5.0.0
- **Network**: Mantle Network (L2 Ethereum)

### Frontend

- **Framework**: Next.js 15 (App Router)
- **Blockchain**: Viem + Wagmi v2
- **Styling**: Tailwind CSS + shadcn/ui
- **State**: React Context + Hooks
- **Deployment**: Vercel

---

## 👥 Team & Contact

### Developers

- **Lead Developer**: andy3638@naver.com
- **GitHub**: [@door-protocol](https://github.com/door-protocol)

### Community

- **Website**: [https://door-protocol-frontend.vercel.app](https://door-protocol-frontend.vercel.app)
- **Twitter/X**: [@door_protocol](https://x.com/door_protocol)
- **GitHub**: [github.com/door-protocol](https://github.com/door-protocol)

### Support

- **Issues**: [GitHub Issues](https://github.com/door-protocol/contracts/issues)
- **Email**: andy3638@naver.com

---

## 🗺️ Roadmap

### Phase 1: Core Protocol (✅ Completed)

- ✅ Dual-tranche vault system
- ✅ Waterfall distribution mechanism
- ✅ Dynamic safety module
- ✅ Oracle-based rate system
- ✅ Epoch management
- ✅ Comprehensive testing (142 tests)
- ✅ Frontend application
- ✅ Testnet deployment

### Phase 2: Production Launch (Q2 2026)

- 🔄 External security audit
- 🔄 Mainnet deployment on Mantle
- 🔄 Initial liquidity bootstrap
- 🔄 Marketing and user acquisition

### Phase 3: Strategy Expansion (Q3 2026)

- 📋 Multi-asset support (WETH, wMNT)
- 📋 Additional yield strategies (Curve, Aave)
- 📋 Cross-protocol yield aggregation
- 📋 Automated rebalancing

### Phase 4: Ecosystem Growth (Q4 2026)

- 📋 Governance token launch
- 📋 DAO transition
- 📋 Liquidity mining programs
- 📋 Third-party integrations

---

## 📊 Use Cases

### Senior Vault - Conservative Investors

**Profile**: Risk-averse investors seeking stable returns

**Features**:

- Fixed APY (5-8%, based on DOR + 1%)
- Priority claim on yields
- Protected by Junior capital buffer
- Suitable for: Long-term holders, institutions, DAOs

**Example**:

```
Deposit: 10,000 USDC
APY: 8%
Annual Yield: 800 USDC
Risk: Low (Junior buffer protection)
```

### Junior Vault - Aggressive Investors

**Profile**: Risk-seeking investors pursuing high returns

**Features**:

- Leveraged APY (15-30%, depends on strategy yield)
- Receives all excess yield
- First-loss position
- Suitable for: Active traders, yield farmers

**Example**:

```
Scenario: $100k total, $20k junior (20% ratio)
Strategy Yield: 12% APY ($12k/year)
Senior Obligation: 8% on $80k = $6.4k
Junior Gets: $12k - $6.4k = $5.6k
Junior APY: 28%
```

---

## 🤝 Contributing

We welcome contributions! Here's how you can help:

### Smart Contracts

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/AmazingFeature`)
3. Write tests for your changes
4. Ensure all tests pass (`forge test`)
5. Submit a pull request

### Frontend

1. Fork the repository
2. Create a feature branch
3. Test your changes locally
4. Run linting (`npm run lint`)
5. Submit a pull request

### Documentation

Found a typo or want to improve documentation?

1. Edit the relevant `.md` file
2. Submit a pull request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](../contract/LICENSE) file for details.

---

## ⚠️ Disclaimer

**IMPORTANT**: DOOR Protocol is experimental software currently deployed on testnet only. While thoroughly tested, smart contracts carry inherent risks. The protocol has not yet undergone external security audits.

**Use at your own risk. This is not financial advice.**

- Smart contracts are immutable once deployed
- Strategy risks include smart contract vulnerabilities and market volatility
- Loss of funds is possible, especially for Junior vault depositors
- Always start with small amounts to test the system
- Never invest more than you can afford to lose

---

## 🎓 Learning Resources

### For Newcomers

1. Start with [Product Overview](./docs/README.md)
2. Understand [Waterfall Distribution](./docs/ARCHITECTURE.md#waterfall-distribution-mechanism)
3. Explore [Use Cases](./docs/README.md#use-cases)
4. Try the [Web App](https://door-protocol-frontend.vercel.app)

### For Developers

1. Read [Architecture Documentation](./docs/ARCHITECTURE.md)
2. Study [API Reference](./docs/API_REFERENCE.md)
3. Follow [Deployment Guide](./docs/DEPLOYMENT_GUIDE.md)
4. Review [Contract Code](../contract/src/)

### For Integrators

1. Check [API Reference](./docs/API_REFERENCE.md)
2. Review [Integration Examples](./docs/API_REFERENCE.md#integration-examples)
3. Test on [Mantle Sepolia Testnet](./docs/DEPLOYMENT_GUIDE.md#testnet-deployment)

---

## 🌟 Acknowledgments

Built with:

- **Mantle Network**: L2 infrastructure and mETH integration
- **OpenZeppelin**: Secure smart contract libraries
- **Foundry**: Development and testing framework
- **Vercel**: Frontend hosting
- **Viem & Wagmi**: Web3 frontend libraries

Special thanks to the Mantle Network team for their support and the amazing L2 infrastructure.

---

For the latest updates, follow us on [X (Twitter)](https://x.com/door_protocol) or visit our [GitHub](https://github.com/door-protocol).
