---
eip: ???
title: Validation Request Message
description: Message from EVM to Beacon chain, which unlocks Smart Contract Based Witness Encryption.
author: Leona Hioki(@leohio), Sora Suegami(SoraSuegami), Observed by Aayush G(@divide-by-0)
discussions-to: 
status: Not revealed
type: Standards Track
category: Core
created: 2023-11-20
requires: 4844
---

# Abstract
Introduction of Validation Request Messages for Beacon chain validators to sign into both EVM and Beacon chain. Currently, validators in the Beacon chain use BLS signatures to sign attestations. This proposal expands the signing target from unpredictable attestations to a hash of two elements: a value with deterministic randomness and data generated within smart contracts. This value ensures non-duplicity of double-voting attestations and can be verified by validators. Additionally, there is zero additional pairing computation cost for validators.
# Motivation
The ultimate goal of this proposal is to enable the Cipher Controller Contract (3C), achievable through the use of Signature-Based Witness Encryption (SWE) [1][2] as per this EIP. SWE allows the Ethereum security model to be utilized for controlling off-chain information. Unlike regular Witness Encryption, SWE can be constructed using pairings alone, making decryption computationally feasible. The ability to maintain or decrypt off-chain information while keeping it confidential, based on the state conditions of smart contracts, opens up both envisioned and unexplored applications for what is referred to as Cipher Controller Contract (3C).
Already widely discussed in the Ethereum space are applications such as interoperability with other chains including Bitcoin, and SWE could potentially aid in fundamentally resolving MEV issues. SWE is thought to enable encryption of mempools for 4337, Rollups, etc. This could further evolve L1 sequenced in Based Rollups, transferring MEV rights of each Rollup to Layer 1, and enabling L1 encrypted. In addition to SWE functioning as a solution to these already debated issues, the nature of 3C in manipulating off-chain information holds the promise of unlocking entirely new applications.


# Specification:

```
additional constant parameter:
UNIQUE_ID: constant 32 bytes

additional parameters in execution payload:
inside_message_hash_array: array[32 bytes]

additional opcode in EVM

name: VALREQ 
description: validate request
gas: 20000
stack_input: message_hash(32 bytes)
stack_output: none
operation: emitting keccak256(UNIQUE_ID, stack_input) and letting it get signed by validators.

```

additional function on the validator-side [4][5]:
https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/validator.md

```
def sign_inside_message(inside_message_hash: 32bytes) -> BLS_signature:
     return bls.Sign(privkey,inside_message_hash)
```


override in validator-side codes: 

https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#signedbeaconblock

```
class SignedBeaconBlock(Container): 
    message: BeaconBlock 
    signature: BLSSignature 
    # Need to add this array of BLSsignature for each inside massage. 
    signatures: array[BLSSignature]

```

https://github.com/ethereum/annotated-spec/blob/master/phase0/beacon-chain.md#state-transition

```
def verify_block_signature(state: BeaconState, signed_block: SignedBeaconBlock) -> bool:
    proposer = state.validators[signed_block.message.proposer_index] 
    signing_root = compute_signing_root(signed_block.message, get_domain(state, DOMAIN_BEACON_PROPOSER)) 
    # Need to make message_array here 
    message_array = signed_block.message.inside_message_hash_array + [signing_root] 
    # Need to replace bls.Verify by bls.AggregateVerify 
    return bls.AggregateVerify(proposer.pubkey, message_array, signed_block.signatures)
```


Note: Unlike signatures for attestations, SignedBeaconBlock.signatures do not need to be permanently stored in the Beacon chain block. It would be better to store them in 4844’s blob.

# Security
Regarding the safety of validator stakes, even if the inside message digest contains potentially dangerous information, it is hashed with a fixed, random unique ID, so the probability of the inside message hash leading to a Layer 1 double vote signature is virtually zero. Furthermore, unless intentionally involving this unique ID in a non-Ethereum chain, it remains safe even outside of Ethereum.
The security related to the decryption of 3C depends on the security of the smart contract itself, similar to traditional decentralized applications. If the smart contract functions correctly, the resulting messages are decrypted by BLS signatures of Beacon chain Validators, who have verified the validity of these messages. Conversely, if a bug in the 3C contract code inadvertently causes a message to be output under unintended conditions, it may be decrypted under incorrect conditions.
Drawback:
There is an increased burden on the Consensus Layer. The frequency of these message signatures will correspond to the number of times this new opcode is called in a block. However, as the signatories for the BLS verification are strictly the same, it is possible to use BLS aggregate verification to validate multiple message signatures, including attestations, in one go. Therefore, in total, only n Merkle proofs and n BLS signatures are added, with no additional pairing required.

# SWE’s Benchmark
The benchmark for Threshold SWE, which will be used for encryption/decryption off the Blockchain, currently assumes a 2/3 threshold. With N=2000, encryption takes about 60 seconds, and decryption takes about 350 seconds. For N=500, encryption takes approximately 10 seconds and decryption around 20 seconds. [1] This will depend on how many of Ethereum's validators are randomly sampled for each use case to keep the figures realistic, and whether the technological advancement of SWE progresses as rapidly as ZKP. The reason random sampling is feasible is that, in most cases, information about the ciphertext or its threshold conditions is off-chain and not disclosed to the validators.

# Reference:

About SWE:
[1] https://eprint.iacr.org/2022/433.pdf
[2] https://eprint.iacr.org/2022/499.pdf

About Blob:
[3] https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4844.md

About Beacon Chain:
[4] https://github.com/ethereum/annotated-spec/blob/master/phase0/beacon-chain.md
[5] https://github.com/ethereum/annotated-spec/blob/master/merge/beacon-chain.md

