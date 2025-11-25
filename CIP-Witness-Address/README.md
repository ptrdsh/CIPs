---
CIP: ????
Title: Governance Metadata - Addresses of Witness Public Keys in Author Objects
Category: Metadata
Status: Proposed
Authors:
  - Alex Moser <alexander.moser@cardanofoundation.org>
Implementors:
  - Alex Moser <alexander.moser@cardanofoundation.org>
Discussions:
  - https://github.com/cardano-foundation/CIPs/pull/1101#issuecomment-3431261749
Created: 2025-11-25
License: CC-BY-4.0
---

## Abstract

This CIP extends [CIP-100 | Governance Metadata][CIP-100] to introduce a standardized `AddressOfWitnessPublicKey` property within the `authors` object. 

The `AddressOfWitnessPublicKey` property expresses which onchain identity the `publicKey` (stated with every witness) corresponds to. This enables verification of an author's identity using explorers, e.g. via identity NFTs, et al, without needing to rely on secondary publications of publicKeys, and overall provides more useful context to given witness information. 

A witness public key should be able to correspond to any "address". To enable cross-chain, or Cardano-independent identity bindings of public keys to addresses, an additional property `AddressType` is added, to specify which type of address `AddressOfWitnessPublicKey` is. Likely useful example types may be: `Cardano`, `CardanoTest`, `Midnight`, `Bitcoin`, `KERI`, `VLEI`, `Atala`, etc.. However, some of those examples would require additional `witnessAlgorithm`'s to be allowed, which is why `AddressType` values remain unspecified within this CIP. 

These two additional properties can be applied to all types of governance metadata, including governance actions, votes, DRep registrations/updates, and Constitutional Committee resignations.

## Motivation: why is this CIP necessary?

Without a standardized mechanism to bind witness public keys to on-chain identities such as addresses, the plain witness public key, and its signature, is meaningless without additional efforts of the author to make said public key known via a blog post on a publicly well known website, or other publication methods in secondary/tertiary environments (X, github, forum,...). 
Such an identity binding mechanism is also slightly relevant for replay attacks, where an attacker could just copy-paste the contents of a Governance Action and replace the pubkey and signature with its own. While this vector has been mainly addressed in CIP-169, this CIP aims at an additional layer of protection for the witnesses. 

The suggestion has been briefly discussed via https://github.com/cardano-foundation/CIPs/pull/1101#issuecomment-3431261749.

### Limitations

Due to `witnessAlgorithm` allowing either `CIP-0008` or `ed25519` at the moment, a `publicKey` can and cannot have an corresponding `AddressOfWitnessPublicKey` onchain identity. Especially for witness signatures signed via `ed25519`, the author may not have or want an address to their pubkey. 
Thus, `AddressOfWitnessPublicKey` would thus be mostly relevant for `CIP-0008` signed witnesses. 

## Specification

### The `AddressOfWitnessPublicKey` Property

The `AddressOfWitnessPublicKey` property is a new **optional** property within the `authors` object of [CIP-100][] governance metadata.
The `AddressType` property is a new **optional** property within the `authors` object of [CIP-100][] governance metadata.

#### JSON-LD Context

The `AddressOfWitnessPublicKey` and `AddressType` properties shall be defined in the JSON-LD `@context` as follows:

```json
"authors": {
            "@id": "https://github.com/cardano-foundation/CIPs/blob/master/CIP-0100/README.md#authors",
            "@container": "@set",
            "@context": {
                "did": "@id",
                "name": "http://xmlns.com/foaf/0.1/name",
                "witness": {
                    "@id": "https://github.com/cardano-foundation/CIPs/blob/master/CIP-0100/README.md#witness",
                    "@context": {
                        "witnessAlgorithm": "https://github.com/cardano-foundation/CIPs/blob/master/CIP-0100/README.md#witnessAlgorithm",
                        "publicKey": "https://github.com/cardano-foundation/CIPs/blob/master/CIP-0100/README.md#publicKey",
                        "signature": "https://github.com/cardano-foundation/CIPs/blob/master/CIP-0100/README.md#signature",
                        "AddressOfWitnessPublicKey": "https://github.com/cardano-foundation/CIPs/blob/master/CIP-0100/README.md#AddressOfWitnessPublicKey"
                        "AddressType": "https://github.com/cardano-foundation/CIPs/blob/master/CIP-0100/README.md#AddressType"

                    }
                }
            }
        }
```

##### Supported Types

**Generally**

The `AddressOfWitnessPublicKey` property does **NOT** conform to the various [CIP-116][] `address` types to enable cross-chain, or Cardano-independent identity bindings of public keys to addresses. A witness public key should be able to correspond to any "address". 

- `RewardAddress`
- `ByronAddress`
- `EnterpriseAddress`
- `BaseAddress`
  
See [CIP-116 cardano-conway.json](https://github.com/cardano-foundation/CIPs/blob/master/CIP-0116/cardano-conway.json) for complete type definitions.

... more here? 
what about addressTypes? 

### Verification Process

Tools **SHOULD** implement the following verification when processing governance actions or votes:

1. **Author Address Check**: Validate if the author public keys are corresponding to the respective addresses, using the `AddressType` to reconstruct the address from a pubkey. 
2. **Alert**: If there are invalid author signatures and/or the `AddressOfWitnessPublicKey` does not match to the `publicKey`, warn the user

### Examples

```json
"authors":
    [
        {
            "name": "Cardano Foundation",
            "witness":
            {
                "witnessAlgorithm": "CIP-0008",
                "publicKey": "a535adf6786e2f8cd4bb7451e9f5b6368a037b8ed42f35b8446fb8946b7d153d",
                "signature": "845846a201276761646472657373583901acb2792c1cb10724327852ac8d03f870732c5bc8e10c22a8b80991411ef0ec112a3c73539566bde6b6abe41bb79191c1baa69edfcb0581f2a166686173686564f45820365b88ab14451ab057845f697ab4c7aa78ce82c26a6dad092db0e3fe48c468e858400825fe66ad9f2f9f003b345336004fd9a4e6d1c9d4b3677e47563b23132fc3879829fc2e2c300cf459cb92c5282daa46230122055528166253715e74024ae20c",
                "AddressOfWitnessPublicKey": "addr1qxkty7fvrjcswfpj0pf2ergrlpc8xtzmersscg4ghqyezsg77rkpz23uwdfe2e4au6m2heqmk7gersd6560dljc9s8eqr6r3v3",
                "AddressType": "Cardano"
            }
        }
    ]
```

## Rationale

By binding addresses to witness public keys within the witness metadata body using the `AddressOfWitnessPublicKey` and `AddressType` properties, verifying authors does not need to rely on secondary publications of publicKeys any longer. 
A blockchain can be used to identify an author through that author's/public key's/address's onchain actions, including identity NFTs. 

IF desired, `AddressOfWitnessPublicKey` and `AddressType` could also enable tools to reference a self maintained onchain smart contract registry of known public keys - something that was not feasible so far, due to manual maintenance requirements. 

### Why Extend CIP-100?

[CIP-100][] provides the base structure for governance metadata.
This extension adds security-critical information while maintaining backward compatibility.

### Why Use CIP-116 Encoding?

[CIP-116][] provides standardized JSON encoding for Cardano domain types.

### Why Optional?

The `AddressOfWitnessPublicKey` and `AddressType` properties are optional to maintain backward compatibility and to keep support for possible non-blockchain public keys and signing Algorithms (such as `ed25519`).

This CIP is fully backward compatible:

- **Existing Metadata**: Governance actions without `AddressOfWitnessPublicKey` and `AddressType` remain valid
- **Parsers**: Parsers ignoring `AddressOfWitnessPublicKey` and `AddressType` continue to work
- **Validators**: Validators not checking `AddressOfWitnessPublicKey` and `AddressType` continue to work

## Open Questions

- Should allowed `AddressType`s values be specified? Or more generally: should other types than Cardano be allowed at all? Is CardanoTest its own type?
- Is it senseful that tools verify whether the address matches the pubkey? This sounds error-prone.
- Are there any other anchors or schemas or standards this should be referenced in cleanly?

## Path to Active

### Acceptance Criteria

This CIP is considered **Active** when:

1. The specification is merged into the CIPs repository
2. At least two governance metadata authoring tools implement support
3. At least one verification tool implements on-chain comparison
4. Documentation and examples are available

### Implementation Plan

- todo

## Copyright

This CIP is licensed under [CC-BY-4.0][].

[CIP-100]: https://github.com/cardano-foundation/CIPs/blob/master/CIP-0100/README.md
[CIP-108]: https://github.com/cardano-foundation/CIPs/blob/master/CIP-0108/README.md
[CIP-116]: https://github.com/cardano-foundation/CIPs/blob/master/CIP-0116/README.md
[CC-BY-4.0]: https://creativecommons.org/licenses/by/4.0/legalcode
