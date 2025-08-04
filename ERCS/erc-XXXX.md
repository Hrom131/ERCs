---
eip: XXXX
title: Simplified Payment Verification Gateway
description: Trustless singleton contract for verification of Bitcoin transactions through block headers
author: Artem Chystiakov (@arvolear), Oleh Komendant (@Hrom131)
discussions-to: 
status: Draft
type: Standards Track
category: ERC
created: 2025-07-31
---

## Abstract

Introduce a singleton contract for on-chain verification of transactions that happened on Bitcoin. The contract is available at "0xTODO", acting as a trustless Simplified Payment Verification (SPV) gateway where anyone can submit Bitcoin block headers to. The gateway maintains the mainchain of blocks and allows the existence of Bitcoin transactions to be verified via Merkle proofs.

## Motivation

Ethereum's long term mission has always been to revolutionize the financial world through decentralization, trustlessness, and programmable value enabled by smart contracts. Many gems have been discovered so far, including the renaissance of Decentralized Finance (DeFi), Real-World Assets (RWA), and privacy-preserving protocols.

However, one gem has been unreachable so far -- Bitcoin. Due to its extremely constrained programmability, one can only hold and transfer bitcoins in a trustless manner. This EIP tries to expand its capabilities by laying a solid foundation for bitcoins to be also used in various EVM-based DeFi protocols, unlocking the whole new trillion dollar market.

The singleton SPV gateway contract defined in this proposal acts as a trustless one-way bridge between Bitcoin and Ethereum, already enabling use cases such as using _native_ BTC as a lending collateral for stablecoin loans. Moreover, with the recent breakthroughs in the BitVM technology, the full-fledged, ownerless two-way bridge may soon become a reality, powering the permissionless and wrapless issuance of BTC on Ethereum.

## Specification

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

### General

#### Bitcoin Block Header Structure

In Bitcoin, each block contains a header, which has a fixed size of 80 bytes and follows the following structure:

| Field          | Size     | Format             | Description                                                          |
| :------------- | :------- | :----------------- | :------------------------------------------------------------------- |
| Version        | 4 bytes  | little-endian      | The version number of the block.                                     |
| Previous Block | 32 bytes | natural byte order | The block hash of a previous block this block is building on top of. |
| Merkle Root    | 32 bytes | natural byte order | A fingerprint for all of the transactions included in the block.     |
| Time           | 4 bytes  | little-endian      | The current time as a Unix timestamp.                                |
| Bits           | 4 bytes  | little-endian      | A compact representation of the current target (difficulty).         |
| Nonce          | 4 bytes  | little-endian      | A 32-bit number which miners compute to find a valid block hash.     |

The fields within the block header are sequentially ordered as presented in the table above.

#### Difficulty Adjustment Mechanism

Bitcoin's Proof-of-Work (PoW) consensus mechanism relies on a dynamic **difficulty target**. This target's initial value was `0x00000000ffff0000000000000000000000000000000000000000000000000000`, which also serves as its minimum threshold (the "minimum difficulty").

The `target` is recalculated approximately every two weeks, specifically every **2016 blocks**, a period commonly referred to as a **difficulty adjustment period** or **retargeting period**.

The expected duration for each adjustment period is **1,209,600 seconds** (2016 blocks * 10 minutes/block). The new `target` value is derived by multiplying the current `target` by the ratio of the actual time taken to mine the preceding 2016 blocks to this expected duration.

To prevent drastic fluctuations, the adjustment multiplier is capped, preventing it from increasing the difficulty by more than 4x or decreasing it by less than 1/4x.

#### Block Header Validation Rules

For a Bitcoin block header to be considered valid and accepted into the chain, it MUST adhere to the following consensus rules:

1.  **Chain Cohesion**: The `Previous Block` hash field MUST accurately reference the hash of a valid block that is present within the SPV gateway's known set of valid block headers.

2.  **Timestamp Rules**:
    * The `Time` field MUST be strictly greater than the **Median Time Past (MTP)** of the previous 11 blocks.
    * The `Time` field MUST NOT be more than 2 hours in the future relative to the validating node's network-adjusted time.

3.  **PoW Constraint**: When the block header is hashed twice using `SHA256`, the resulting hash MUST be less than or equal to the current `target` value, as derived from the difficulty adjustment mechanism.

#### Canonical Chain Definition

Bitcoin's **canonical chain** is determined not merely by its length, but by possessing the highest **cumulative PoW** among all valid competing chains. This cumulative work represents the total computational effort expended to mine all blocks within a specific chain.

The **work** contributed by a single block is inversely proportional to its `target` value. Specifically, the work of a block can be calculated as $1/target$. The `target` value for a block is derived from its `Bits` field.

The total cumulative work of a chain is the sum of the work values of all blocks within that chain. A block is considered part of the canonical chain if it extends the chain with the greatest cumulative PoW. This mechanism is crucial for resolving forks and maintaining a single, agreed-upon history of transactions.

### SPV Gateway

The `SPVGateway` contract MUST provide a permissionless mechanism for its initial bootstrapping. This mechanism MUST allow for the submission of a valid Bitcoin block header, its corresponding block height, and the cumulative PoW up to that block, without requiring special permissions.

All fields within the `BlockHeaderData` MUST be converted to **big-endian** byte order for internal representation and processing within the smart contract.

Furthermore, the `SPVGateway` MUST implement the following interface, which MAY be extended to provide additional functionality:

```solidity
pragma solidity ^0.8.0;

/**
 * @notice Interface for an SPV (Simplified Payment Verification) gateway contract.
 */
interface ISPVGateway {
    /**
     * @notice Represents the essential data contained within a Bitcoin block header
     * @param prevBlockHash The hash of the previous block
     * @param merkleRoot The Merkle root of the transactions in the block
     * @param version The block version number
     * @param time The block's timestamp
     * @param nonce The nonce used for mining
     * @param bits The encoded difficulty target for the block
     */
    struct BlockHeaderData {
        bytes32 prevBlockHash;
        bytes32 merkleRoot;
        uint32 version;
        uint32 time;
        uint32 nonce;
        bytes4 bits;
    }

    /**
     * @notice Represents the data of a block
     * @param header The parsed block header data
     * @param blockHeight The block height
     */
    struct BlockData {
        BlockHeaderData header;
        uint256 blockHeight;
    }

    /**
     * @notice Possible directions for hashing:
     * Left: computed hash is on the left, sibling hash is on the right.
     * Right: computed hash is on the right, sibling hash is on the left.
     * Self: node has no sibling and is hashed with itself
     * */
    enum HashDirection {
        Left,
        Right,
        Self
    }

    /**
     * MUST be emitted whenever the mainchain head changed (e.g. in the `addBlockHeader`, `addBlockHeaderBatch` functions)
     */
    event MainchainHeadUpdated(
        uint256 indexed newMainchainHeight,
        bytes32 indexed newMainchainHead
    );

    /**
     * MUST be emitted whenever the new block header added to the SPV contract state
     * (e.g. in the `addBlockHeader`, `addBlockHeaderBatch` functions)
     */
    event BlockHeaderAdded(uint256 indexed blockHeight, bytes32 indexed blockHash);

    /**
     * @notice OPTIONAL Function that adds a batch of the block headers to the contract.
     * Each block header is validated and added sequentially
     * @param blockHeaderRawArray An array of raw block header bytes
     */
    function addBlockHeaderBatch(bytes[] calldata blockHeaderRawArray) external;

    /**
     * @notice Adds a single raw block header to the contract.
     * The block header is validated before being added
     * @param blockHeaderRaw The raw block header bytes
     */
    function addBlockHeader(bytes calldata blockHeaderRaw) external;

    /**
     * @notice Validates a given block hash and returns its mainchain status and confirmation count
     * 
     * @param blockHash The hash of the block to validate
     * @return isInMainchain True if the block is in the mainchain, false otherwise
     * @return confirmationsCount The number of blocks that have been mined on top of the validated block
     */
    function validateBlockHash(bytes32 blockHash) external view returns (bool, uint256);

    /**
     * @notice Verifies that given txid is included in the specified block
     * @param blockHash The hash of the block in which to verify the transaction
     * @param txid The transaction id to verify
     * @param merkleProof The array of hashes used to build the Merkle root
     * @param directions The array indicating the hashing directions for the Merkle proof
     */
    function verifyTx(
        bytes32 blockHash,
        bytes32 txid,
        bytes32[] memory merkleProof,
        HashDirection[] calldata directions
    ) external view returns (bool);

    /**
     * @notice Returns the Merkle root of a given block hash.
     * This function retrieves the Merkle root from the stored block header data
     * @param blockHash The hash of the block
     * @return The Merkle root of the block
     */
    function getBlockMerkleRoot(bytes32 blockHash) external view returns (bytes32);

    /**
     * @notice Returns the basic block data for a given block hash.
     * This includes the block header and its height
     * @param blockHash The hash of the block
     * @return The basic block data
     */
    function getBlockData(bytes32 blockHash) external view returns (BlockData memory);

    /**
     * @notice Returns the hash of the current mainchain head.
     * This represents the highest block on the most accumulated work chain
     * @return The hash of the mainchain head
     */
    function getMainchainHead() external view returns (bytes32);

    /**
     * @notice Returns the height of the current mainchain head.
     * This represents the highest block number on the most accumulated work chain
     * @return The height of the mainchain head
     */
    function getMainchainBlockHeight() external view returns (uint256);

    /**
     * @notice Returns the block height for a given block hash
     * This function retrieves the height at which the block exists in the chain
     * @param blockHash The hash of the block
     * @return The height of the block
     */
    function getBlockHeight(bytes32 blockHash) external view returns (uint256);

    /**
     * @notice Returns the block hash for a given block height.
     * This function retrieves the hash of the block from the mainchain at the specified height
     * @param blockHeight_ The height of the block
     * @return The hash of the block
     */
    function getBlockHash(uint256 blockHeight_) external view returns (bytes32);

    /**
     * @notice Checks if a block exists in the contract's storage.
     * This function verifies the presence of a block by its hash
     * @param blockHash The hash of the block to check
     * @return True if the block exists, false otherwise
     */
    function blockExists(bytes32 blockHash) external view returns (bool);

    /**
     * @notice Checks if a given block is part of the mainchain.
     * This function determines if the block is on the most accumulated work chain
     * @param blockHash The hash of the block to check
     * @return True if the block is in the mainchain, false otherwise
     */
    function isInMainchain(bytes32 blockHash) external view returns (bool);
}
```

The `addBlockHeader` function MUST:
- Validate that the submitted raw block header has a fixed size of 80 bytes.
- Enforce all block header validation rules as specified in the 'Block Header Validation Rules' section.
- Integrate the new block header into the known chain by calculating its cumulative PoW and managing potential chain reorganizations as defined in the 'Canonical Chain Definition' section.
- Emit a `BlockHeaderAdded` event upon successful addition of the block header.
- Emit a `MainchainHeadUpdated` event if the canonical chain was updated.

## Rationale

The design of this EIP for an on-chain Bitcoin SPV gateway aims to securely bridge the Bitcoin and Ethereum ecosystems without trusted intermediaries, maximizing decentralization, security, and usability.

To enable trustless verification of Bitcoin transactions on Ethereum, the `SPVGateway` contract MUST be entirely permissionless. Therefore, the initialization of the initial block MUST be as transparent and trust-minimized as possible, avoiding reliance on any privileged entity.

The following initialization options were considered:

1.  **Hardcoding the Bitcoin genesis block:** This approach is the simplest for contract deployment as it embeds the initial state directly in the code. While offering absolute immutability of the starting point, it limits flexibility, as the client can only begin verifying from the very first Bitcoin block and cannot be bootstrapped from a more recent block height.
2.  **Initialization from an arbitrary block height by trusting provided cumulative work and height:** This EIP adopts this method for its primary initialization mechanism. It allows for flexible bootstrapping of the SPV gateway from any valid historical block, significantly reducing the initial gas cost and time required compared to validating an entire chain from genesis. While this implies trust in the initial submitted values, it's a common practice for bootstrapping light clients and can be secured via off-chain mechanisms for initial validation (e.g., community-verified checkpoints).
3.  **Initialization with Zero-Knowledge Proof (ZKP) for historical correctness:** This advanced method involves proving the entire history of Bitcoin up to a specific block using ZKP.

Upon submission or internal processing of the raw block header, all fields within the `BlockHeaderData` structure MUST be converted to big-endian byte order. This ensures optimal compatibility and efficiency with Ethereum's native EVM arithmetic and cryptographic algorithms, which primarily operate on big-endian, unlike Bitcoin's native little-endian integer serialization.

The `addBlockHeader` function is designed to accept any valid Bitcoin block headers, even if not currently part of the canonical chain. This is crucial as Bitcoin's consensus rules allow for chain reorganizations of arbitrary depth. A rigid SPV gateway tracking only the immediate canonical head risks vulnerability during forks.

By maintaining multiple valid forks and tracking their cumulative Proof-of-Work, the `SPVGateway` enhances robustness against chain reorganizations. Consequently, the `SPVGateway` contract **does not** include internal block 'finalization' parameters. This determination is left to consuming protocols, promoting modularity and allowing each to define its own security thresholds.

The inclusion of an OPTIONAL `addBlockHeaderBatch` function offers significant gas optimizations. For batches exceeding 11 blocks, `Median Time Past (MTP)` can be calculated using timestamps from memory, substantially reducing storage reads and transaction costs.

The `verifyTx` function's `directions_` parameter is an integral part of a standard Merkle Proof from a full Bitcoin node. It provides explicit instructions for hashing order during Merkle Proof verification, ensuring accurate on-chain replication of the off-chain Merkle tree construction and cryptographic integrity.

## Backwards Compatibility

This EIP is fully backwards compatible.

## Reference Implementation

A reference implementation of the `SPVGateway` contract, which adheres to this standard, is available [here](../assets/erc-XXXX/contracts/SPVGateway.sol).

The following supporting libraries are provided to facilitate the implementation of a compliant `SPVGateway`:

- [`BlockHeader`](../assets/erc-XXXX/contracts/libs/BlockHeader.sol): A utility for parsing a raw, 80-byte block header into a structured format for on-chain processing.
- [`TargetsHelper`](../assets/erc-XXXX/contracts/libs/TargetsHelper.sol): A helper library containing functions for converting the `Bits` field to the `target` value and back, as well as for calculating the new difficulty target during a retargeting period.
- [`TxMerkleProof`](../assets/erc-XXXX/contracts/libs/TxMerkleProof.sol): A library for performing cryptographic verification of Merkle Proofs, which are used to prove the inclusion of a Bitcoin transaction in a block.

> Please note that the reference implementation depends on the `@openzeppelin/contracts v5.2.0` and `solady v0.1.23`.

## Security Considerations

Among potential vulnerabilities, the following can be noted.

The security of the `SPVGateway` is directly dependent on the security of Bitcoin's underlying PoW consensus. A successful 51% attack on the Bitcoin network, would allow an attacker to submit fraudulent block headers that would be accepted by the contract, thereby compromising its state.

Unlike other blockchain systems with deterministic finality, Bitcoin's consensus is probabilistic. The `SPVGateway` contract is designed to handle chain reorganizations of arbitrary depth, but it cannot prevent them. As a result, transactions included in a block may not be permanently final. All dApps and protocols relying on this contract MUST implement their own security policies to determine a sufficient number of block confirmations before a transaction is considered 'final' for their specific use case.

While the `addBlockHeader` function is permissionless and validates each new header cryptographically, the contract's initial state (its starting block header, height, and cumulative PoW) is a point of trust. The integrity of the entire chain history within the contract is built upon the correctness of this initial data. Although the EIP's design allows for flexible bootstrapping, the responsibility for verifying the initial state falls on the community and the dApps that choose to use a specific deployment of the `SPVGateway`.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
