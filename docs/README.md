# DOOR Protocol - Production Documentation

**Decentralized Offered Rate Protocol on Mantle Network**

## Overview

DOOR Protocol is a production-ready structured DeFi product implementing a waterfall distribution mechanism for risk-adjusted yield generation on Mantle Network. The protocol provides sophisticated capital allocation through a dual-tranche system that separates risk and return profiles for different investor preferences.

## Key Features

### 1. Dual-Tranche System

- **Senior Vault (sDOOR)**: Fixed-rate returns with priority claim on yields
- **Junior Vault (jDOOR)**: Leveraged returns with first-loss protection for seniors

### 2. Waterfall Distribution

Automated yield distribution following a strict priority order:

1. Protocol fees (2%)
2. Senior fixed obligations (based on DOR + 1%)
3. Remaining yield to Junior vault

### 3. Dynamic Safety Module

- Real-time risk management with configurable thresholds
- Automatic deposit pausing when safety levels are breached
- Dynamic senior deposit caps based on junior capital availability

### 4. Oracle-Based Rate System (DOR)

- Decentralized Offered Rate calculation from multiple sources
- Challenge mechanism for rate updates (24-hour period)
- Adaptive APY based on market conditions

### 5. Epoch Management

- 7-day withdrawal windows
- Early withdrawal penalties (configurable 1-5%)
- Fair withdrawal queue processing

### 6. Flexible Yield Strategies

- Pluggable strategy architecture
- Current support for mETH staking
- Future expansion for multi-protocol integrations

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     DOOR Protocol Architecture               │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐           ┌─────────────┐                 │
│  │ SeniorVault │           │ JuniorVault │                 │
│  │  (ERC-4626) │           │  (ERC-4626) │                 │
│  └──────┬──────┘           └──────┬──────┘                 │
│         │                          │                         │
│         └──────────┬───────────────┘                        │
│                    ▼                                         │
│            ┌───────────────┐                                │
│            │   CoreVault   │                                │
│            │ (Coordinator) │                                │
│            └───────┬───────┘                                │
│                    │                                         │
│    ┌───────────────┼───────────────┐                       │
│    ▼               ▼               ▼                        │
│ ┌────────┐  ┌──────────┐  ┌──────────────┐               │
│ │ Epoch  │  │  Safety  │  │ Rate Oracle  │               │
│ │Manager │  │  Module  │  │    (DOR)     │               │
│ └────────┘  └──────────┘  └──────────────┘               │
│                    │                                         │
│                    ▼                                         │
│            ┌───────────────┐                                │
│            │Yield Strategy │                                │
│            │   (mETH, LP)  │                                │
│            └───────────────┘                                │
└─────────────────────────────────────────────────────────────┘
```

## Smart Contracts

### Core Components

| Contract           | Purpose               | Key Functions                                                  |
| ------------------ | --------------------- | -------------------------------------------------------------- |
| **CoreVault**      | Central coordinator   | `harvest()`, `depositToStrategy()`, `syncSeniorRate()`         |
| **SeniorVault**    | Fixed-rate tranche    | `deposit()`, `withdraw()`, `addYield()`                        |
| **JuniorVault**    | Leveraged tranche     | `deposit()`, `withdraw()`, `slashPrincipal()`                  |
| **EpochManager**   | Withdrawal management | `requestWithdrawal()`, `processWithdrawal()`, `advanceEpoch()` |
| **SafetyModule**   | Risk management       | `performHealthCheck()`, `updateSafetyLevel()`                  |
| **DOORRateOracle** | APY calculation       | `updateRate()`, `getDOR()`, `getSeniorTargetAPY()`             |
| **VaultStrategy**  | Yield generation      | `allocate()`, `deallocate()`, `harvest()`                      |

### Contract Addresses

See [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md) for deployed contract addresses on Mantle Network.

## Technical Specifications

### Technology Stack

- **Smart Contracts**: Solidity 0.8.26
- **Framework**: Foundry + Hardhat
- **Standards**: ERC-4626 (Tokenized Vaults)
- **Network**: Mantle Network (L2)
- **Dependencies**: OpenZeppelin Contracts v5.0.0

### Key Parameters

| Parameter                | Default Value | Configurable |
| ------------------------ | ------------- | ------------ |
| Min Junior Ratio         | 5%            | Yes          |
| Senior Premium           | 1% above DOR  | Yes          |
| Protocol Fee             | 2%            | Yes          |
| Epoch Duration           | 7 days        | Yes          |
| Early Withdrawal Penalty | 1%            | Yes          |
| Max Rate Change          | 2% per update | Yes          |
| Challenge Period         | 24 hours      | Yes          |

## Security

### Testing

- 142 tests passed (100% pass rate)
- 8 test suites covering all core functionality
- Fuzz testing with 256 runs per test
- Coverage for edge cases and revert conditions

### Security Features

1. **Access Control**: Role-based permissions (OpenZeppelin)
2. **Reentrancy Guards**: All external calls protected
3. **Integer Overflow**: Solidity 0.8.26 built-in protection
4. **Emergency Controls**: Pause mechanisms and emergency withdrawals
5. **Rate Limiting**: Max rate changes and challenge periods
6. **Input Validation**: Comprehensive parameter checks

### Audit Status

- Internal security review completed
- External audit: Pending (recommended before mainnet launch)
- Bug bounty program: To be announced

## Getting Started

### For Developers

See [ARCHITECTURE.md](./ARCHITECTURE.md) for detailed technical documentation.

### For Integrators

See [API_REFERENCE.md](./API_REFERENCE.md) for contract interfaces and integration examples.

### For Deployment

See [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md) for deployment instructions and network configurations.

## Use Cases

### Senior Vault (Conservative Investors)

- **Profile**: Risk-averse, seeking stable returns
- **Expected APY**: 5-8% (based on DOR + 1%)
- **Protection**: Junior capital acts as first-loss buffer
- **Suitable For**: Long-term holders, institutional investors, DAOs

**Example**:

```
Deposit: 10,000 USDC
APY: 8% (DOR 7% + 1% premium)
Annual Yield: 800 USDC
Risk: Low (protected by junior capital)
```

### Junior Vault (Aggressive Investors)

- **Profile**: Risk-seeking, pursuing high returns
- **Expected APY**: 15-30% (depending on total strategy yield)
- **Risk**: First-loss position, potential principal loss
- **Suitable For**: Active traders, yield farmers, risk-takers

**Example**:

```
Scenario: $100k total, $20k junior (20% ratio)
Strategy Yield: 12% APY ($12k/year)
Senior Obligation: 8% on $80k = $6.4k
Junior Gets: $12k - $6.4k = $5.6k
Junior APY: $5.6k / $20k = 28%
```

## Yield Distribution Flow

```
1. VaultStrategy generates yield from mETH staking + DeFi
                    │
                    ▼
2. CoreVault.harvest() collects yields
                    │
                    ├─► Protocol Fee (2%) → Treasury
                    │
                    ▼
3. Calculate Senior Obligation (Principal × Rate × Time)
                    │
                    ├─► Senior Gets: Full obligation amount
                    │
                    ▼
4. Remaining Yield → Junior Vault
                    │
                    └─► Junior Gets: All excess yield
```

## Safety Module

The Safety Module provides real-time risk management based on junior capital ratio:

| Junior Ratio | Safety Level | Actions             |
| ------------ | ------------ | ------------------- |
| ≥ 15%        | HEALTHY      | Normal operations   |
| 10-15%       | WARNING      | Monitoring mode     |
| 5-10%        | DANGER       | Restricted deposits |
| < 5%         | CRITICAL     | Emergency mode      |

**Automatic Responses**:

- **WARNING**: Alert keepers, monitor closely
- **DANGER**: Pause senior deposits, reduce senior rate
- **CRITICAL**: Pause all deposits, withdrawal-only mode

## Governance

### Roles

1. **DEFAULT_ADMIN_ROLE**: Protocol owner, can grant/revoke roles
2. **KEEPER_ROLE**: Automated operations (harvest, health checks, epoch advancement)
3. **ORACLE_ROLE**: Rate oracle updates
4. **EMERGENCY_ROLE**: Emergency pause and recovery
5. **STRATEGY_ROLE**: Strategy management

### Multi-Signature Requirements

Critical operations require multi-signature approval:

- Protocol parameter updates (3/5)
- Treasury address changes (3/5)
- Emergency pause (2/5)
- Rate oracle updates with >2% change (24-hour timelock + challenge)

## Roadmap

### Phase 1: Core Protocol (Completed)

- ✅ Dual-tranche vault system
- ✅ Waterfall distribution
- ✅ Dynamic safety module
- ✅ Oracle-based rates
- ✅ Epoch management
- ✅ Comprehensive testing

### Phase 2: Production Launch (Q2 2026)

- 🔄 External security audit
- 🔄 Mainnet deployment on Mantle
- 🔄 Frontend dApp launch
- 🔄 Initial liquidity bootstrap

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

## Resources

### Documentation

- [Architecture](./ARCHITECTURE.md) - Detailed technical design
- [API Reference](./API_REFERENCE.md) - Contract interfaces and examples
- [Deployment Guide](./DEPLOYMENT_GUIDE.md) - Deployment instructions

### Links

- **Website**: [https://door-protocol-frontend.vercel.app](https://door-protocol-frontend.vercel.app)
- **Frontend Repository**: [GitHub - Frontend](https://github.com/door-protocol/frontend)
- **Contract Repository**: [GitHub - Contracts](https://github.com/door-protocol/contracts)
- **X (Twitter)**: [@door_protocol](https://x.com/door_protocol)

### Contact

- **Email**: andy3638@naver.com
- **GitHub**: [@door-protocol](https://github.com/door-protocol)

## License

This project is licensed under the MIT License - see the [LICENSE](../../contract/LICENSE) file for details.

## Disclaimer

DOOR Protocol is experimental software. While thoroughly tested, smart contracts carry inherent risks. The protocol has not yet undergone external security audits. Use at your own risk. This documentation does not constitute financial advice.
