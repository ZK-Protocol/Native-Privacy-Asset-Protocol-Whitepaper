# Native Privacy Asset Protocol Whitepaper
## Next-Generation Privacy Infrastructure Based on Zero-Knowledge Proofs

**Version:** 1.0  
**Release Date:** 2025

---

## Abstract

This protocol is an open-source, permissionless infrastructure based on zero-knowledge proofs (zk-SNARKs), designed for Ethereum and its compatible multi-chain ecosystem to provide native privacy asset issuance and trading capabilities. Unlike traditional mixing protocols, native privacy assets possess privacy attributes from their inception, allowing anyone to issue and use them as easily as ERC-20 tokens.

The protocol introduces the **ZRC-20 standard** and **factory model**, supporting fair launch and composability. Through innovative **tiered/dual-layer Merkle tree architecture**, efficient **zk-SNARK proof systems**, and forward-looking **pv1 multi-chain privacy address format**, it achieves a balance between performance, scalability, and compliance.

This protocol aims to become the privacy infrastructure layer for the Ethereum ecosystem, unlocking unprecedented possibilities for DeFi, DAOs, enterprise payments, and metaverse applications.

---

## 1. Introduction

### 1.1 The Blockchain Privacy Dilemma

The transparency of public blockchains brings security and auditability to the system, but also exposes user assets, transactions, and strategies. For individuals, this means loss of financial privacy; for enterprises, it may reveal trade secrets and investment strategies.

### 1.2 Limitations of Existing Privacy Solutions

**Privacy Coins (Zcash, Monero)**: Possess native privacy but lack programmability and are disconnected from the Ethereum ecosystem.

**Mixing Protocols (Tornado Cash)**: Provide one-time privacy but cannot create new privacy assets, have limited composability, and face compliance risks.

**Privacy Rollups (Aztec, etc.)**: Reduce costs but introduce new trust assumptions such as centralized sequencers.

### 1.3 The Value of Native Privacy Assets

This protocol proposes the concept of **Native Privacy Assets**: privacy is an inherent property of the asset, not a result of post-hoc mixing. This not only enhances the simplicity of technical implementation but also creates new possibilities for legal compliance.

---

## 2. Protocol Architecture

The protocol consists of four core pillars: Privacy Model, State Management, ZRC-20 Standard & Factory, and Privacy Addresses.

### 2.1 Privacy Model

**Stealth Addresses**: Generate one-time addresses based on sender's ephemeral keys and recipient's public keys, ensuring recipient identity cannot be linked by external parties.

**Zero-Knowledge Proofs (zk-SNARKs)**: Users prove ownership, prevent double-spending, and maintain value conservation without revealing notes, amounts, or identities.

**Note Encryption (Baby Jubjub ECIES)**: Notes contain amounts and randomness, encrypted end-to-end, decryptable only by recipients.

**View Tags**: Single-byte identifiers derived from shared secrets, helping recipients quickly filter relevant transactions and significantly reducing scanning costs.

### 2.2 State Management: Tiered/Dual-Layer Merkle Tree

The protocol uses a dual-layer tree structure:

**Active Subtree**: Small-scale, low-cost insertion and verification, storing the latest notes.

**Root Tree**: Stores subtree roots, serving as historical archive.

**Three Transaction Modes**:

- **Active Transfer**: Intra-active tree transactions, lowest cost
- **Finalized Transfer**: Historical tree transactions, supports parallel processing
- **Rollover Transfer**: Used when triggering subtree rollover

---

## 3. ZRC-20 Standard

### 3.1 Standard Introduction

ZRC-20 (ZK Privacy ERC-20) is a native privacy token standard built on zero-knowledge proofs. While maintaining ERC-20-like interface semantics for developer experience, it is completely based on commitments, nullifiers, and zero-knowledge proofs at the implementation level, ensuring privacy of transaction amounts and account balances.

ZRC-20's goal is to enable anyone to issue privacy tokens as easily as ERC-20 tokens, while maintaining composability and embedding anti-double-spending and transaction confidentiality mechanisms.

### 3.2 Interface Specification

The following is the ZRC-20 interface definition (IZRC20.sol):

```solidity
interface IZRC20 {
    /**
     * @notice Mints new privacy tokens.
     * @param proofType The type of proof (0: Regular Mint, 1: Rollover Mint).
     * @param proof An ABI-encoded bytestring containing the zk-SNARK proof and all its public signals.
     * @param encryptedNote The encrypted note data for the minter's own decryption.
     */
    function mint(
        uint8 proofType,
        bytes calldata proof,
        bytes calldata encryptedNote
    ) external payable;

    /**
     * @notice Executes a private transfer.
     * @param proofType The type of proof (0: Active, 1: Finalized, 2: Rollover).
     * @param proof An ABI-encoded bytestring containing the zk-SNARK proof and all its public signals.
     * @param encryptedNotes The encrypted output notes for the recipient and/or change.
     */
    function transfer(
        uint8 proofType,
        bytes calldata proof,
        bytes[] calldata encryptedNotes
    ) external;

    /// @notice A new commitment has been appended to the Merkle tree.
    event CommitmentAppended(uint32 indexed subtreeIndex, bytes32 commitment, uint32 indexed leafIndex, uint256 timestamp);

    /// @notice A nullifier has been spent, marking a note as used.
    event NullifierSpent(bytes32 indexed nullifier);
    
    /// @notice A transaction has occurred, emitting public data for scanners.
    event Transaction(
        bytes32[2] newCommitments, 
        bytes[] encryptedNotes, 
        uint256[2] ephemeralPublicKey, 
        uint256 viewTag
    );
}
```

### 3.3 Functionality Description

**mint**
- Converts public assets (like ERC-20) into ZRC-20 privacy notes
- Input is zero-knowledge proof (proving user paid assets and generated new commitments)
- Triggers `CommitmentAppended` event upon success

**transfer**
- Users select their commitments as inputs, generate zk-SNARK proof:
  - Input commitments belong to Merkle Tree
  - Prover owns private keys of input notes
  - Nullifiers are unspent
  - Input amounts = Output amounts (value conservation)
- Upon successful execution:
  - Triggers `NullifierSpent` event (marking old notes as void)
  - Triggers `CommitmentAppended` event (generating new commitments)
  - Triggers `Transaction` event (only exposing a public hash for on-chain verification without revealing amounts and addresses)

**Event Mechanism**
- **CommitmentAppended**: Clients sync new privacy notes by monitoring this event
- **NullifierSpent**: Clients record whether notes are spent to prevent double-spending
- **Transaction**: Only exposes an irreversible hash value externally, protecting transfer details

### 3.4 Differences from ERC-20

| Feature | ERC-20 | ZRC-20 |
|---------|---------|---------|
| Balance Query | `balanceOf(address)` | None (balance privacy, requires local client calculation) |
| Transfer Interface | `transfer(address,uint256)` | `transfer(proof,publicInputs,encryptedNotes)` |
| Visibility | Fully public | Amounts, addresses, balances all private |
| Double-spending Prevention | Account balance check | Nullifier mechanism |
| State Storage | `address → balance` mapping | Merkle Tree of commitments + nullifiers |

### 3.5 Security and Privacy Guarantees

- **Privacy**: Amounts, senders, and recipients do not appear in on-chain events
- **Anti-double-spending**: Each nullifier is unique and can only be used once
- **Composability**: ZRC-20 maintains ERC-20-like abstraction, enabling integration with DeFi, DAO scenarios
- **Scalability**: All instances created through factory contracts with unified logic, reducing security risks

### 3.6 Factory Model and ZRC-20 Lifecycle

1. **Token Creation**: Any user deploys new ZRC-20 contract instances through PrivacyTokenFactory
2. **Minting**: Any user can call `mint`, pay public assets and generate privacy commitments
3. **Transfer**: Private circulation between users through `transfer`
4. **Event-driven**: Clients monitor events to update local Merkle Tree and wallet state

---

## 4. Encryption and Security Model

- **Commitment**: Binds amount, key, randomness, serving as state tree leaves
- **Nullifier**: Derived from note secrets, prevents double-spending
- **Note Encryption**: Uses Baby Jubjub curve ECDH → AES-GCM encryption, ensuring security and efficiency
- **Optional Disclosure**: Supports compliance through Viewing Key / Auditing Key

---

## 5. Economic Model

**Creation Fee**: Creators pay a small fee to publish their own native privacy tokens. This prevents too many users from creating unused tokens and avoids bot activity.

**Minting Fee**: Any user can issue their own native privacy tokens. Initiators can set minting fees, requiring users to pay certain fees to participate in fair minting. The platform collects 2.5% protocol fees to maintain long-term protocol development.

**Transaction Gas Costs**: Users pay regular transaction fees on-chain.

This model ensures sustainable protocol development while incentivizing developers and community.

---

## 6. Application Scenarios

**DAO Governance**: Anonymous voting, preventing collusion and opinion pressure.

**Privacy DeFi**:
- Privacy DEX (trading amounts and strategies hidden)
- Anonymous lending (collateral and lending behavior confidential)
- Shielded Yield Farming (hiding returns and positions)

**Enterprise Payments and Supply Chain Finance**: On-chain settlement while protecting trade secrets.

**Gaming and Metaverse**:
- Protect player asset privacy
- Avoid wealth disparity affecting competition
- Cross-game asset privacy circulation

---

## 7. Compliance and Regulatory Considerations

| Aspect | Mixing Protocols | Native Privacy Assets |
|--------|------------------|----------------------|
| Asset Source | Existing tokens | Native creation |
| Privacy Acquisition | Post-hoc mixing | Inherent property |
| Money Laundering Risk | High | Low |
| Regulatory Classification | High-risk tool | New asset category |
| Compliance Path | Difficult | Feasible |

The protocol can support compliance modes, such as providing **optional viewing keys** for regulatory purposes.

---

## 8. Performance and Scalability

- **60% reduction in Gas costs** Achieves up to 60% reduction in gas costs compared to traditional L1 privacy solutions that require full Merkle tree updates on-chain.
- **40% reduction in client resources** The viewTag system and tiered tree structure can lead to a 40% reduction in client-side computation and storage compared to scanning every transaction in a single, monolithic tree.
- **50% reduction in network bandwidth**
- **Parallel Finalized Transfers** In L2 environments or under ideal batching conditions for Finalized Transfers, the protocol's theoretical throughput can reach hundreds of transactions per second (TPS).

---

## 9. Technical Specifications

### 9.1 Dual-Layer Merkle Tree Architecture

**Subtree Layer**:
- Height: 10 levels
- Capacity: 1024 leaf nodes
- Purpose: Store recent notes with low-cost updates

**Root Tree Layer**:
- Height: 16 levels
- Capacity: 65,536 subtrees
- Purpose: Archive completed subtree roots

### 9.2 Proof System Performance

| Proof Type | Constraint Count | Generation Time | Verification Time | Gas Cost |
|------------|------------------|-----------------|-------------------|----------|
| MINT | ~20K | 1s | <100ms | ~250K |
| ACTIVE_TRANSFER | ~30K | 2-3s | <100ms | ~300K |
| FINALIZED_TRANSFER | ~50K | 3-4s | <100ms | ~350K |
| ROLLOVER | ~50K | 3-4s | <100ms | ~350K |

### 9.3 PV1 Address Format

```
pv1[N][T][CompressedData][Checksum]
```

- **pv1**: Protocol version identifier
- **N**: Network code (supports 58 networks)
- **T**: Address type (Standard/Disposable/Social/Temporary)
- **CompressedData**: Compressed elliptic curve public key data
- **Checksum**: 4-character FNV-1a checksum

**Compression Ratio**: 35% size reduction compared to traditional formats

### 9.4 Multi-Chain Support

| Network | Code | Status | Primary Use |
|---------|------|--------|-------------|
| Ethereum | M | Active | Mainnet deployment |
| Polygon | P | Active | Low-cost transactions |
| Arbitrum | A | Planned | L2 scaling |
| Base | B | Planned | Coinbase ecosystem |
| Reserved Slots | - | 54 slots | Future expansion |

---

## 10. Security Analysis

### 10.1 Cryptographic Security

- **Elliptic Curve**: Baby Jubjub curve compatible with zk-SNARK systems
- **Hash Function**: Poseidon hash optimized for zero-knowledge proofs
- **Encryption**: ECDH key exchange + AES-GCM authenticated encryption
- **Randomness**: Cryptographically secure random number generation

### 10.2 Protocol Security

- **Double-spending Prevention**: Nullifier uniqueness enforced by smart contracts
- **Replay Protection**: Each proof is cryptographically bound to specific inputs
- **State Integrity**: Merkle tree roots provide tamper-evident state commitment
- **Privacy Preservation**: Zero-knowledge proofs reveal no sensitive information


---

## 11. Conclusion and Vision

This protocol, through the ZRC-20 standard, pv1 multi-chain addresses, tiered Merkle trees, and efficient zk-SNARK proofs, constructs a native privacy asset layer for the Ethereum ecosystem. It represents not only technical innovation but a paradigm shift in the market: making privacy a fundamental attribute of any digital asset.

**Future directions include**:

- **L2 Rollup Integration**: Further cost reduction through Layer 2 solutions
- **Privacy Smart Contract Support**: Extension to general-purpose applications  
- **Compliance Mode Adoption**: Enabling enterprise-level adoption
- **Cross-chain Interoperability**: Seamless asset movement across networks

We believe that privacy is a necessary condition for Web3 mass adoption. This protocol serves as the foundation stone for that future.

---

## Disclaimer

This whitepaper is for technical communication and educational reference only and does not constitute investment advice. Please use this protocol within the scope permitted by local laws and regulations. Protocol development may face multiple risks including technical, market, and regulatory challenges. Please evaluate carefully.
