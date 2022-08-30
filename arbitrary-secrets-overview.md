# Arbitrary Secrets 

Lock Keeper should provide a general-purpose, human-centric system that stores and retrieves secrets on a single server. 

## Page Contents
1. [Assumptions and Requirements](#assumptions-and-requirements) <br>
1. [System Architecture](#system-architecture) <br>
1. [Workflows](#workflows) <br>
    1. [Setting up secure channels with the key server](#setting-up-secure-channels-with-the-key-server)<br>
    1. [Operations on arbitrary secrets](#operations-on-arbitrary-secrets)<br>
1. [Cryptographic Protocol and Implementation Dependencies](#cryptographic-protocol-and-implementation-dependencies) <br>

## Assumptions and Requirements
- The primary stakeholder is a human with secrets. We call this human an _asset owner_.
- All workflows MUST be initiated from a local device owned and operated by a human, namely the asset owner.
- There is only one key server. This key server is either run on external cloud infrastructure or hosted locally by a service provider.

## System Architecture
See the [Systems Architecture page](systems-architecture.md) for details.

There are two basic components: a [client](systems-architecture.md#client) (which runs on an asset owner device and initiates all workflows) and a [key server](systems-architecture.md#key-server) (which provides the storage and retrieval of arbitrary secrets). These components communicate over secure channels at all times. 

## Workflows
We provide sketches of the basic generation, storage, and use of arbitrary secrets below. These flows are initiated by the asset owner. 

### Setting up secure channels with the key server
See the [Networking subsection](systems-architecture.md#networking) on the [Systems Architecture page](systems-architecture.md) for details.
1. The asset owner may _register_ with the key server via an asymmetric password-authenticated key exchange protocol.
    1. Registration MUST occur via a channel that satisfies authentication of the server, confidentiality, and integrity.
1. The asset owner may _open an authenticated session_ with the key server via an asymmetric password-authenticated key exchange protocol. 
    1. Once established, the asset owner can make operation requests on secrets to the key server. 
    1. The session MUST provide a secure channel that satisfies mutual authentication, confidentiality, and integrity.
1. The asset owner may _close_ a session that the asset owner previously established with the key server.

### Operations on arbitrary secrets
See the [Operations on Arbitrary Secrets page](cryptographic_flows.md#) for details.

1. The asset owner can _generate_ secrets. Generation of secrets may be _local_ or _remote_:
    1. The asset owner can generate a secret locally, i.e, using the client.
        1. This secret MUST be randomly generated according to the uniform distribution.
        1. This secret should default to 256-bits.
    1. The asset owner can request the key server to generate a secret remotely.
        1. This secret MUST be randomly generated according to the uniform distribution.
        1. This secret should default to 256-bits.
1. The asset owner can _store_ a secret. 
    1. If the secret is generated locally, then the secret is by default stored both locally and at the key server.
    1. If the secret is generated remotely, then the secret is by default stored only at the key server.
    1. Storage on a key server MUST occur via a mutually authenticated channel that satisfies confidentiality and integrity. 
    1. Local storage on the user's device should be secure.
    1. An additional functionality that allows the user to store secrets on the remote key server only WILL be added in the future.
    1. An additional functionality that allows the user to store secrets on the local device only may be added in the future. 
1. The asset owner can _retrieve_ a secret from the key server. 
    1. Retrieval from a key server MUST occur via a mutually authenticated channel that satisfies confidentiality and integrity. 
    1. The key retrieved MUST be a key associated to the authenticated user.
    1. A retrieved secret can be:
        1. Stored locally in secure storage on the user's device.
        1. Copied to the system clipboard.
1. The asset owner may _import_ a secret to the system in an appropriate format. 
    1. The default expected format is as bytes of the form ``len || secret``, where `len` is 1 byte that represents the length of the secret `secret` in bytes.
    1. The implementation may define other acceptable import formats.
    1. Any communication with the key server that occurs in serving an import request MUST occur via a mutually authenticated channel that satisfies confidentiality and integrity.
1. The asset owner may _export_ the secret from the system. The export format for the key SHOULD allow for easy transfer of the key material to another digital asset management system, i.e., secrets should be portable.
    1. The default expected format is as bytes of the form ``len || secret``, where `len` is 1 byte that represents the length of the secret `secret` in bytes.
    1. The implementation may define other acceptable export formats.
    1. Any communication with the key server that occurs in serving an export request MUST occur via a mutually authenticated channel that satisfies confidentiality and integrity. 
1. The asset owner may _audit_ the operations performed by the key server on a given secret. This allows the asset owner to retrieve a log of operations from the key server.
    1. Audit log retrieval MUST occur via a mutually authenticated channel that satisfies confidentiality and integrity.
    1. Audit logs should be portable, i.e., easily exportable from the system.
    1. Non-normative note: Given the above properties, we do not achieve integrity of the audit logs in the presence of a cheating key server. Future work may address this concern.


