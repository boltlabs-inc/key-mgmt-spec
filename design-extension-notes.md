# Design Extension Notes

## Overview
There are two technologies we wish to incorporate into the Lock-Keeper design: _secure processors_ (or _enclaves_) and _multi-party-computation (MPC)_. This in-progress document will capture the requirements and draft designs for each.

The expected order of integration:
1. Secure processors.
2. MPC.

The notes for MPC therefore build on the notes for integrating secure processors.

## Common Requirements
This section details requirements that are common to both technologies.

### Sessions
### Session management
It may be helpful to use the outer (TLS, perhaps Noise in the future) authentication layer to authenticate the client.

#### Communication channel requirements
We need a secure channel (satisfying mutual authentication, confidentiality, and integrity) between the _key server framework_ (i.e., the component that implements the server side [system functionalities](system-functionalities.md) and the client. This may entail using a session key derived from the OPAQUE handshake to secure all messages sent during an open session. Implicit in this statement is that the server side of the OPAQUE handshake must be handled directly by the key server framework.

#### Password-guessing attacks 
Our system MUST be robust against password-guessing attacks. Specifically, the system should satisfy the following properties:
- A suitably robust password choice renders the probability of a successful attack negligible.
- Servers limit the number of authentication attempts a user makes and actively respond to possible password-guessing attacks to prevent key theft and misuse.

### Secure Storage
The key server must always know that they are looking at the fresh state of the database, i.e., we need secure deletion. In particular, we treat the database as untrusted.

Replication and backup of secrets by the key servers should require active participation by the asset owner.

There is metadata related the keys that the key server MUST track securely. The simplest way to do this is to encrypt this data under a key known only to the enclave and then store in the untrusted portion of the server (or another server). This list is a WIP:
- The functionalities the key server is willing to compute is different based on both type of account and type of key. This implies the key server must securely store:
    - The type of account, i.e., the type of client used to register a user. The key server should be able to track, on a per-key basis, what type of account is associated.
    - The key identifier, key type, account mapping. 

### Availability
See the discussion on availability goals with respect to [network-level adversaries](design-philosophy-and-threat-model.md#passive-and-active-network-level-adversaries).
- TODO: Record strategies we will integrate here.

### Recovery

In the near-term, we intend to incorporate an export functionality that allows for the export of signing keys from the system. This functionality may be used by the calling application to build various backup mechanisms, e.g., use of secure local storage on an iPhone or dedicated storage device. We will also explore support for social recovery mechanisms in the future.

## Secure Processors
We intend to incorporate secure processors on the key server(s). Our end goal is that the [system functionalities](system-functionalities.md) and OPAQUE handshake are run in an enclave. Other non-core functionalities can be handled outside of the enclave. Additionally, we desire compatibility with multiple cloud providers and therefore multiple enclave technologies. 

### Goals
Designing and integrating enclaves is a large design effort, with significant challenges. In particular, due to the restrictions/limited functionalities of enclave environments, together with the limited available documentation, we will take an iterative approach to design and integration. We will start with an exploration of the enclave boundaries, with the goal of using secure processors for a minimal set of functionalities, e.g., for all server-side computations on sensitive user data, but not including (all of) session handling, message framework, general user management, etc. inside the enclave itself.

Additionally, we will design an enclave abstraction and define a path for future integration of multiple enclave technologies, starting with SGX and AWS Nitro.

### Design notes
- Ideally, the server-side computations for the OPAQUE handshake should be handled by the enclave. This is because the `session_key` used to secure communication channels is derived from OPAQUE, and our goal is to not trust the service provider with the content of messages sent over this channel. That is, there should be a secure channel that terminates inside the enclave, over which all system fucntionality protocol messages are sent.
- Secure deletion: In the single server setting, ensuring that the server is always working with a fresh view of the database likely requires storing information in the hardware-encrypted RAM of the enclave.
- The authentication oracle (part of the registration record for OPAQUE held by the server) should be held in encrypted storage, under a key known only to the enclave. This mitigates password-guessing attacks upon server compromise and further limits the amount of trust users place in Forte with respect to the security of their passwords.

#### Keys
We want the key servers to hold a per-user seed `user_seed`, for use in creating user keys.

##### Self-custodial signing keys
Then the user key derivation process works as follows:

Common Input:
- a domain separator,
- an application identifier `application_id`, where the application identifier designates a particular payment network and cryptographic primitive,
- a key type, `"self-custodial"`.

Client input:
- custodial_key 
    - derived from export key (i.e., output from OPAQUE)? 
    - Add a layer of indirection?, e.g., derived from `master_key`? or stored under master key together with `storage_key` (as in the arbitrary secrets PoC for e2e-encrypted storage?) and then retrieved at time of use? What choice makes updating things easier?

Key server input:
- `user_seed`, a user-specific seed.

Protocol:
The key can then be computed as an OPRF computation between the user and key server(s), where the client does not receive an output, and the key server receives the key (share). The underlying protocol can look very much like [remote generation and storage functionality](system-functionalities.md#remote-only-secret-generation-and-storage), modified to include the joint OPRF computation.

Complications:
- Ideally a verifiable, partially oblivious PRF would be used here. But this will not be easy or efficient to thresholdize for the MPC variant, i.e., open research question here.
- We can try using a verifiable PRF, or just a regular PRF instead, relying on the use of enclaves for security. In this case, perhaps fold the common inputs into the key server inputs instead.
- What happens when we want to change the password. It may be simpler to use updateable encryption for storage of generated keys.

##### Delegated signing keys
Not supported at this time.

Notes:
- Basic [remote generation and storage functionality](system-functionalities.md#remote-only-secret-generation-and-storage) can be used as a building block. In this use case, the user does not have to be an active participant in generating the keys.
- We will need to build a delegated client.
- Design an authorization protocol to set up delegated keys, where the user must be an active participant.


## Multi-party Computation (MPC)
We intend to incorporate multiple key servers, each of which contributes to key generation and use. In this way, we can distribute trust across multiple cloud providers, with each provider having limited access to system and user data.

### Basics
We have `n` servers `S_1, ..., S_n` and a threshold parameter `t`.

### Sessions
- When the client registers in our system, the client registers with each key server.
- When the client opens a session in our system, the client authenticates to a subset of key servers (`t` out of `n`)
- The authentication process should result in `t` shared secrets, `k_1, k_2, ..., k_t`, where the secret `k_i` (for `1 <= i <= t`) is known only to the client and the server `S_i`. These secrets can then be used to establish an authenticated channel between the client and the subset of servers.
- The client should receive as output a secret known only to the client, as in basic OPAQUE.

Based on these requirements, a T-PAKE is an appropriate design choice for user authentication in the multiple server setting. In particular, a T-PAKE provides increased robustness against server compromise against offline dictionary attacks (i.e., an attacker must compromise `t+1` servers in order to _launch_ an offline dictionary attack on the user's password.)

We can extend OPAQUE to handle this threshold setting, and our current dependency for OPAQUE may be a good start point for development efforts on this. A possible interim solution is to have the clients run OPAQUE separately for each server, although this does not provide us with the security advantages of a T-PAKE.

Architecturally, it should be possible to use an aggregator to help with routing and processing of messages from the key servers on behalf of the client.

#### Password-guessing
The key servers can use a consensus protocol to track the number of login attempts by a user.
- Question: what trust model is appropriate here? See note below.

### Secure storage
The key servers can use a consensus protocol to determine freshness of database state.
- Question: what trust model is appropriate here? See note below.

### Design notes
- We could use vanilla user authentication with distributed OPAQUE and treat the generated keys as a separate DKG problem (where generated keys still need to be bound to the user authentication process). Options here include updateable encryption, as observed in the secure processor notes above (i.e., we need to be able to update user passwords still, as well as do key share updates).
- We could consider replacing the OPRF in OPAQUE to do "double duty", i.e., providing mutual authentication plus (master) key share derivation for the server and user (i.e., each party receives their own secret master key). Subkeys could then be produced as an additional OPRF computation.
    - We should consider how well our threshold ECDSA primitive options integrate with such a solution.
    - If we do vanilla threshold ECDSA (including a DKG) in combination with enclaves, this may be sufficient security for delegated keys, and, in a non-decentralized setting, self-custodial keys. In a decentralized setting, it is likely advisable to have the users be full participants in the threshold computation and require the user share for self-custodial key use (and a delegated authority share for delegated key use). Users could keep e2e-encrypted backups of their share and secure this under their password.
- _Trust model_: For the trust model implications of this design, to what extent do we wish to handle Byzantine server behavior (as opposed to relying only on enclaves to have "trusted" portions of the server)? Options:
    - If one entity runs (a large enough subset of) the servers, the servers can wait for the user to log in and use this opportunity to exfiltrate the key (without enclaves), or have a better chance of exfiltrating the key (with enclaves).
    - In a legal sense, enclaves provide protection for the entities running the key servers (both our direct customer plus external partners that run the infrastructure).
    - We can use verifiable primitives that satisfy the property that a misbehaving server can be identified. The protocol can then abort. This is a tradeoff between efficiency and robustness made in several designs, e.g., FROST., and may be reasonable for our current use case. It is also less complex.
    - We could go full Byzantine with a goal of robustness, but this is more complex, and probably only needed for future, decentralized versions of the system.
    - Providing users the option of holding a key share and considering more general access control structures could mitigate concerns about misbehaving servers in the decentralized setting.
- Threshold ECDSA options: We likely want a proactive scheme with identifiable aborts that is compatible with cold storage/HSM use. This immediately points us in the direction of [Canetti, Gennaro, Goldfeder, Makriyanni, and Peled's solution [CGGMP20]](https://eprint.iacr.org/2021/060).








