# DOOR Protocol

> **Decentralized Offered Rate Protocol on Mantle Network**
>
> A production-ready DeFi protocol implementing risk-tranched vaults with waterfall distribution.

- 📄 **[One-Paper Pitch](https://github.com/door-protocol/one-paper/blob/main/README.md)**
- 🎥 **[Live Demo](https://youtu.be/JIF715N5R3U)**

---

## What is DOOR Protocol?

DOOR Protocol is a **dual-tranche vault system** that offers investors choice between risk profiles:

- **Senior Vault (sDOOR)**: Fixed-rate returns (~6% APY) with priority protection
- **Junior Vault (jDOOR)**: Leveraged returns (15-30% APY) with higher risk

Junior capital acts as a buffer to protect senior investors, while earning amplified yields in return.

---

## Documentation

- **[Full Documentation](https://github.com/door-protocol/.github/blob/main/profile/FULL_DOCUMENTATION.md)** - Complete project details
- **[Product Overview](https://github.com/door-protocol/.github/blob/main/docs/README.md)** - Features and use cases
- **[Architecture](https://github.com/door-protocol/.github/blob/main/docs/ARCHITECTURE.md)** - Technical specifications
- **[API Reference](https://github.com/door-protocol/.github/blob/main/docs/API_REFERENCE.md)** - Integration guide
- **[Deployment Guide](https://github.com/door-protocol/.github/blob/main/docs/DEPLOYMENT_GUIDE.md)** - Setup instructions

---

## Getting Started

### For Users

1. Visit **[door-protocol-frontend.vercel.app](https://door-protocol-frontend.vercel.app)**
2. Connect your wallet (MetaMask, WalletConnect, etc.)
3. Switch to **Mantle Sepolia Testnet** (Chain ID: 5003)
4. Choose your vault and deposit to start earning

### For Developers

**Smart Contracts**:

```bash
cd contract/
npm install && forge install
forge test                    # Run 142 tests
npm run deploy:testnet        # Deploy to Mantle Sepolia
```

**Frontend**:

```bash
cd frontend/
npm install
cp .env.local.example .env.local
npm run dev                   # Start on http://localhost:3000
```

---

## Deployed Contracts (Mantle Sepolia)

| Contract    | Address                                      |
| ----------- | -------------------------------------------- |
| CoreVault   | `0x6D418348BFfB4196D477DBe2b1082485F5aE5164` |
| SeniorVault | `0x766624E3E59a80Da9801e9b71994cb927eB7F260` |
| JuniorVault | `0x8d1fBEa28CC47959bd94ece489cb1823BeB55075` |
| MockUSDC    | `0xbadbbDb50f5F0455Bf6E4Dd6d4B5ee664D07c109` |
| MockMETH    | `0x374962241A369F1696EF88C10beFe4f40C646592` |

[See all contracts →](https://github.com/door-protocol/contract)

---

## Key Features

### Waterfall Distribution

```
Total Yield → Protocol Fee (1%) → Senior Obligation (Fixed) → Junior (Remainder)
```

### Risk Management

- Dynamic safety thresholds
- Minimum junior capital ratio enforcement
- Emergency pause mechanisms
- Epoch-based withdrawals

### ERC-4626 Standard

Both vaults implement the standard tokenized vault interface for maximum composability.

---

## Repository Structure

```
production/
├── contract/              # Solidity smart contracts
├── frontend/              # Next.js web application
└── .github/               # GitHub profile & documentation
    ├── profile/           # Profile README
    └── docs/              # Technical documentation
```

---

## Technology Stack

**Contracts**

- [Solidity 0.8.26](https://soliditylang.org/)
- [Foundry](https://getfoundry.sh/)
- [OpenZeppelin](https://www.openzeppelin.com/)
- [ERC-4626](https://eips.ethereum.org/EIPS/eip-4626)

**Frontend**

- [Next.js 16](https://nextjs.org/)
- [Viem](https://viem.sh/)
- [Wagmi v2](https://wagmi.sh/)
- [Tailwind CSS](https://tailwindcss.com/)
- [shadcn/ui](https://ui.shadcn.com/)

**Network**

- [Mantle Network](https://www.mantle.xyz/)

---

## Security

- ✅ Internal review completed
- ⏳ External audit pending
- 142 tests passing with comprehensive coverage
- OpenZeppelin libraries for battle-tested security

**⚠️ Disclaimer**: Testnet only. Not audited. Use at your own risk.

---

## Contact

- **Email**: andy3638@naver.com
- **Website**: [door-protocol-frontend.vercel.app](https://door-protocol-frontend.vercel.app)
- **Twitter/X**: [@door_protocol](https://x.com/door_protocol)
- **GitHub**: [github.com/door-protocol](https://github.com/door-protocol)

---

## License

MIT License - see [LICENSE](https://github.com/door-protocol/contract/blob/main/LICENSE) for details.
