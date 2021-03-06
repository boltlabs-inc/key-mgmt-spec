# System Architecture

## Components

### Client 
This software MUST run on the asset owner's device in order for the desired security property to be achieved. 
The client functionality is split between a library and a demo repository that shows how to integrate the library into a human-facing application; see [here](repository-list.md) for more details. This component interfaces with the key server component and is responsible for realizing [the asset owner workflows](current-development-phase.md#workflows) in the current development phase.

In practice, the Service Provider may choose to directly integrate the library component into a mobile application or as hardened WASM into a browser-based service. 



### Key server
The library component is integrated into a host server and interfaces with the client component.

This component is responsible for encrypted storage of secrets on behalf of the asset owner. That is, the key server performs operations on secrets as requested by the asset owner. See [the asset owner workflows](current-development-phase.md#workflows) for more information on the allowed operations. 

In practice, the Service Provider may choose to either run the host server themselves or use one or more external cloud providers. The best security properties will be achieved by distributing trust across multiple cloud providers.

## Networking
### Session requirements
We wish to ensure the following in our implementation:
1. Logical sessions between the client and key server should be independent of the underlying transport layer. This allows:
    1. Resumption between different actual underlying connections.
    1. Multiplexing many sessions over a single connection. (NB: This may not be relevant for realizing the currrent PoC, but may be advantageous later.)
1. There are two basic types of application-layer sessions between the client and key server, each of which has different requirements:
    1. A _registration session_: This type of session is opened when an asset owner _registers_ with the key server for the first time. That is, this type of session is required when an asset owner is a new user of the system or has never interacted with the key server before.
        1. Registration sessions have no requirement for authentication of the asset owner, although future versions of this project may provide this as an optional feature for the Service Provider.
        1. Registration sessions MUST authenticate the key server to the local client.
        1. Registration sessions MUST provide confidentiality and integrity.
    1. A _request session_: This type of session is opened when an asset owner who has previously registered with the system sends a request for the key sever to perform an operation on a secret (i.e., store, retrieve, audit, import, export).
        1. Request sessions MUST provide mutual entity authentication.
        1. Request sessions MUST provide confidentiality and integrity.

### Session details
#### Underlying transport layer
We assume a Public Key Infrastructure (PKI). For all session types, the local client first authenticates the key server and opens a channel using TLS 1.3.
  - [TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/22): A PKI and TLS configuration must be selected and the details added to this specification in this section and [here](current-development-phase.md#cryptographic-protocol-and-implementation-dependencies) as appropriate.

#### Registration session

1. The client authenticates the key server and opens a channel with the key server as specified [above](#underlying_transport_layer).
1. The client and key server run the OPAQUE registration stage; see [this section](current-development-phase.md#cryptographic-protocol-and-implementation-dependencies) for protocol version and dependency information.
    1. The client receives as output a value `export_key`, which is a pseudorandom value, independent of all other OPAQUE protocol values, that is known only to the client.
    1. The key server receives as output a record that corresponds to the client's registration.

#### Request session

1. The client authenticates the key server and opens a channel with the key server as specified [above](#underlying_transport_layer).
1. The client and key server mutually authenticate via the OPAQUE protocol; see [this section](current-development-phase.md#cryptographic-protocol-and-implementation-dependencies) for protocol version and dependency information.
    1. The client receives as output two values, an `export_key` (matching that from the registration phase) and a `session_key` (which is the output of an authenticated key exchange).
    1. The server receives as output a `session_key` matching that of the client.
1. The client and server open an encrypted channel secured under a key derived from `session_key`. [TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/29): Include additional details here.
1. The client sends their request (i.e., one of store, retrieve, audit, import, export) to the key server.
