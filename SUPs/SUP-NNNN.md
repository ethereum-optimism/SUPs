---
sup: NNNN
title: Batched Commitments for AltDA-based OP Stack Chains
champion:
author: alvarius (@alvrs), tdot (@tchardin), vdrg (@vdrg)
created: 2025-01-23
eligible: YYYY-MM-DD
requires:
status: Draft
---

## Description

A protocol upgrade that enables batching multiple DA commitments into a single L1 transaction for AltDA-based OP Stack chains. This introduces a new commitment type (`BatchedCommitmentType`) that allows multiple sub-commitments to be submitted in a single transaction while preserving individual challengeability, resulting in significant gas cost savings by sharing the base transaction cost across multiple commitments.

## Motivation and Impact

AltDA-based OP Stack chains currently submit DA commitments to L1 individually, with each commitment requiring its own transaction. This approach incurs separate base transaction costs (21,000 gas on Ethereum) for each commitment. This inefficiency leads to higher operational costs for chain operators and, ultimately, higher fees for users.

The implementation of batched commitments will provide several key benefits:

1. **Cost Reduction**: By combining multiple commitments into a single L1 transaction, some chains will save significantly on L1 transaction costs, as only a single 21,000 gas fee will be paid per batch instead of per commitment (excluding the additional calldata costs).
2. **Preserved Security**: Each sub-commitment within a batch remains individually challengeable, maintaining the security properties of the existing system.
3. **Minimal Infrastructure Changes**: The proposal requires minimal changes to the existing derivation pipeline, as the AltDA Data Source will handle the decoding of batched commitments transparently.

## Design

### Overview

The design introduces a new commitment type while maintaining compatibility with existing commitment types and challenge mechanisms. Here are the key design decisions:
1. **New Commitment Type**
    - Introduce `BatchedCommitmentType = 2` to represent batched commitments
    - Support both existing commitment types as sub-commitments:
        - `Keccak256CommitmentType (0)`: 32-byte fixed-size
        - `GenericCommitmentType (1)`: Variable-size
2. **Encoding Structure**
    - Length-prefixed format for sub-commitments enabling parsing by the AltDA data source
    - Preserved derivation version for compatibility
3. **Derivation pipeline**
    - Single L1 transaction contains multiple sub-commitments
    - AltDA Data Source decodes and processes each sub-commitment independently
    - Existing challenge mechanisms remain unchanged


## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Batched Commitment Encoding

The L1 transaction data MUST follow this format:

```
[1 byte:  derivation_version = 0x01]
[1 byte:  commitment_type    = 0x02]    // BatchedCommitmentType
[1 byte:  subcommitment_type = 0x00 or 0x01]
[variable length: sub_commitments]

```

Each sub-commitment MUST be encoded as:

```
[2 bytes: comm_len (big-endian)]
[comm_len bytes: commitment_data]

```

### Example Encoding

For two Keccak256 sub-commitments (each 32 bytes), the L1 transaction data would be structured as follows:

| Field | Example (Hex) | Explanation |
| --- | --- | --- |
| derivation_version | 0x01 | OP Stack derivation version |
| batched_commitment_type | 0x02 | Indicates a batched commitment |
| subcommitment_type | 0x00 | Keccak256CommitmentType |
| comm1_len | 0x0020 | length = 32 |
| comm1_data | [32 bytes] | First commitment hash |
| comm2_len | 0x0020 | length = 32 |
| comm2_data | [32 bytes] | Second commitment hash |


### Implementation Requirements

1. The derivation pipeline MUST:
    - Detect BatchedCommitmentType (0x02) and process accordingly
    - Extract and validate each sub-commitment independently
    - Maintain existing challenge mechanisms for each sub-commitment
2. The batcher MUST:
    - Support configuration for consuming multiple frames when using Batched Commitments (similar to current implementation for Blobs DA)
    - Submit each frame to the DA Server independently
    - Construct a valid batched commitment from returned commitments

### Alternative Approaches Considered

#### Alternative Encoding for Keccak256 Sub-commitments

For batches containing only Keccak256 commitments, a more efficient encoding could omit the length prefix since these commitments are always 32 bytes. While this optimization would reduce gas costs, it has not been included in the main specification since Keccak256 Commitments are expected to be deprecated in the future.

#### Aggregating Commitments in DA Server

An alternative approach for constructing Batched Commitments is to do it in the DA Server. The batcher would send the concatenated frames to the server and the server would be in charge of parsing and constructing the resulting commitment.

However, to keep sub-commitments individually challengeable (and not change the existing logic related to challenges) the server would need to store the input frame for each sub-commitment independently, which would break the server's API semantics.

## Backwards Compatibility

This upgrade introduces a new commitment type while maintaining full compatibility with existing commitment types. No changes are required for existing challenge mechanisms or the DA Server specification. The AltDA data source will need to be updated to handle the new BatchedCommitmentType, but can maintain existing handling for other commitment types.

## Failure Modes Analysis

Batched commitments are subject to same failure modes as existing commitments.

---

## Progress Checklist

_To be updated only by SUP editors_.

Prior to SUP merge:

- [ ] There is a named champion for this SUP
- [ ] All sections of SUP completed and reviewed by SUP editor
- [ ] Design review done and approved
- [ ] SUP number has been assigned

Prior to SUP inclusion:

- [ ] Specification review done and approved
- [ ] Implemented in a client and run on an Alphanet (where applicable)
- [ ] Implemented in all relevant clients and run on an upgrade Betanet (where applicable)
- [ ] FMA security reviewed and actions completed (audit, run books, etc.)
- [ ] Included in a governance proposal and has passed governance

---

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
