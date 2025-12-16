# SeedPay Protocol

**Seeders earn crypto for sharing files. Leechers pay for faster downloads or earn credits by seeding.**

SeedPay is an open payment protocol that enables BitTorrent seeders to earn cryptocurrency for sharing files, while giving leechers two paths for accessing content: pay seeders directly with stablecoins (e.g. USDC), or earn ratio credits by seeding other torrents.

## 🎯 Project Goals

SeedPay aims to solve the **free-rider problem** in BitTorrent networks by providing direct economic incentives for seeding. The protocol:

- ✅ Incentivizes seeding after download completion
- ✅ Works for both crypto-native and traditional users
- ✅ Requires no centralized infrastructure
- ✅ Maintains backward compatibility with existing BitTorrent clients
- ✅ Supports micropayments as low as $0.0001/MB

## 📋 Current Status

**Early Draft - Seeking Feedback**

This protocol specification is in active development. We're seeking feedback from:

- BitTorrent protocol experts
- P2P/decentralized tech communities
- Solana developers (initial blockchain implementation)
- Crypto payment protocol builders
- Open source contributors

The core protocol flow (handshake, payment submission, verification, data transfer) is specified, but several sections are still in draft form and marked as "Draft/TBD" in the specification.

## 📖 Documentation

- **[PROTOCOL.md](./PROTOCOL.md)** - Complete protocol specification
- **[CONTRIBUTING.md](./CONTRIBUTING.md)** - How to contribute and provide feedback
- **[diagrams/flow.png](./diagrams/flow.png)** - Visual protocol flow diagram

## 🚀 Quick Start

1. **Read the specification**: Start with [PROTOCOL.md](./PROTOCOL.md) to understand the protocol design
2. **Review the flow**: Check out the [protocol flow diagram](./diagrams/flow.png)
3. **Provide feedback**: Open an issue or discussion on GitHub
4. **Contribute**: See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines

## 🔧 How It Works

SeedPay extends the BitTorrent Wire Protocol (BEP 10) to add payment capabilities:

1. **Handshake**: Peers advertise SeedPay support and payment terms
2. **Payment Submission**: Leechers create on-chain transactions to prepay for downloads
3. **Verification**: Seeders verify payments on-chain before serving data
4. **Data Transfer**: Standard BitTorrent piece exchange with payment accounting

The protocol is opt-in and backward compatible—peers without SeedPay support continue to work normally.

## 🤝 Contributing

We welcome contributions! Please see [CONTRIBUTING.md](./CONTRIBUTING.md) for:

- How to provide feedback
- How to report issues
- How to propose changes
- Code of conduct expectations

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details.

## ⚖️ Legal Notice

**This is a protocol specification only.**

Implementation and use of this protocol may have legal implications depending on jurisdiction and content being shared. Users and implementers are responsible for ensuring compliance with applicable laws and regulations, including but not limited to:

- Copyright and intellectual property laws
- Financial regulations (depending on jurisdiction)
- Anti-money laundering (AML) and know-your-customer (KYC) requirements
- Content distribution and sharing regulations

The authors and contributors of this specification assume no liability for how this protocol is implemented or used.

## 🔗 Links

- **GitHub**: [GitHub Repo](https://github.com/NiravJoshi33/seedpay)
- **Issues**: [GitHub Issues](https://github.com/NiravJoshi33/seedpay/issues)
- **Discussions**: [GitHub Discussions](https://github.com/NiravJoshi33/seedpay/discussions/)

## 🙏 Acknowledgments

SeedPay builds on the BitTorrent protocol (BEP 3, BEP 10) and the broader P2P and cryptocurrency communities. We're grateful for the foundational work that makes this possible.

---

**Status**: Early Draft | **License**: MIT | **Contributions**: Welcome
