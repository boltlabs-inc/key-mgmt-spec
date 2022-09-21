# Development Notes
We are iteratively developing a proof of concept in the single-server setting. In the first phase, we focus on basic support for [arbitrary secrets](#arbitrary-secrets). In the second phase, we extend the system to support [signing keys](#signing-keys).

## Page Contents
1. [Arbitrary Secrets](#arbitrary-secrets)<br>
1. [Signing Keys](#signing-keys) <br>
1. [Cryptographic Protocol and Implementation Dependencies](#cryptographic-protocol-and-implementation-dependencies)<br>

## Arbitrary Secrets 
We want to build a general-purpose, human-centric system that stores and retrieves secrets on a single server. The resulting proof of concept (PoC) will include basic cryptography to realize an end-to-end encrypted storage of arbitrary secrets. This constitutes a fundamental building block of our larger imagined digital asset management system. 

Basic support for arbitrary secrets includes:
- Local generation of secrets, where the generated secret is by default stored both locally and remotely at the key server;
- Retrieving a key from the key server;
- Importing an externally-generated key into the system for storage at the key server;
- Exporting a key from the system; and
- Obtaining an audit log of the asset owner's system and key use activity.

As part of completion of our first development phase, we are implementing a working command-line demo with cryptography for handling arbitrary secrets for the above workflows. 

Extended support for arbitrary secrets includes:
- Remote-only generation and storage of secrets, where the secret is generated and stored entirely at the key server.

For more details on the above, see [Arbitrary Secrets Overview](arbitrary-secrets-overview.md).

## Signing Keys

We want our system to include support of signing keys. In this phase, we will include support for basic ECDSA signing key generation and use.

As part of this phase, we will extend our existing system functionalities to include:
- Remote generation and storage of secrets, without local storage.
- Generic support for generation and use of signing keys. Uses will include retrieval, import/export, generating signatures, and obtaining audit logs.

## Cryptographic Protocol and Implementation Dependencies
We have the following dependencies:
- We instantiate the asymmetric password-based authenticated key exchange protocol with OPAQUE. We are using [opaque-ke](https://docs.rs/opaque-ke/2.0.0-pre.3/opaque_ke/index.html), which currently implements [version 09 of the IETF RFC](https://datatracker.ietf.org/doc/draft-irtf-cfrg-opaque/09/). This library is under active development, as is the IETF draft. An earlier release of this repository has been audited by NCC Group in June 2021. 
    - This implementation relies on [voprf](https://github.com/novifinancial/voprf), which is tracking [the IETF RFC on OPRFs](https://datatracker.ietf.org/doc/draft-irtf-cfrg-voprf/).
    - As both of the above RFCs are in flux, we expect ongoing updates.
    - Following the terminology of the RFC, we use the following OPAQUE-3DH configuration: OPRF(ristretto255, SHA-512), HKDF-SHA-512, HMAC-SHA-512, SHA-512, ristretto255, with Argon2 as the KSF, and no shared context information. 
- TLS 1.3. 
    - [TODO #22](https://github.com/boltlabs-inc/key-mgmt-spec/issues/22): Select and add config, setup, and implementation dependency information.
- Cryptographic Hash Function `Hash`. We use [SHA3-256](https://nvlpubs.nist.gov/nistpubs/fips/nist.fips.202.pdf) throughout in our constructions, as implemented in [sha3](https://docs.rs/sha3/latest/sha3/) by RustCrypto.
- CSPRNG, `rng`.
    - We use the [`rand` crate's `CryptoRng` trait](https://docs.rs/rand/latest/rand/trait.CryptoRng.html) to require cryptographically secure random number generators in the crypto module.
    - In the client and server code, we instantiate random number generator using the [`StdRng` provided by the `rand` crate](https://docs.rs/rand/latest/rand/rngs/struct.StdRng.html).
    - In most tests, we use the [`ThreadRng` provided by the `rand` crate](https://docs.rs/rand/latest/rand/rngs/struct.ThreadRng.html). Occasionally, we use a manually seeded `StdRng` to get predictable behavior.
- Symmetric AEAD scheme. We are using [chacha20poly1305](https://docs.rs/chacha20poly1305/0.10.1/chacha20poly1305/index.html) by RustCrypto, which implements [RFC 8439](https://tools.ietf.org/html/rfc8439). This library is under active development. An earlier release of this repository was audited by NCC Group in February 2020.
    - This scheme uses a 256-bit pseudorandom key. There are no further requirements on the format or properties of the key.
    - This implementation will not execute in constant time on processors with a variable-time multiplication operation.
 - HMAC-based key derivation function. We use [hkdf](https://docs.rs/rust-crypto/0.2.36/crypto/hkdf/) by RustCrypto, which implements [RFC 5869](https://datatracker.ietf.org/doc/html/rfc5869).
- A message authentication code (MAC).
    - [TODO #149](https://github.com/boltlabs-inc/key-mgmt/issues/149), [TODO #49](https://github.com/boltlabs-inc/key-mgmt-spec/issues/49): Propagate implementation decisions from #149 here.
- Cryptographic signing primitives:
    - ECDSA on secp256
    - Ed25519 (i.e., EdDSA on edwards25519).
    - [TODO #49](https://github.com/boltlabs-inc/key-mgmt-spec/issues/49): Propagate implementation decisions here.

