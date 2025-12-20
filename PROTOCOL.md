<div align="center">

# SeedPay: Payments Protocol for BitTorrent Networks

## Seeders earn crypto for sharing files. Leechers pay for faster downloads or earn credits by seeding.

</div>

> ⚠️ **Status: Pre-Alpha / Request For Comments (RFC)**
>
> This specification is **v0.2 draft** incorporating privacy protections based on community feedback.
>
> **What's New in v0.2:**
>
> 1. ✅ **Privacy Protection Implemented**: Ephemeral Session Keys (ECDH-based) replace raw PeerIDs in on-chain memos, preventing linkage between IP addresses and wallet addresses.
> 2. ✅ **Simplified Payment Model**: Direct on-chain payments for V1 (no channels needed given Solana's ~$0.00025 transaction costs).
> 3. 🚧 **Ratio Credits (TBD)**: System redesign in progress to prevent Sybil attacks and ensure fairness.
>
> **This is research software. Do not use in production.**
>
> See [CONTRIBUTING.md](./CONTRIBUTING.md) for how to provide feedback.

## Abstract

SeedPay is an open payment protocol that enables BitTorrent seeders to earn cryptocurrency for sharing files, while giving leechers two paths for accessing the content: pay seeders directly with stablecoins (e.g. USDC), or earn ratio credits by seeding other torrents.

By extending the BitTorrent Wire Protocol with payment handshakes and blockchain-verified proofs, SeedPay solves the free-rider problem without requiring centralized infrastructure or breaking compatibility with existing clients.

The protocol supports micropayments as low as $0.0001/MB with near-instant settlement, backward compatibility with standard BitTorrent, and optional participation. **V0.2 introduces ephemeral session keys to ensure payment privacy—blockchain observers cannot link wallet addresses to download activity.**

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
- **Privacy-preserving** (v0.2): On-chain payments cannot be linked to download activity

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

#### 3.1.2 Extended Handshake (BEP 10)

Immediately after the standard handshake, peers that support the extension protocol exchange an extended handshake (BitTorrent message ID `20`, extended ID `0`). The payload of this message is a bencoded dictionary that:

- Advertises which extension the peer supports via `m` map (extension name → numeric ID)
- May include additional, extension-specific data

SeedPay defines a new extension named "seedpay". A Seeder that supports paid seeding MUST include an entry for "seedpay" in the m dictionary:

```json
{
  "m": {
    "seedpay": 5,
    "ut_metadata": 2
  },
  "v": "SeedPayClient 0.2",
  "seedpay": {
    "wallet": "DYw8jCN...",
    "price_per_mb": 0.0001,
    "accepts_ratio": 1,
    "min_prepayment": 0.01,
    "chain": "solana"
  }
}
```

- The numeric value `5` is the local extension ID the Seeder will use when sending and receiving SeedPay messages on this connection. The exact ID is implementation-defined and only needs to be consistent for this peer pair.
- The nested `seedpay` dictionary contains the Seeder's payment capabilities:
  - `wallet`: seeder's on-chain wallet address for receiving payments
  - `price_per_mb`: quoted price in USDC per MB
  - `accepts_ratio`: whether this peer accepts ratio credits instead of payments
  - `min_prepayment`: minimum amount required to start a paid session
  - `chain`: identifier of the settlement chain (e.g. `"solana"`)

A Leecher that supports SeedPay should also include `seedpay` in its `m` map:

```json
{
  "m": {
    "seedpay": 3
  },
  "v": "SeedPayClient 0.2"
}
```

After both extended handshakes are exchanged, each peer:

- checks whether the remote `m` contains `"seedpay"`
- records the remote extension id for `"seedpay"`
- parses the remote `seedpay` object (if present) to learn the counterparty's wallet, pricing, and ratio policy.

If either side's extended handshake does not include `"seedpay"` in `m`, the connection continues as a normal BitTorrent session without payments.

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

Only after this handshake phase completes does the protocol move into the Payment Submission Phase, where Leechers establish ephemeral session keys and create on-chain transactions.

### 3.2 Payment Submission

Once the handshake phase is complete and the Leecher has discovered a Seeder advertising SeedPay support, the next step is to establish a cryptographically-bound payment session. **V0.2 uses Ephemeral Session Keys to ensure privacy**—the on-chain payment contains no peer_id or IP address information that could link wallet activity to download behavior.

#### 3.2.1 Ephemeral Key Exchange (ECDH)

Before creating the payment transaction, both peers establish a shared secret using Elliptic Curve Diffie-Hellman (ECDH) key agreement. This happens inside the existing MSE (Message Stream Encryption) tunnel, ensuring the exchange cannot be observed by network eavesdroppers.

**Process:**

1. **Leecher generates ephemeral keypair:**

   ```
   leecher_secret_key = random_32_bytes()
   leecher_public_key = leecher_secret_key × G
   ```

   Where `G` is the standard generator point on Curve25519.

2. **Seeder generates ephemeral keypair:**

   ```
   seeder_secret_key = random_32_bytes()
   seeder_public_key = seeder_secret_key × G
   ```

3. **Public key exchange (over encrypted MSE tunnel):**

   The peers exchange their ephemeral public keys via a SeedPay extension message:

   ```json
   {
     "type": "ecdh_init",
     "ephemeral_pk": "<32-byte-public-key-hex>"
   }
   ```

4. **Both parties compute shared secret:**

   ```
   Leecher: shared_secret = leecher_secret_key × seeder_public_key
   Seeder:  shared_secret = seeder_secret_key × leecher_public_key
   ```

   Due to the properties of elliptic curve mathematics, both computations yield the same `shared_secret`.

5. **Derive Session UUID using HKDF:**
   ```
   Session_UUID = HKDF-Expand(
     key: shared_secret,
     info: "seedpay-v1-session",
     length: 32 bytes
   )
   ```

The `Session_UUID` is now known only to these two peers and cryptographically binds this payment to this specific TCP connection.

**Why "seedpay-v1-session"?**

This context string provides:

- **Domain Separation**: Ensures Session_UUID is cryptographically distinct from other derived keys (e.g., if we later use the same shared_secret for encryption)
- **Version Compatibility**: Future versions can use "seedpay-v2-session" without conflicts
- **Protocol Isolation**: Prevents key reuse if another protocol also uses ECDH on the same connection

**Security Properties:**

- **Forward Secrecy**: If ephemeral keys are deleted after the session, past sessions cannot be decrypted even if long-term wallets are compromised.
- **MITM Resistance**: An attacker cannot forge the `Session_UUID` without knowing one of the private keys.
- **Unlinkability**: Blockchain observers see only `Hash(Session_UUID)` in the memo—they cannot reverse it or link it to peer activity.
- **Replay Protection**: Each session has a unique `Session_UUID`; old payment proofs cannot be reused.

#### 3.2.2 Pricing and Session Setup

From the extended handshake, the Leecher learns the Seeder's terms:

- `wallet`: on-chain wallet address that receives the funds
- `price_per_mb`: quoted price in USDC per MB
- `min_prepayment`: minimum amount required to open a session
- `chain`: settlement chain identifier (e.g. `"solana"`)

The Leecher may also apply local policy, such as:

- Maximum price per MB it is willing to pay
- Maximum total spend per torrent per session
- Preference for using ratio credits instead of payment when available

If the Seeder's advertised terms are acceptable, the Leecher computes an initial prepayment amount large enough to cover some target amount of data (for example, 50–200 MB), but at least `min_prepayment`.

#### 3.2.3 Creating the On-Chain Payment

To start a paid session, the Leecher creates and submits a token transfer transaction on the configured chain. The transaction MUST:

- Transfer at least `min_prepayment` (typically `price_per_mb × target_mb`) of the agreed token (e.g. USDC)
- Use the Seeder's `wallet` as the recipient
- Include a memo containing the **hash of the Session UUID** (not raw peer_ids)

**Memo Format (Privacy-Preserving):**

The memo is a simple JSON object containing only the opaque session identifier:

```json
{
  "protocol": "seedpay",
  "version": "1.0",
  "session_hash": "a3f5c8d9e2b1...",
  "nonce": 1702700000000
}
```

Where:

- `session_hash` = hex(SHA-256(Session_UUID)) — an opaque 32-byte hash
- `nonce` = timestamp (Unix milliseconds) for freshness and replay prevention

**Privacy Properties:**

- No `peer_id` on chain (cannot link to swarm activity)
- No IP address information
- `session_hash` is computationally irreversible (256-bit preimage resistance)
- Each session has a unique hash (unlinkable across sessions)
- Blockchain observers cannot determine what torrent is being downloaded

The Leecher signs and submits the transaction directly to the blockchain. Once the network accepts the transaction and returns the transaction signature, the Leecher SHOULD wait until the transaction reaches at least **confirmed** status before proceeding.

#### 3.2.4 Sending the Payment Proof to the Seeder

After the transaction is confirmed on-chain, the Leecher sends a payment proof to the Seeder over the SeedPay extension channel. This message contains only the transaction signature—the Seeder will independently verify the payment on-chain.

```json
{
  "type": "payment_proof",
  "tx_signature": "5Kn8yF...",
  "amount": 0.01,
  "timestamp": 1702700000000
}
```

- `tx_signature`: the blockchain transaction signature (Solana base58 format)
- `amount`: the amount paid (for UI/logging purposes only, not trusted by Seeder)
- `timestamp`: when the proof was created (for freshness checks)

**Important:** The Seeder MUST NOT rely on the `amount` field or any other client-provided data. All validation is done against the on-chain transaction.

A Seeder that receives a `payment_proof` MUST NOT start sending pieces yet. Instead, it proceeds to the Verification Phase (Section 3.3).

### 3.3 Verification Phase

In the verification phase, the Seeder independently checks the on-chain payment before opening a paid session. The Seeder treats the blockchain as the source of truth and MUST NOT rely solely on the `payment_proof` contents.

#### 3.3.1 On-Chain Lookup

Upon receiving a `payment_proof` message, the Seeder performs a read-only lookup on the configured chain using the provided `tx_signature`. The Seeder MUST:

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

   - The recipient of the token transfer MUST equal the Seeder's configured `wallet` from the handshake.

2. **Amount check**

   - The transferred amount MUST be greater than or equal to the Seeder's `min_prepayment`.
   - The Seeder MAY also enforce a maximum amount for its own policy.

3. **Token check**

   - The transfer MUST be in the expected token (e.g. USDC SPL token) and on the expected chain.

4. **Memo binding check (Privacy-Preserving)**

   - The transaction MUST contain a memo whose decoded JSON matches the SeedPay memo format:
     ```json
     {
       "protocol": "seedpay",
       "version": "1.0",
       "session_hash": "<hex-string>",
       "nonce": 1702700000000
     }
     ```
   - `protocol` MUST equal `"seedpay"` and `version` MUST match a version the Seeder supports.
   - **Session Binding**: The Seeder computes `expected_hash = SHA-256(Session_UUID)` using the Session_UUID derived during ECDH key exchange (Section 3.2.1).
   - The `session_hash` field in the memo MUST equal `expected_hash`.
   - This proves the payment is bound to THIS specific connection without revealing peer_id or IP information.

5. **Freshness / replay protection**

   - The Seeder MUST ensure the transaction is recent: `nonce` or the transaction's block time MUST be within an acceptable window (e.g. 5–10 minutes) to prevent reuse of old payments.
   - The Seeder MUST maintain a set of already-consumed transaction signatures and reject any payment proof that references a previously accepted `tx_signature`.

6. **Error-free execution**
   - The transaction's metadata MUST indicate success; any failed or reverted transaction MUST be treated as invalid.

If any validation step fails, the Seeder MUST reject the payment.

#### 3.3.3 Seeder Response

After running the validation checks, the Seeder responds over the SeedPay extension channel.

**On success**, the Seeder:

1. Creates a **payment session** associated with this connection, initializing:

   - `prepaid_balance` = verified token amount
   - `bytes_downloaded` = 0
   - `price_per_mb` = value from the handshake
   - `expires_at` = current time + session lifetime (implementation-defined, e.g. 1 hour)

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
               "insufficient_amount" | "session_mismatch" | "replayed_tx" |
               "expired"
   }
   ```

   Note: `"session_mismatch"` replaces `"memo_mismatch"` from v0.1 to reflect the new ECDH-based binding.

2. Keeps the Leecher **choked**, so no paid pieces are sent for this session.
3. MAY allow the Leecher to retry with a new payment, or MAY close the connection according to local policy.

Once a payment has been confirmed and a payment session created, the protocol transitions to the **Data Transfer Phase** (Section 3.4).

### 3.4 Data Transfer

After a payment session is established, the Seeder begins serving pieces to the Leecher while decrementing the Leecher's prepaid balance. This phase reuses the normal BitTorrent request/response messages and adds only local accounting and a few SeedPay control messages.

#### 3.4.1 Mapping Pieces to Cost

SeedPay doesn't modify the BitTorrent wire messages used for data transfer (`request`, `piece`, `cancel`). Instead the Seeder:

- observes each `request` message from the Leecher
- computes the byte length of the requested block
- converts that length into monetary cost using the agreed `price_per_mb` from the handshake:

$$
\text{cost} = \frac{\text{bytes}}{1024 \times 1024} \times \text{pricePerMB}
$$

The Seeder maintains, per payment session:

- `prepaid_balance` (remaining paid amount, in token units)
- `bytes_downloaded` (cumulative bytes served under this session)
- `price_per_mb` (agreed rate)

A Seeder MAY track usage either on `request` or on successful `piece` send. For accuracy, tracking on successful `piece` send is RECOMMENDED.

#### 3.4.2 Serving Requests Under Balance Constraints

For each upcoming `request(index, begin, length)` from the Leecher, the Seeder performs:

1. Look up the associated payment session for this connection.

   - If no active session exists, the Seeder MUST keep the peer choked and MUST NOT serve paid pieces.

2. Compute the monetary cost of serving this block as described above.

3. Check whether `prepaid_balance >= cost`

If `prepaid_balance` is sufficient:

- The Seeder decrements `prepaid_balance` by `cost`.
- Increments `bytes_downloaded` by `length`.
- Sends the corresponding `piece` message as in standard BitTorrent.

If `prepaid_balance` is not sufficient:

- The Seeder MAY send a `balance_low` control message over the SeedPay extension:

  ```json
  {
    "type": "balance_low",
    "remaining": 0.0003,
    "estimated_remaining_mb": 3.0
  }
  ```

- The Seeder SHOULD choke the Leecher (normal BitTorrent `choke` state) to prevent further paid downloads until the balance is topped up or a new payment proof is received.

The Leecher, upon receiving `balance_low` or observing stalled progress, MAY initiate another Payment Submission Phase with a fresh transaction and `payment_proof`. The new payment will use the same `Session_UUID` (derived from the same ECDH keys for this connection), allowing the Seeder to top up the existing session balance.

#### 3.4.3 Session Lifetime and Expiry

Payment sessions are not intended to last indefinitely. A Seeder SHOULD treat a session as expired if any of the following conditions hold:

- `expires_at` (set during payment_confirmed) is in the past.
- The connection is closed or idle for longer than an implementation-defined timeout.
- The ephemeral keys have been deleted (session cannot be verified for new payments).

On expiry, the Seeder:

- Discards the session state (balance, usage counters).
- Deletes the ephemeral secret key (ensures forward secrecy).
- Keeps or reverts to the choked state for that peer.
- MAY continue to serve as a free seeder (if its local policy allows) or close the connection.

#### 3.4.4 Interaction with Ratio Credits

If the Seeder's handshake advertised `accepts_ratio = true`, the client MAY combine balance checks with a separate ratio credit check instead of strict "balance or nothing" behavior:

- If `prepaid_balance` is exhausted but the Leecher presents valid ratio credits, the Seeder may continue serving pieces and decrement ratio credits instead of token balance.

This design allows the same Data Transfer Phase to handle pure paid sessions, pure ratio-based sessions, or hybrid sessions without modifying the underlying BitTorrent wire protocol.

## 4. Ratio Credits System

### 4.1 Overview

_[Draft/TBD - This section will detail the ratio credits mechanism]_

Ratio credits allow leechers to earn access to paid seeders by seeding other torrents, creating a circular economy within the SeedPay network.

**Design Challenges Being Addressed:**

- Preventing Sybil attacks (users farming credits with fake seeders)
- Ensuring credits have real value (tied to actual bandwidth provision)
- Cross-torrent credit portability

### 4.2 Credit Earning

_[Draft/TBD - How leechers earn credits by seeding]_

### 4.3 Credit Spending

_[Draft/TBD - How credits are spent and verified]_

### 4.4 Credit Transfer and Verification

_[Draft/TBD - On-chain or off-chain credit ledger, verification mechanisms]_

## 5. Security Considerations

### 5.1 Privacy Model

**V0.2 Privacy Guarantees:**

The ephemeral session key design provides the following privacy properties:

1. **Unlinkability (Blockchain → Swarm)**

   - Blockchain observers see: `wallet_A → wallet_B, memo: {session_hash: "0xabc..."}`
   - They CANNOT determine: which torrent, which peer_id, which IP address
   - Preimage resistance of SHA-256 prevents reversing the session_hash

2. **Unlinkability (Session → Session)**

   - Each TCP connection uses fresh ephemeral keys
   - Different sessions produce different Session_UUIDs
   - Blockchain observers cannot link multiple payments from the same user

3. **Forward Secrecy**
   - Ephemeral keys deleted after session ends
   - Compromising a wallet AFTER the fact cannot decrypt past sessions
   - Past download history remains private

**What is NOT Private:**

- The fact that wallet_A paid wallet_B (amounts and timing are public)
- Seeder wallet addresses (visible in handshake and on-chain)
- Connection metadata visible to ISP/network observer (use VPN/Tor for full privacy)

**Privacy Best Practices for Users:**

- Use Tor or VPN for IP address privacy
- Use burner wallets funded via mixers for maximum anonymity
- Avoid reusing the same Seeder/Leecher combination if privacy is critical

### 5.2 Payment Verification

**Security Properties:**

- Seeders MUST verify payments on-chain (blockchain is source of truth)
- ECDH binding prevents payment proof replay across different connections
- Nonce freshness prevents replay of old payments
- Transaction signature tracking prevents double-spending

**Attack Mitigations:**

- **Fake Payment Proof**: Seeder fetches transaction independently, ignores Leecher-provided `amount`
- **Replay Attack**: Seeder checks nonce freshness and maintains consumed tx_signature set
- **MITM Attack**: ECDH ensures only peers with correct ephemeral keys can derive Session_UUID

### 5.3 Peer Authentication

_[Draft/TBD - How to prevent peer_id spoofing, Sybil attacks in ratio credit system]_

### 5.4 Economic Attacks

_[Draft/TBD - Mitigations for payment fraud, griefing attacks, etc.]_

## 6. Implementation Notes

### 6.1 Cryptographic Requirements

**Elliptic Curve:** Curve25519 (x25519)

- Well-established, audited implementations available
- Fast key generation and ECDH computation
- Widely supported (libsodium, NaCl, OpenSSL)

**Key Derivation:** HKDF-SHA256

- Context string: `"seedpay-v1-session"`
- Output length: 32 bytes

**Hashing:** SHA-256

- For computing `session_hash` from Session_UUID

**Random Number Generation:**

- MUST use cryptographically secure RNG (e.g., `/dev/urandom`, `crypto.getRandomValues()`)
- Ephemeral secret keys MUST be truly random (no deterministic derivation)

### 6.2 Blockchain Integration

_[Draft/TBD - Specific implementation details for Solana integration, transaction construction, RPC usage]_

**Solana Specifics:**

- Use SPL Token program for USDC transfers
- Memo program for attaching JSON memo
- Target confirmation level: "confirmed" minimum, "finalized" recommended

### 6.3 Client Compatibility

_[Draft/TBD - How to implement SeedPay as a BitTorrent client extension, backward compatibility testing]_

**MSE (Message Stream Encryption) Requirement:**

- SeedPay V0.2 REQUIRES MSE for ECDH key exchange
- Clients MUST implement BEP 52 (MSE) to support SeedPay payments
- Fallback to unencrypted mode is NOT supported for payment sessions

### 6.4 Performance Considerations

**ECDH Overhead:**

- Key generation: ~0.1ms on modern hardware
- Key exchange: 2 round-trips (within existing handshake flow)
- HKDF derivation: <0.01ms

**RPC Rate Limits:**

- Solana public RPCs: ~10 req/sec
- Recommended: use dedicated RPC provider for production Seeders
- Payment verification requires 1 RPC call per payment proof

### 6.5 Error Handling

_[Draft/TBD - Common failure modes, retry strategies, connection recovery]_

## 7. Future Extensions

### 7.1 Multi-Chain Support

_[Draft/TBD - Extending to other blockchains beyond Solana]_

The ECDH-based session binding is chain-agnostic and can be implemented on:

- Ethereum (use EIP-3009 for meta-transactions)
- Base, Arbitrum, Optimism (L2s with low fees)
- Other EVM-compatible chains

### 7.2 Advanced Payment Models

**Payment Channels (V2+):**

- For high-frequency micropayments (1000+ payments per session)
- Reduces on-chain footprint to 2 transactions (open/close)
- Uses same ECDH session binding for off-chain state

**Probabilistic Payments:**

- Lottery-style micropayments for extreme scalability
- Trade certainty for reduced transaction count

### 7.3 Reputation Systems

_[Draft/TBD - Seeder reputation, leecher trust scores]_

## 8. References

### 8.1 BitTorrent Protocol Specifications

- BEP 3: [The BitTorrent Protocol Specification](https://www.bittorrent.org/beps/bep_0003.html)
- BEP 10: [Extension Protocol](https://www.bittorrent.org/beps/bep_0010.html)
- BEP 52: [Message Stream Encryption](https://www.bittorrent.org/beps/bep_0052.html)

### 8.2 Cryptographic Standards

- RFC 7748: [Elliptic Curves for Security (Curve25519)](https://tools.ietf.org/html/rfc7748)
- RFC 5869: [HKDF (HMAC-based Key Derivation Function)](https://tools.ietf.org/html/rfc5869)
- NIST FIPS 180-4: [Secure Hash Standard (SHA-256)](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.180-4.pdf)

### 8.3 Related Work

- [Coinbase x402 Protocol](https://github.com/coinbase/x402) - HTTP-native micropayments
- [Lightning Network](https://lightning.network/) - Bitcoin payment channels
- [BitTorrent Token (BTT)](https://www.bittorrent.com/token/btt/) - Previous attempt at paid BitTorrent

---

**Changelog:**

- **v0.2 (2024-12-20)**: Introduced ECDH-based ephemeral session keys for privacy, removed peer_id from on-chain memos, simplified payment model (no channels for V1)
- **v0.1 (2024-12-15)**: Initial draft with peer_id-based memo binding (deprecated due to privacy concerns)

---

**Note**: Sections marked as "Draft/TBD" are placeholders for future specification work. Contributions and feedback on these areas are especially welcome.
