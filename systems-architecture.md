# System Architecture

## Components

### Client 
This software MUST run on the asset owner's device in order for the desired security property to be achieved. 
The client functionality is split between a library and a demo repository that shows how to integrate the library into a human-facing application; see [here](repository-list.md) for more details. This component interfaces with the key server component and is responsible for realizing [the asset owner workflows](current-development-phase.md#workflows) in the current development phase.

In practice, the Service Provider may choose to directly integrate the library component into a mobile application or as hardened WASM into a browser-based service. 

### Key server
The binary is integrated into a host server and interfaces with the client component.

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
1. For security, all verification checks run by the key server during session setup and resumption MUST run in constant-time.


### Underlying transport layer
We assume a Public Key Infrastructure (PKI). For all session types, the local client first authenticates the key server and opens a channel using TLS 1.3.
  - [TODO #22](https://github.com/boltlabs-inc/key-mgmt-spec/issues/22): A PKI and TLS configuration must be selected and the details added to this specification in this section and [here](current-development-phase.md#cryptographic-protocol-and-implementation-dependencies) as appropriate.

Non-normative note: In the current design, we rely on TLS for confidentiality and, during the registration stage of OPAQUE, for authentication of the server.

### Application-layer authenticated channel
In Lock Keeper, we create an additional, [application-layer channel](#application-layer-authenticated-channel) that provides mutual authentication, using an underlying [mutual authentication protocol](#underlying-mutual-authentication-protocol) as a building block.

#### Underlying mutual authentication protocol
We use the OPAQUE protocol for mutual authentication of asset owners and the key server; see [this section](current-development-phase.md#cryptographic-protocol-and-implementation-dependencies) for protocol version and dependency information. 

##### Registration stage 

The registration stage of OPAQUE satisfies the following:
1. The client inputs `user_credentials`, which consists of:
    - `account_name`, bytes that represent the asset owner's human-memorable account information, e.g., email address; and
    - `password`, bytes that represent the asset owner's human-memorable secret authentication information.
1. The key server MUST check that the given user identifier `account_name` is _fresh_ (i.e., no user has previously registered with the given identifier) and of the expected format and length. If this check fails, the key server MUST fail the registration request.
1. The client receives as output a value `export_key`, which is a pseudorandom value, distributed independently of all other OPAQUE protocol values, that is known only to the client.
1. The key server receives as output a record that corresponds to the client's registration, which includes an identifier `account_name` that matches that of the client.

##### Authentication stage
The authentication stage of OPAQUE is a password-based authenticated key exchange protocol that satisfies the following:
1. The client receives as output two values, an `export_key` (matching that from the registration phase) and a `session_key` (which is the output of an authenticated key exchange).
1. The server receives as output a `session_key` matching that of the client.
1. Both client and server receive confirmation of the success of the AKE.

Non-normative note: The client and server MUST receive confirmation that the AKE has completed successfully _prior_ to using `session_key`, else security is compromised.

#### Opening the application-layer authenticated channel
 The client and server open an authenticated channel secured under a key derived from `session_key`. 
- [TODO #149](https://github.com/boltlabs-inc/key-mgmt/issues/149): Include additional details here once the implementation from #149 is complete.
- Implementation Note: This key is deterministically derived from `session_key` and should be used as a key for a message authentication code scheme `MAC`. In the following, when the client and key server send each other messages, it is assumed that these messages are authenticated under this MAC/key pair. The key server and the client should additionally reject all messages sent over this channel that fail verification checks on the received ciphertexts. The implementor should be careful to use constant-time verification of the authentication tags.

### Opening and using a registration session
The client initiates a registration session as a subprotocol of the Lock Keeper [register](cryptographic_flows.md#register) functionality.

#### Opening a registration session
Input: 
- `user_credentials`, credentials for use in the OPAQUE protocol:
    - `account_name`, bytes that represent the asset owner's human-memorable account information, e.g., email address; and
    - `password`, bytes that represent the asset owner's human-memorable secret authentication information.
- `rng`, a seeded CSPRNG held by the key server.

Output:
- An open, secure channel available for use in the remaining steps of [register](cryptographic_flows.md#register).
- `user_id`, a globally unique identifier for the asset owner.

Protocol:
1. The client authenticates the key server and opens a channel with the key server as specified in the [underlying transport section](#systems-architecture.md/#underlying_transport_layer).
1. The client and key server run the [OPAQUE registration stage](#registration-stage).
1. The key server generates `user_id`, a 32-byte globally unique identifier (GUID).
    1. Generate `user_id` as 32 bytes of of random output from `rng`.
    1. Checks the server database for a user id with the same value as `user_id`. If one exists, go back to the previous step.
1. The key server stores the identifier `user_id` together with the registration record output from the OPAQUE registration stage.
1. The client and key server mutually authenticate via the [OPAQUE authentication stage](#authentication-stage).
1. The client and server [open an authenticated channel](#opening-the-application-layer-authenticated-channel) secured under a key derived from `session_key`. 
1. The key server sends `user_id` to the client over the authenticated channel.
1. At this point, the registration session is considered _open_. 

Implementation guidance: For security, all verification checks run by the key server MUST run in constant-time.

#### Using a registration session
We have the following requirements for using an open registration session:
    
1. All messages sent between the client and server MUST be over this authenticated channel, i.e., all messages should include an authentication tag that is computed under the selected MAC scheme using the shared key. 
1. The key server and the client MUST additionally reject all messages sent over this channel that fail the MAC verification checks.
1. The only valid request for a registration session is a request to [complete registration](cryptographic_flows.md#complete-registration).
1. The key server MUST close the session after completion of the complete registration request.
    - [TODO #51](https://github.com/boltlabs-inc/key-mgmt-spec/issues/51): Set recommendations for request limits and timeouts that allow more than one request per session.

Implementation guidance: For security, all verification checks run by the key server MUST run in constant-time.

### Opening and using a request session
The client initiates a request session as a prerequisite for processing any _requests_, i.e., operations that involve communication with the key server.

#### Opening a request session
Input: 
- `user_credentials`, credentials for use in the OPAQUE protocol.

Output:
- An open, secure channel available for use in the processing of a request.
- `user_id`, a 128-bit globally unique identifier (GUID) representing the identity of the asset owner.

Protocol:
1. The client authenticates the key server and opens a channel with the key server as specified above in the [underlying transport section](#underlying_transport_layer).
1. The client and key server mutually authenticate via the [OPAQUE authentication stage](#authentication-stage).
1. The server stores the authentication attempt, including the outcome, in an [audit log](cryptographic_flows.md#audit-logs) associated with the given user. 
1. The client and server [open an authenticated channel](#opening-the-application-layer-authenticated-channel) secured under a key derived from `session_key`. 
1. The key server retrieves the identifier `user_id` associated with the authenticated client and sends `user_id` to the client over the authenticated channel.
1. At this point, the request session is consider _open_. 

Implementation guidance: For security, all verification checks run by the key server MUST run in constant-time.


#### Using a request session
We have the following requirements for using an open request session:
1. All messages sent between the client and server MUST be over this authenticated channel, i.e., all messages should include an authentication tag that is computed under the selected MAC scheme using the shared key. 
1. The key server and the client MUST additionally reject all messages sent over this channel that fail the MAC verification checks.
1. The client may then send a single request (i.e., one of store, retrieve, audit, import, export) to the key server during this session.
1. The key server MUST close the session upon completion of the given request.
    - [TODO #51](https://github.com/boltlabs-inc/key-mgmt-spec/issues/51): Set recommendations for request limits and timeouts that allow more than one request per session.

Implementation guidance: For security, all verification checks run by the key server MUST run in constant-time.
