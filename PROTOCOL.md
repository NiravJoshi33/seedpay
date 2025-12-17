<div align="center">

# SeedPay: Payments Protocol for BitTorrent Networks

## Seeders earn crypto for sharing files. Leechers pay for faster downloads or earn credits by seeding.

</div>

> ⚠️ **Status: Pre-Alpha / Request For Comments (RFC)**
>
> This specification is a **v0.1 draft** released to gather architectural feedback.
> Based on initial community review, we are actively refactoring three core components for the v0.2 spec:
>
> 1. **Privacy (Anti-Doxxing)**: We are replacing raw PeerIDs in on-chain memos with **Ephemeral Session Keys**. This ensures that public observers cannot link a user's IP address (swarm activity) to their permanent wallet history.
> 2. **Trust Model**: We are moving from _Atomic Prepayment_ to **Unidirectional State Channels** (or probabilistic micropayments). This solves the "Fair Exchange" problem (preventing seeders from taking funds without sending data) and reduces RPC latency.
> 3. **Sybil Resistance**: The "Ratio Credits" system is being redesigned to prevent local Sybil farming, likely moving toward a "Web of Trust" or "Pay-to-Play" closed loop.
>
> **Current implementation is for research only. Contributions on these specific issues are welcome.**
>
> See [CONTRIBUTING.md](./CONTRIBUTING.md) for how to provide feedback.

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

#### 3.1.3 SeedPay Capability Detection

From the Leecher's perspective, the handshake phase answers two questions:

1. Does this peer support SeedPay?

   - Yes, if `"seedpay"` is present in remote `m` map

2. What options are available?
   - If `seedpay.price_per_mb` is set → peer supports paid seeding.
   - If `seedpay.accepts_ratio` is true → peer will also accept ratio credits.

Based on this information the Leechers classifies the peer as:

- Free-only (No `seedpay` entry)
- Paid seeder (SeedPay with pricing, no ratio)
- Hybrid seeder (SeedPay with both pricing and ratio support)

Only after this handshake phase completes does the protocol move into the Payment Submission Phase, where Leechers create on-chain transactions and send payment proofs to seeders.

### 3.2 Payment Submission

Once the handshake phase is complete and Leecher has discovered a Seeder that advertises SeedPay support and pricing, next step is to prepay for a portion of the download. In SeedPay V1, payments are made via direct on-chains initiated by the Leecher, with the transaction bound to specific peer connection via a memo.

#### 3.2.1 Pricing and Session Setup

From the extended handshake, the Leecher learns the Seeder's terms:

- `wallet`: on-chain wallet address that receives the funds
- `price_per_mb`: quoted price in USDC per MB
- `min_prepayment`: min. amount required to open a session
- `chain`: settlement chain identifier (e.g. `"solana"`)

The Leecher may also apply local policy, such as:

- Max. price per MB it is willing to pay
- Max. total spend per torrent per session
- Preference of using ratio credits instead of payment when available

If the Seeder’s advertised terms are acceptable, the Leecher computes an initial prepayment amount large enough to cover some target amount of data (for example, 50–200 MB), but at least `min_prepayment`.

#### 3.2.2 Creating the On-Chain Payment

To start paid session, Leecher creates and submits a token transfer transaction on the configured chain. The transaction MUST:

- Transfer at least `min_prepayment` (and typically `price_per_mb \* target_mb`) of the agreed token (e.g. USDC)
- Use the Seeder’s `wallet` as the recipient
- Include a memo that cryptographically binds the payment to this specific SeedPay peer connection

SeedPay defines the memo payload as a JSON object serialized to UTF‑8:

```json
{
  "protocol": "seedpay",
  "version": "1.0",
  "from_peer_id": "<leecher-peer-id>",
  "to_peer_id": "<seeder-peer-id>",
  "nonce": 1702700000000
}
```

- `protocol` and `version` identify the memo as a SeedPay payment
- `from_peer_id` is the Leecher’s BitTorrent `peer_id` from the wire handshake
- `to_peer_id` is the Seeder’s `peer_id` from the wire handshake
- `nonce` is a timestamp or random value used to ensure freshness and prevent replay

The Leecher signs and submits the transaction directly to blockchain. Once the network accepts the transaction and returns the transaction signature, the Leecher should wait until the transaction reaches at least confirmed status before proceeding.

#### 3.2.3 Sending the Payment Proof to the Seeder

After the transaction is confirmed on‑chain, the Leecher sends a payment proof to the Seeder over the SeedPay extension channel. This is the first SeedPay data message (as opposed to handshake metadata).

The `payment_proof` message has the following JSON structure (sent as the payload of an extended message using the negotiated SeedPay extension ID):

```json
{
  "type": "payment_proof",
  "tx_signature": "<base58-or-hex-signature>",
  "amount": 0.01,
  "from_peer_id": "<leecher-peer-id>",
  "to_peer_id": "<seeder-peer-id>",
  "timestamp": 1702700000000
}
```

- `tx_signature`: the blockchain transaction signature of the prepayment
- `amount`: the amount the Leecher believes it has paid (for accounting and UI)
- `from_peer_id` and `to_peer_id`: must match the values embedded in the on‑chain memo
- `timestamp`: time at which the proof was created (used for freshness checks)

A Seeder that receives a `payment_proof` MUST NOT start sending pieces yet. Instead, it proceeds to the Verification Phase, where it independently validates the payment on‑chain before unchoking the Leecher and opening a paid session.

### 3.3 Verification Phase

In the verification phase, the Seeder independently checks the on‑chain payment before opening a paid session. The Seeder treats the blockchain as the source of truth and MUST NOT rely solely on the `payment_proof` contents.

#### 3.3.1 On‑Chain Lookup

Upon receiving a `payment_proof` message, the Seeder performs a read‑only lookup on the configured chain using the provided `tx_signature`. The Seeder MUST:

1. Fetch the transaction by `tx_signature` from a trusted RPC endpoint.
2. Reject the proof if the transaction cannot be found or has not yet reached at least **confirmed** status.
3. Parse the transaction to locate:
   - The token transfer instruction for the agreed token (e.g. USDC)
   - The sender and recipient accounts
   - The transferred amount
   - Any memo instruction attached to the transaction

If any of these steps fail (e.g. malformed transaction, missing token transfer, missing memo), the Seeder MUST treat the payment as invalid.

#### 3.3.2 Validation Rules

A payment proof is considered **valid** only if all of the following checks pass:

1. **Recipient check**
   - The recipient of the token transfer MUST equal the Seeder’s configured `wallet` from the handshake.
2. **Amount check**
   - The transferred amount MUST be greater than or equal to the Seeder’s `min_prepayment`.
   - The Seeder MAY also enforce a maximum or minimum useful amount for its own policy.
3. **Token check**
   - The transfer MUST be in the expected token (e.g. USDC) and on the expected chain.
4. **Memo binding check**
   - The transaction MUST contain a memo whose decoded JSON matches the SeedPay memo format:
     ```json
     {
       "protocol": "seedpay",
       "version": "1.0",
       "from_peer_id": "<leecher-peer-id>",
       "to_peer_id": "<seeder-peer-id>",
       "nonce": 1702700000000
     }
     ```
   - `protocol` MUST equal `"seedpay"` and `version` MUST match a version the Seeder supports.
   - `from_peer_id` MUST equal the Leecher’s `peer_id` observed on this connection.
   - `to_peer_id` MUST equal the Seeder’s own `peer_id`.
5. **Freshness / replay protection**
   - The Seeder MUST ensure the transaction is recent: `nonce` or the transaction’s block time MUST be within an acceptable window (e.g. 5–10 minutes) to prevent reuse of old payments.
   - The Seeder MUST maintain a set of already‑consumed transaction signatures and reject any payment proof that references a previously accepted `tx_signature`.
6. **Error‑free execution**
   - The transaction’s metadata MUST indicate success; any failed or reverted transaction MUST be treated as invalid.

If any validation step fails, the Seeder MUST reject the payment.

#### 3.3.3 Seeder Response

After running the validation checks, the Seeder responds over the SeedPay extension channel.

**On success**, the Seeder:

1. Creates a **payment session** associated with the Leecher’s `peer_id` and this connection, initializing:
   - `prepaid_balance` = verified token amount
   - `bytes_downloaded` = 0
   - `price_per_mb` = value from the handshake
   - `expires_at` = current time + session lifetime (implementation‑defined)
2. Sends a `payment_confirmed` message to the Leecher:

```json
{
  "type": "payment_confirmed",
  "confirmed": true,
  "balance": 0.01,
  "price_per_mb": 0.0001,
  "expires_at": 1702703600000
}
```

3. **Unchokes** the Leecher on the BitTorrent wire, allowing piece requests to proceed.

**On failure**, the Seeder:

1. Sends a `payment_rejected` message:

```json
{
  "type": "payment_rejected",
  "confirmed": false,
  "reason": "tx_not_found" | "tx_failed" | "wrong_recipient" |
            "insufficient_amount" | "memo_mismatch" | "replayed_tx" |
            "expired"
}
```

2. Keeps the Leecher **choked**, so no paid pieces are sent for this session.
3. MAY allow the Leecher to retry with a new payment, or MAY close the connection according to local policy.

Once a payment has been confirmed and a payment session created, the protocol transitions to the **Data Transfer Phase**, where each piece sent is accounted against the Leecher’s prepaid balance.

### 3.4 Data Transfer

After a payment session is established, the Seeder begins serving pieces to the Leecher while decrementing the Leecher's prepaid balance. This phase reuses the normal BitTorrent request/response messages and adds only local accounting and a few SeedPay control messages.

#### 3.4.1 Mapping Pieces to Cost

SeedPay doesn't modify the BitTorrent wire messages used for data transfer (`request`, `piece`, `cancel`). Instead the Seeder:

- observes each `request` message from the Leecher
- computes the byte length of the requested block
- converts that length into monetary cost using the agreed `price_per_mb` from the handshake:

$$
\text{cost} = \frac{\text{bytes}}{1024 \times 1024} \times \text{price\_per\_mb}
$$

The Seeder maintains, per payment session:

- `prepaid_balance` (remaining paid amount, in token units)
- `bytes_downloaded` (cumulative bytes served under this session)
- `price_per_mb` (agreed rate)

A Seeder MAY track usage either on `request` or on successful `piece` send.

#### 3.4.2 Serving Requests Under Balance Constraints

For each upcoming `request(index, begin, length)` from the Leecher, the Seeder performs:

1. Look up the associated payment session for this `peer_id` and connection.
   - If no active session exists, the Seeder MUST keep the peer chocked and MUST NOT serve paid pieces.
2. Compute the monetary cost of serving this block as described above.
3. Check whether `prepaid_balance >= cost`

If `prepaid_balance` is sufficient:

- The Seeder decrements `prepaid_balance` by `cost`.
- Increments `bytes downloaded` by `length`.
- Sends the corresponding `piece` message as in standard BitTorrent.

If `prepaid_balance` is not sufficient:

- The Seeder may send a `balance_low` control message over the SeedPay extension:

```json
{
  "type": "balance_low",
  "remaining": 0.0003,
  "estimated_remaining_mb": 3.0
}
```

- The Seeder SHOULD choke the Leecher (normal BitTorrent `choke` state) to prevent further paid downloads until the balance is topped up or a new payment proof is received.

The Leecher, upon receiving `balance_low` or observing stalled progress, MAY initiate another Payment Submission Phase with a fresh transaction and `payment_proof`.

#### 3.4.3 Session Lifetime and Expiry

Payment sessions are not intended to last indefinitely. A Seeder SHOULD treat a session as expired if any of the following conditions hold:

- `expires_at` (set during payment_confirmed) is in the past.
- The connection is closed or idle for longer than an implementation‑defined timeout.
- A new `payment_proof` is received for the same peer_id and torrent, in which case the Seeder MAY close the old session and open a new one with the new balance.

On expiry, the Seeder:

- Discards the session state (balance, usage counters).
- Keeps or reverts to the choked state for that peer.
- MAY continue to serve as a free seeder (if its local policy allows) or close the connection.

#### 3.4.4 Interaction with Ratio Credits

If the Seeder’s handshake advertised `accepts_ratio = true`, the client MAY combine balance checks with a separate ratio credit check instead of strict “balance or nothing” behavior:

- If `prepaid_balance` is exhausted but the Leecher presents valid ratio credits, the Seeder may continue serving pieces and decrement ratio credits instead of token balance.

This design allows the same Data Transfer Phase to handle pure paid sessions, pure ratio‑based sessions, or hybrid sessions without modifying the underlying BitTorrent wire protocol.

## 4. Ratio Credits System

### 4.1 Overview

_[Draft/TBD - This section will detail the ratio credits mechanism]_

Ratio credits allow leechers to earn access to paid seeders by seeding other torrents, creating a circular economy within the SeedPay network.

### 4.2 Credit Earning

_[Draft/TBD - How leechers earn credits by seeding]_

### 4.3 Credit Spending

_[Draft/TBD - How credits are spent and verified]_

### 4.4 Credit Transfer and Verification

_[Draft/TBD - On-chain or off-chain credit ledger, verification mechanisms]_

## 5. Security Considerations

### 5.1 Payment Verification

_[Draft/TBD - Security considerations for on-chain payment verification, replay attacks, etc.]_

### 5.2 Peer Authentication

_[Draft/TBD - How to prevent peer_id spoofing, Sybil attacks]_

### 5.3 Economic Attacks

_[Draft/TBD - Mitigations for payment fraud, double-spending attempts, etc.]_

### 5.4 Privacy Considerations

_[Draft/TBD - Privacy implications of on-chain payments, peer_id linking, etc.]_

## 6. Implementation Notes

### 6.1 Blockchain Integration

_[Draft/TBD - Specific implementation details for Solana integration, transaction construction, RPC usage]_

### 6.2 Client Compatibility

_[Draft/TBD - How to implement SeedPay as a BitTorrent client extension, backward compatibility testing]_

### 6.3 Performance Considerations

_[Draft/TBD - Latency implications, RPC rate limits, session management overhead]_

### 6.4 Error Handling

_[Draft/TBD - Common failure modes, retry strategies, connection recovery]_

## 7. Future Extensions

### 7.1 Multi-Chain Support

_[Draft/TBD - Extending to other blockchains beyond Solana]_

### 7.2 Advanced Payment Models

_[Draft/TBD - Pay-per-piece, subscription models, etc.]_

### 7.3 Reputation Systems

_[Draft/TBD - Seeder reputation, leecher trust scores]_

## 8. References

### 8.1 BitTorrent Protocol Specifications

- BEP 3: [The BitTorrent Protocol Specification](https://www.bittorrent.org/beps/bep_0003.html)
- BEP 10: [Extension Protocol](https://www.bittorrent.org/beps/bep_0010.html)

### 8.2 Related Work

_[Draft/TBD - References to related protocols, research papers, etc.]_

---

**Note**: Sections marked as "Draft/TBD" are placeholders for future specification work. Contributions and feedback on these areas are especially welcome.
