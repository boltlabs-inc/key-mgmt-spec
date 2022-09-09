# Design Extension Notes

## Overview
There are two technologies we wish to incorporate into the Lock-Keeper design: _secure processors_ (or _enclaves_) and _multi-party-computation (MPC)_. This in-progress document will capture the requirements and design notes for each.

- Secure processors: Designing and integrating enclaves is a large design effort, with significant challenges. In particular, enclave environments have significant resource restrictions and limited functionalitie. Some notes on our approach follow:
    - We intend to incorporate secure processors on the key server(s). Our end goal is that the [system's cryptographic functionalities](system-functionalities.md) and OPAQUE handshake are run in an enclave. Other non-core functionalities can be handled outside of the enclave.
    - We will design an enclave abstraction and define a path for future integration of multiple enclave technologies, starting with SGX and AWS Nitro. 
    - We will start with an exploration of appropriate enclave boundaries. The goal is to use secure processors for a minimal set of functionalities, e.g., for all server-side computations on sensitive user data, but excluding, to the extent possible, session handling, message framework, general user management.
- MPC: We intend to incorporate multiple key servers, each of which contributes to key generation and use. 
    - In this way, we can distribute trust across multiple cloud providers, with each provider having limited access to system and user data. In this setting, we have `n` servers `S_1, ..., S_n` and a threshold parameter `t`.
    - Architecturally, it may also be possible to use an aggregator to help with routing and processing of messages from the key servers on behalf of the client, as well as in the final processing of a signature request (i.e., the formation of a signature from partial signatures, where the partial signatures are provided by a subset of key servers).


## Requirements and High-level Design Notes
### Threat Model 
See the [Design Philosophy and Threat Model section](design-philosophy-and-threat-model) for information on our existing approach and requirements with respect to threat model. Of particular note for our system extensions:
1. Password-guessing attacks: Our system MUST be robust against password-guessing attacks. Specifically, the system should satisfy the following properties:
    1. A suitably robust password choice renders the probability of a successful attack negligible.
    1. Servers limit the number of authentication attempts a user makes and actively respond to possible password-guessing attacks to prevent key theft and misuse.
1. Key server storage: We treat databases as untrusted to the extent possible. This is in line with our principle of limiting trust placed in individual components (and therefore the attack surface) as much as possible. Additionally, this choice ideally allows for flexible deployment scenarios, such as the use of managed persistence solutions.

Enclave integration notes:
- Password guessing attacks: We can consider using hardware-encrypted RAM to track login attempts.

MPC integration notes:
- Password guessing attacks: The key servers can use a consensus protocol to track the number of login attempts by a user.
- Trust model questions: For the trust model implications of this design, to what extent do we wish to handle Byzantine server behavior (as opposed to relying only on enclaves to have "trusted" portions of the server)? Options:
    - If one entity runs (a large enough subset of) the servers, the servers can wait for the user to log in and use this opportunity to exfiltrate the key (without enclaves), or have a better chance of exfiltrating the key (with enclaves).
    - In a legal sense, enclaves provide protection for the entities running the key servers (both our direct customer plus external partners that run the infrastructure).
    - We can use verifiable primitives that satisfy the property that a misbehaving server can be identified. The protocol can then abort. This is a tradeoff between efficiency and robustness made in several designs, e.g., FROST., and may be reasonable for our current use case. It is also less complex.
    - We could go full Byzantine with a goal of robustness, but this is more complex, and probably only needed for future, decentralized versions of the system.
    - Providing users the option of holding a key share and considering more general access control structures could mitigate concerns about misbehaving servers in the decentralized setting.

### Transport Layer
TLS with server authentication is used to secure the transport layer.

### Application-Layer Sessions

In our context, an application-layer session consists of a mutually authenticated, secure interaction between a client and key server.

To open a session, the client and key server mutually authenticate using OPAQUE. Successful authentication via OPAQUE and the creation of a secure channel between client and server results in an _open session_. 

Additional notes on session management:
1. Session identifiers: Each session is associated with a unique _session identifier (sid)_.
1. Subsessions: Each [system functionality](system-functionalities.md) that involves interaction between the client and server(s) requires the establishment of a _subsession_. Each subsession is associated with a unique _subsession identifier_ (ssid). Subsessions must be bound to a session.
1. Communication channels: We need a secure channel (satisfying mutual authentication, confidentiality, and integrity) between the _key server framework_ (i.e., the component that implements the server side [system functionalities](system-functionalities.md) and the client. This will entail using a session key derived from the OPAQUE handshake to encrypt all cryptographic protocol messages sent during an open (sub)session. Implicit in this statement is that the server side of the OPAQUE handshake must be handled directly by the key server framework.
1. The server holds a per-user `user_seed` and uses this value to derive a per-user `oprf_seed` for use in OPAQUE. The derivation MUST include an appropriate domain separator.

Enclave integration notes:
1. The key server framework should be implemented in the enclave. For example, the server-side computations for the OPAQUE handshake should be handled by the enclave. This is because the `session_key` used to secure communication channels is derived from OPAQUE, and our goal is to not trust the service provider with the content of messages sent over this channel. That is, there should be a secure channel that terminates inside the enclave, over which all system functionality protocol messages are sent.
1. The `user_seed` values are highly sensitive and must be stored securely.

MPC integration notes:
1. OPAQUE can be thresholdized to support a t-of-n server setup natively. Alternatively, as an interim solution, OPAQUE may be run pairwise. In particular:
    1. Each key server (cluster) would have a unique per-user `user_seed`.
    1. When the client registers in our system, the client registers with each key server.
    1. When the client opens a session in our system, the client authenticates to a subset of key servers (`t` out of `n`)
    1. The authentication process should result in `t` shared secrets, `k_1, k_2, ..., k_t`, where the secret `k_i` (for `1 <= i <= t`) is known only to the client and the server `S_i`. These secrets can then be used to establish an authenticated channel between the client and the subset of servers.
    1. The client should also receive as output a secret known only to the client, as in basic OPAQUE.
1. Based on these requirements, a T-PAKE is an appropriate design choice for user authentication in the multiple server setting. In particular, a T-PAKE provides increased robustness against server compromise against offline dictionary attacks (i.e., an attacker must compromise `t+1` servers in order to _launch_ an offline dictionary attack on the user's password.)
1. Our current dependency for OPAQUE may be a good start point for development efforts to thresholdize the OPAQUE. As stated above, a possible interim solution is to have the clients run OPAQUE separately for each server, although this does not provide us with the security advantages of a T-PAKE.

### Secure Storage
Requirements for secure storage include:
- The key server must always know that they are looking at a fresh database state.
- We need to achieve secure deletion: an asset owner who has exported (and requested the deletion of key material from the key server) should be confident that a key server cannot later access this deleted key material. 
- Replication and backup of secrets by the key servers should require active participation by the asset owner.

There is metadata related to the keys that the key server MUST track securely: 
- The functionalities the key server is willing to compute is different based on both type of account and type of key. This implies the key server must securely store:
    - The type of account, i.e., the type of client used to register a user. The key server should be able to track, on a per-key basis, what type of account is associated.
    - The key identifier, key type, and account mapping. 
- There may be additional pieces of metadata that have security implications if this data is not handled properly; this list is a work in a progress.

Enclave integration notes:
- One option for secure storage is to encrypt data under a key known only to the enclave. The ciphertexts may then be stored in the untrusted portion of the server (or another server).
- If there is data that can be revealed to the untrusted storage, we can use less heavyweight techniques to ensure integrity of such data.
- We should consider whether data encrypted clientside may be stored in untrusted storage _without_ (or with minimal) involvement from the enclave. See also notes on communication channel requirements.
- Ensuring that the key server is always working with a fresh view of the database may require storing information in the hardware-encrypted RAM of the enclave. This is particularly true in a single (or cluster-based) server setting.
- We should consider storing the authentication oracle (part of the registration record for OPAQUE held by the server) in secure storage, i.e., encrypted under a key known only to the enclave. This mitigates password-guessing attacks upon server compromise and further limits the amount of trust users place in the key servers with respect to the security of their passwords.

MPC integration notes:
- The key servers can use a consensus protocol to determine freshness of database state. 

### Availability
See the discussion on availability goals with respect to [network-level adversaries](design-philosophy-and-threat-model.md#passive-and-active-network-level-adversaries).

Enclave integration notes:
- We can take a standard approach with clustering to achieving high availability. 

MPC integration notes:
- MPC itself can be useful for achieving high availability: by picking a threshold value `t`, where `t < n`, it suffices for `t` servers to be available.

### Recovery

In the near-term, we intend to incorporate an export functionality that allows for the export of signing keys from the system. This functionality may be used by the calling application to build various backup mechanisms, e.g., use of secure local storage on an iPhone or dedicated storage device. We will also explore support for social recovery mechanisms in the future.

## (Slightly) Lower-Level Design Notes: Signing Keys and Arbitrary Secrets
As stated in [Application Layer Sessions](#application-layer-sessions), the key server holds a per-user seed `user_seed`. The server also uses this seed as part of the protocol to create and store keys.

##### Self-custodial signing keys
Then the user key derivation process works as follows:

Common Input:
- a domain separator,
- an application identifier `application_id`, where the application identifier designates a particular payment network and cryptographic primitive,
- a key type, `"self-custodial"`.

Client input:
- TBD

Key server input:
- `user_seed`, a user-specific seed, TBD.

Protocol Notes:
Both the signing key and the encryption key used for storage of the generated signing key can be computed as a computation between the user and key server(s), where the key server computes the final the key (share) and corresponding encryption key. The underlying protocol can look very much like [remote generation and storage functionality](system-functionalities.md#remote-only-secret-generation-and-storage), modified appropriately.

Complications:
- If we have just one storage key per client, then when the client makes a request to generate a signature/access a secret, the key server can access _any_ of the client keys (or key shares). Alternatively, we can have a per-secret storage key with appropriate use of domain separation by the client and server(s).
- Rotating the encryption keys used to store signing keys is particularly expensive if we use a per-secret storage key.
- Rotating passwords requires rotating storage keys as well, unless we are careful to build in a layer of indirection to allow for the flexibility of a password change without changing the underlying storage key. This  requires more analysis and thought:
    - What are the circumstances for changing a password? What is the most common use case? What is our threat model?
    - Given the unique use case of securing digital assets _that hold money_: what is the appropriate behavior after a potential compromise of a key? 

Notes:
- Without MPC, this approach is very dependent on enclaves for security of signing keys.
- With MPC, this is very dependent on independent running of key servers and/or enclaves unless we require the client to hold a share and use an access structure that requires the client's share. There are several considerations here: while strictly speaking this would be a nice way to offer asset owner custody, performance and usability come to mind. If we really expect the average user to choose a poor password, then what is added by this approach?
- We could consider replacing the OPRF in OPAQUE to do "double duty", i.e., providing mutual authentication plus (master) key share derivation for the server and user (i.e., each party receives their own secret master key). Subkeys could then be produced as an additional computation.
- If we do vanilla threshold ECDSA (including a DKG) in combination with enclaves, this may be sufficient security for delegated keys, and, in a non-decentralized setting, self-custodial keys. In a decentralized setting, it is likely advisable to have the users be full participants in the threshold computation and require the user share for self-custodial key use (and a delegated authority share for delegated key use). Users could keep e2e-encrypted backups of their share and secure this under their password; we could consider _where_ these backups should be stored. e.g., We may wish to store them at a special-purpose (set of) server(s), rather than at the same key servers that hold key shares.
- Threshold ECDSA protocol choice: As we likely want a proactive scheme with identifiable aborts that is compatible with cold storage/HSM use. This immediately points us in the direction of [Canetti, Gennaro, Goldfeder, Makriyanni, and Peled's solution [CGGMP20]](https://eprint.iacr.org/2021/060).

##### Delegated signing keys
Will be supported at a later date.

Notes:
- Basic [remote generation and storage functionality](system-functionalities.md#remote-only-secret-generation-and-storage) can be used as a building block. In this use case, the user does not have to be an active participant in generating the keys.
- It is possible to use a policy-based, enclave-dependent approach to achieve this functionality, but MPC is preferable. e.g., The server might always have access to a given delegated key (share), and it authenticates the signing request under the delegatee's public key (or as normal, if the request comes from the asset owner).
- An MPC-based solution depends on supporting an access structure of the form `(C OR D) AND (t-of-n)`, where `C` represents a client, `D` represents a `delegatee`, and the `t-of-n` clause represents the requirement that a threshold `t` out of `n` servers must participate in the creation of a signature.
- We will either need to build a delegated client or incorporate delegated functionality into LockKeeperClient.
- We need to design an authorization protocol to set up delegated keys, where the user must be an active participant.








