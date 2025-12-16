<div align="center">

# SeedPay: Payments Protocol for BitTorrent Networks

## Seeders earn crypto for sharing files. Leechers pay for faster downloads or earn credits by seeding.

</div>

## Abstract

SeedPay is an open payment protocol that enables BitTorrent seeders to earn cryptocurrency for sharing files, while giving leechers two paths for accessing the content: pay seeders directly with stablecoins (i.g. USDC), or earn ratio credits by seeding other torrents.

By extending the BitTorrent Wire Protocol with payment handshakes and blockchain verified proofs, SeedPay solves the free-rider problem without requiring centralized infrastructure or breaking compatibility with existing clients.

The protocol supports micropayments as low as $0.0001/MB with near-instant settlement, backward compatibility with standard BitTorrent, and optional participation, making it suitable for both public and private swarms.

## 1. Motivation

### The Free-Rider Problem in BitTorrent

BitTorrent's success depends on users seeding (uploading) files after downloading them. However, rational actors have no incentive to continue seeding once their download completes—leading to the "free-rider problem."

**Consequences:**

- Popular torrents thrive (many seeders)
- Long-tail content dies (no seeders after initial interest)
- Download speeds degrade as seeder/leecher ratio drops
- Network relies on altruism, which doesn't scale

### Why Existing Solutions Are Insufficient

**Tit-for-tat (built into BitTorrent):**

- Only works during active download sessions
- No incentive to seed after completion
- Easily gamed by "hit-and-run" users

**Private trackers:**

- Enforce ratio requirements (must seed 1:1 or more)
- Require centralized coordination
- High barrier to entry (invite-only)
- Not viable for public/open content

**BitTorrent Token (BTT):**

- Centralized ledger, proprietary implementation
- Limited adoption, high friction
- Didn't integrate with existing clients

### How SeedPay Solves This

SeedPay provides **direct economic incentives** for seeding through two mechanisms:

1. **Micropayments**: Leechers pay seeders in stablecoins
   ($0.0001-$0.001/MB typical)
2. **Ratio credits**: Leechers earn credits by seeding, spend them
   to download from paid seeders

This dual model:

- Incentivizes seeding after download completion
- Works for both crypto-native and traditional users
- No centralized infrastructure required
- Backward compatible with existing clients

## 2. Protocol Overview

### 2.1 Roles

- **Leecher**: Downloads files, pays with crypto or ratio credits
- **Seeder**: Uploads files, earns crypto or ratio credits

### 2.2 Dual Participation Model

In SeedPay, Leecher has two paths for accessing the content from the paid Seeder. Either by directly paying the cryptocurrency or by using its ratio credits earned by seeding other content.

The opt-in nature of payments flow allows well-behaved actors to get access to faster download speeds through paid seeders without paying them.

### 2.3 Design Principles

- Backward compatible with BitTorrent
- Opt-in (No forced payments)
- Blockchain-agnostic (Solana first, extensible)
- Minimal trust required

## 3. Core Protocol Flow

The payment flow consists of four phases:

![SeedPay Protocol Flow](./diagrams/flow.png)

### 3.1 Handshake Phase

The handshake phase extends the existing BitTorrent peer wire handshake (BEP 3) and extension protocol (BEP 10) to advertise SeedPay support and payment terms without breaking compatibility with non-SeedPay clients.

#### 3.1.1 Standard BitTorrent Handshake

When two peers connect over TCP, they first perform the normal BitTorrent handshake as defined in BEP 3. The handshake includes:

- The protocol string `"BitTorrent protocol"`
- An 8-byte `reserved` field used for feature flags
- The torrent's `info hash`
- Each peer's 20 byte `peer_id`

SeedPay doesn't modify this message. Instead, it relies on the extension protocol flag in the `reserved field`. If both peers set the BEP 10 "extended messaging" bit in the handshake, they proceed with extended handshake.

#### 3.1.2 Extended Handhshake (BEP 10)

Immediately after the standard handshake, peers that support the extension protocol exchange an extended handshake(BitTorrent message ID `20`, extended ID `0`). The payload of this message is a bencoded dictionary that:

- Advertises which extension the peer supports via `m` map (extension name -> numeric ID)
- May include additional, extension specific data

SeedPay defines a new extension named "seedpay". A Seeder that supports paid seeding MUST include an entry for "seedpay" in the m dictionary:

```json
{
  "m": {
    "seedpay": 5,
    "ut_metadata": 2
  },
  "v": "SeedPayClient 0.1",
  "seedpay": {
    "wallet": "DYw8jCN...",
    "price_per_mb": 0.0001,
    "accepts_ratio": 1,
    "min_prepayment": 0.01,
    "chain": "solana"
  }
}
```

- The numeric value `5` is the local extension ID the Seeder will use when sending and receiving SeedPay messages on this connection. The exact ID is implementation‑defined and only needs to be consistent for this peer pair.
- The nested `seedpay` dictionary contains the Seeder's payment capabilities:
  - `wallet`: seeder’s on‑chain wallet address for receiving payments
  - `price_per_mb`: quoted price in USDC per MB
  - `accepts_ratio`: whether this peer accepts ratio credits instead of payments
  - `min_prepayment`: min. amount required to start a paid session
  - `chain`: identifier of the settlement chain (e.g. `"solana"`)

A Leecher that supports SeedPay should also include `seedpay` in its `m` map, but it's not required to send payment terms.

```json
{
  "m": {
    "seedpay": 3
  },
  "v": "SeedPayClient 0.1",
  "peer_id": "LEECHR-ABC123..."
}
```

After both extended handshakes are exchanged, each peer:

- checks whether the remote `m` contains `"seedpay"`
- records the remote extension id for `"seedpay"`
- parses the remote `seedpay` object (if present) to learn the counterparty's wallet, pricing, and ratio policy.

If either side’s extended handshake does not include `"seedpay"` in `m`, the connection continues as a normal BitTorrent session without payments.
