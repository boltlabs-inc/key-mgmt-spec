

## Development Notes
We are iteratively developing a proof of concept in the single-server setting. In the first phase, we focus on basic support for [arbitrary secrets](#arbitrary-secrets). In the second phase, we extend the system to support [signing keys](#signing-keys).

## Page Contents
1. [Arbitrary Secrets](#arbitrary-secrets)<br>
1. [Signing Keys](#signing-keys) <br>

### Arbitrary Secrets 
We want to build a general-purpose, human-centric system that stores and retrieves secrets on a single server. The resulting proof of concept (PoC) will include basic cryptography to realize an end-to-end encrypted storage of arbitrary secrets. This constitutes a fundamental building block of our larger imagined digital asset management system. 

Basic support for arbitrary secrets includes:
- Local generation of secrets, where the generated secret is by default stored both locally and remotely at the key server;
- Retrieving a key from the key server;
- Importing an externally-generated key into the system for storage at the key server;
- Exporting a key from the system; and
- Obtaining an audit log of the asset owner's system and key use activity.

As part of completion of this phase, we are implementing a working command-line demo with cryptography for handling arbitrary secrets. 

For more details on the above, see [Arbitrary Secrets Overview](arbitrary-secrets-overview.md).

### Signing Keys

We want our system to include support of signing keys. In this phase, we will include support for basic ECDSA signing key generation and use.

As part of this phase, we will extend our existing system functionalities to include:
- Remote generation and storage of secrets, without local storage.
- Generic support for generation and use of signing keys. Uses will include retrieval, import/export, generating signatures, and obtaining audit logs.


