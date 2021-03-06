# Arbitrary Secrets Proof of Concept 

In this phase, we want to build a general-purpose, human-centric system that stores and retrieves secrets on a single server. This proof of concept (PoC) will include basic cryptography to realize an end-to-end encrypted storage of arbitrary secrets. This constitutes a fundamental building block of our larger imagined digital asset management system. 

## Assumptions and Requirements
- The primary stakeholder is a human with secrets. We call this human an _asset owner_.
- All workflows MUST be initiated from a local device owned and operated by a human, namely the asset owner.
- There is only one key server. This key server is either run on external cloud infrastructure or hosted locally by a service provider.

## System Architecture
See the [Systems Architecture page](systems-architecture.md) for details.

There are two basic components: a [local client](systems-architecture.md#local_client) (which runs on an asset owner device and initiates all workflows) and a [key server](systems-architecture.md#key_server) (which provides the storage and retrieval of arbitrary secrets). These components communicate over secure channels at all times. 

## Workflows
We provide sketches of the basic generation, storage, and use of arbitrary secrets below. These flows are initiated by the asset owner. 

### Setting up secure channels with the key server
1. The asset owner may _register_ with the key server via an asymmetric password-authenticated key exchange protocol.
    1. Registration MUST occur via a channel that satisfies authentication of the server, confidentiality, and integrity.
1. The asset owner may _open an authenticated session_ with the key server via an asymmetric password-authenticated key exchange protocol. 
    1. Once established, the asset owner can make operation requests on secrets to the key server. 
    1. The session MUST provide a secure channel that satisfies mutual authentication, confidentiality, and integrity.
1. The asset owner may _close_ a session that the asset owner previously established with the key server.

### Operations on arbitrary secrets
1. The asset owner can _generate_ a secret locally.
    1. This secret MUST be randomly generated according to the uniform distribution.
    1. This secret should default to 256-bits.
1. The asset owner can _store_ a secret. By default the secret is stored both locally and at the key server.
    1. Storage on a key server MUST occur via a mutually authenticated channel that satisfies confidentiality and integrity. 
    1. Local storage on the user's device should be secure.
    1. An additional functionality that allows the user to store secrets on the remote key server only WILL be added in the future.
    1. An additional functionality that allows the user to store secrets on the local device only may be added in the future. and on the remote key server only may be added in the future.
1. The asset owner can _retrieve_ a secret from the key server. 
    1. Retrieval from a key server MUST occur via a mutually authenticated channel that satisfies confidentiality and integrity. 
    1. The key retrieved MUST be a key associated to the authenticated user.
    1. A retrieved secret can be:
        1. Stored locally in secure storage on the user's device.
        1. Copied to the system clipboard.
1. The asset owner may _import_ a secret to the system in an appropriate format. 
    1. The default expected format is as bytes of the form ``len || secret``, where `len` is 1 byte that represents the length of the secret `secret` in bytes.
    1. The implementation may define other acceptable import formats.
    1. The user may choose to store the imported secret locally or on the key server.
    1. Any communication with the key server that occurs in serving an import request MUST occur via a mutually authenticated channel that satisfies confidentiality and integrity.
1. The asset owner may _export_ the secret from the system. The export format for the key SHOULD allow for easy transfer of the key material to another digital asset management system, i.e., secrets should be portable.
    1. The default expected format is as bytes of the form ``len || secret``, where `len` is 1 byte that represents the length of the secret `secret` in bytes.
    1. The implementation may define other acceptable import formats.
    1. Any communication with the key server that occurs in serving an export request MUST occur via a mutually authenticated channel that satisfies confidentiality and integrity. 
1. The asset owner may _audit_ the operations performed by the key server on a given secret. This allows the asset owner to retrieve a log of operations from the key server.
    1. Audit log retrieval MUST occur via a mutually authenticated channel that satisfies confidentiality and integrity.
    1. Audit logs should be portable, i.e., easily exportable from the system.
    1. Non-normative note: Given the above properties, we do not achieve integrity of the audit logs in the presence of a cheating key server. Future work may address this concern.

## Cryptographic Protocol and Implementation Dependencies

- We instantiate the asymmetric password-based authenticated key exchange protocol with OPAQUE. We are using [opaque-ke](https://docs.rs/opaque-ke/2.0.0-pre.3/opaque_ke/index.html), which currently implements [version 09 of the IETF RFC](https://datatracker.ietf.org/doc/draft-irtf-cfrg-opaque/09/). This library is under active development, as is the IETF draft. An earlier release of this repository has been audited by NCC Group in June 2021. 
    - This implementation relies on [voprf](https://github.com/novifinancial/voprf), which is tracking [the IETF RFC on OPRFs](https://datatracker.ietf.org/doc/draft-irtf-cfrg-voprf/).
    - As both of the above RFCs are in flux, we expect ongoing updates.
    - Following the terminology of the RFC, we use the following OPAQUE-3DH configuration: OPRF(ristretto255, SHA-512), HKDF-SHA-512, HMAC-SHA-512, SHA-512, ristretto255, with Argon2 as the KSF, and no shared context information. 

- TLS 1.3. 
    - [TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/22): Select and add config, setup, and implementation dependency information.
- Cryptographic Hash Function `Hash`. We use SHA3-256 throughout.
- CSPRNG, `rng`.
- A symmetric AEAD scheme that consists of:
    - An encryption function `Enc` that takes a pair `(key, msg, data)`, where `key` is the symmetric key, `msg` is the message to be encrypted, and `data` is OPTIONAL associated data, and outputs a ciphertext.
    - A decryption function `Dec` that takes a pair `(key, ciphertext, data)`, where `key` is the symmetric key,`ciphertext` is the a ciphertext to be decrypted, and `data` is OPTIONAL associated data, and outputs a plaintext.

TODO: Add dependency information for the above.

## Expected Outcomes
A working command-line demo with cryptography for handling arbitrary secrets. 