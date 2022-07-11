# Current Development Phase: Arbitrary Secrets Proof of Concept

In this phase, we want to build a general-purpose, human-centric system that stores and retrieves secrets on a single server. This proof of concept (PoC) will include basic cryptography to realize an end-to-end encrypted storage of arbitrary secrets. This constitutes a fundamental building block of our larger imagined digital asset management system. 

## Assumptions and Requirements
- The primary stakeholder is a human with secrets. We call this human an _asset owner_.
- All workflows MUST be initiated from a local device owned and operated by a human, namely the asset owner.
- There is only one key server. This key server is either run on external cloud infrastructure or hosted locally by a service provider.


## Workflows
We provide sketches of the generation, storage, and use of arbitrary secrets below, focusing on the high-level system flows initiated by the asset owner. 

1. The asset owner may _register_ with the key server via an asymmetric password-authenticated key exchange protocol.
1. The asset owner may _open an authenticated session_ with the key server via an asymmetric password-authenticated key exchange protocol. 
    1. Once established, the asset owner can make operation requests on secrets to the key server. 
    1. The session MUST provide a secure channel that satisfies mutual authentication, confidentiality, and integrity.
1. The asset owner may _close_ a session that the asset owner previously established with the key server.
1. The asset owner can _generate_ a secret locally.
    1. This secret MUST be randomly generated according to the uniform distribution.
    1. This secret should default to 256-bits.
1. The asset owner can _store_ a secret, either on the key server or locally.
    1. Storage on a key server MUST occur via a mutually authenticated channel that satisfies confidentiality and integrity. 
    1. Local storage on the user's device should be secure.
1. The asset owner can _retrieve_ a secret from the key server. A retrieved secret can be:
    1. Stored locally in secure storage on the user's device.
    1. Copied to the system clipboard.
1. The asset owner may _import_ a secret to the key server in an appropriate format. 
    1. [TODO: define or reference standard format.]
1. The asset owner may _export_ the secret from the key server. The export format for the key SHOULD allow for easy transfer of the key material to another digital asset management system, i.e., secrets should be portable.
1. The asset owner may _audit_ the operations performed by the key server on a given secret. This allows the asset owner to retrieve a log of operations from the key server.

## Cryptographic Protocol and Implementation Dependencies
1. We instantiate the asymmetric password-based authenticated key exchange protocol with OPAQUE. We are currently using [opaque-ke](https://docs.rs/opaque-ke/2.0.0-pre.2/opaque_ke/index.html), which implements [version 08 of the IETF RFC](https://datatracker.ietf.org/doc/draft-irtf-cfrg-opaque/08/). This library is under active development, as is the IETF draft. An earlier release of this repository has been audited by NCC Group in June 2021. 
    1. [TODO: Add any configuration details relevant to a cryptographic audit.]
1. TLS 1.3. 
    1. [TODO: add implementation dependency information.]

## Expected Outcomes
A working command-line demo with cryptography for handling arbitrary secrets. 