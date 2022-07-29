
# Operations on Arbitrary Secrets

## Setup
The asset owner MUST have successfully completed [Registration](https://github.com/boltlabs-inc/key-mgmt-spec/issues/11) to generate and store arbitrary secrets at the server.
- [TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/11#issuecomment-1194099508): Registration details needs to be fleshed out.
- [TODO]() Authentication details need to be fleshed out.

We will assume the following:
   - `storage_key`, A symmetric key for [`Enc`](#external-dependencies) will be generated client-side during registration. This key will be stored in a ciphertext at the key server, secured under `export_key`.
   - `retrieve_storage_key`, A function will be implemented that allows the client to retrieve and locally decrypt the ciphertext to recover `storage_key`.
   - `session_key`, A key that is derived during authentication and is shared between the client and the key server. This key can be used as a key for the AEAD scheme `Enc`. In the following, when the client and key server send each other messages, it is assumed that these messages are encrypted using `Enc` under `session_key`. The key server should additionally reject all messages sent over this channel that fail verification checks.

## Generate and Store
This client-side functionality generates a secret locally and stores the result both locally and remotely:

Input:
- `user_id`, a 128-bit globally unique identifier (GUID) representing the identity of the asset owner.

Output:
- `key_id`, a 128-bit unique identifier representing a key, computed as the (possibly truncated) output of `Hash` over user and scheme parameters and a randomly generated salt.

Protocol:
1. The client:
   1. Checks if there is an existing open session with the key server and the input `user_id` and [opens a Request Session](systems-architecture.md#request-session) if not. 
   1. Calls `retrieve_storage_key`, the output of which is `storage_key`.
   1. Sends a request message to the key server over the session's secure channel. This message MUST indicate the desire to store a secret remotely and contain `user_id`.
1. The key server:
   1. Runs a validity check on the received `user_id` (i.e., `user_id` must be of the expected format and length, and should match the current authenticated session).
   1. Generates `key_id`, a globally unique identifier as follows:
        1. Generates `randomness` as 32 bytes of randomness output from `rng`.
        1. Computes `key_id` as `Hash(domain_sep, 16, user_id, 32, randomness)`, truncated to 128-bits if necessary, where `domain_sep` is a static string acting as a domain separator, prepended with its length in bytes.
        1. This functionality should fail (with negligible probability) if the generated identifier is not unique among the key server's stored identifiers. 
        1. Sends `key_id` to the client over the secure channel.
1. The client:
    1. Runs the [generate](#generate-a-secret) protocol on input `(32, rng, user_id||key_id)` to get a secret `arbitrary_key`.
    1. Computes `Enc(storage_key, arbitrary_key, user_id||key_id)` and sends the resulting ciphertext to the key server over the secure channel.
    1. [Stores](#store-a-secret-locally) the ciphertext and associated data locally.
1. The key server:
    1. Runs a validity check on the received ciphertext (i.e., the ciphertext must be of the expected format and length).
    1. [Stores](#server-side-storage) a tuple containing the received ciphertext and `user_id` in the server database.
    1. Sends an ACK to the client.
1. The client closes the session.

[TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/43): Determine failure and retry behavior for this protocol: what is the client behavior after receipt of ACK?

## Cryptographic and Supporting Operations
### External dependencies
This includes:
- Cryptographic Hash Function `Hash`. 
- CSPRNG, `rng`.
- A symmetric AEAD scheme that consists of:
    - An encryption function `Enc` that takes a pair `(key, msg, data)`, where `key` is the symmetric key, `msg` is the message to be encrypted, and `data` is OPTIONAL associated data, and outputs a ciphertext.
    - A decryption function `Dec` that takes a pair `(key, ciphertext, data)`, where `key` is the symmetric key,`ciphertext` is the a ciphertext to be decrypted, and `data` is OPTIONAL associated data, and outputs a plaintext.
- [A HMAC-based key derivation function](https://datatracker.ietf.org/doc/html/rfc5869) that is parameterized by `Hash` and consists of:
    - A key derivation function `HKDF` that takes a tuple `(salt, input_key, context, len)`, where `salt` is an optional, non-secret random value, `input_key` is the input key material, `context` is an optional context and application-specific information, and `len` is the length of the output keying material in bytes.


Inter-dependency constraints include:
- The length of the a key for `Enc` must be no more than 255 times the length of the output of `Hash`.
See [here](current-development-phase.md#cryptographic-protocol-and-implementation-dependencies) for our selections.

### Generate a secret
Input:
  - A length `len` in bytes. Default length is 32 bytes.
  - A seeded CSPRNG `rng`.
  - An optional context string, `context`, that SHOULD include non-sensitive application-specific context and parameter choices for usage of the generated secret.

Output: 
   - An element `arbitrary_key`, where `arbitrary_key` is a tuple consisting of:
       - A randomly generated secret `secret` of length `len` in bytes.
       - The input context string, `context`.
    

Protocol:
  1. Generate `secret` as `len` bytes of randomness output from `rng`.
  1. Output `arbitrary_key` as the pair `(secret, context)`.

Usage guidance:
Code that consumes an `arbitrary_key` SHOULD include validation checks specific to the context before use, i.e., an `arbitrary_key` SHOULD be used only in the context for which it was initially generated.

### Client-side storage

For now, simple clear-text storage is acceptable.
- [TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/39) Include appropriate requirements for client-side secure storage and generate relevant issues in key-mgmt.

### Server-side storage
- [TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/28). Include appropriate requirements for server-side secure storage and generate relevant issues in key-mgmt.



