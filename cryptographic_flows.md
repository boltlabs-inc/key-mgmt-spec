
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
- `user_id`, A (globally unique) user identifier.
    - TODO: What is the length of this user identifier? Is this a global default?

Output:
- `key_tag`: TODO
- `key_id`: TODO

Protocol:
1. The client:
   1. Checks if there is an existing open session with the key server and the input `user_id` and opens a [Request Session](systems-architecture.md#request-session) if not. 
   1. Calls `retrieve_storage_key`, the output of which is `storage_key`.
   1. Runs the [generate](#generate-a-secret) protocol to get a tag `tag` and a secret `arbitrary_key`.
   1. Sends a request message to the key server over the secure channel. This message MUST indicate the desire to store a secret remotely and contain `user_id` and `tag`.
1. The key server:
   1. Runs a validity check on the received `tag` and `user_id` (i.e., `tag` and `user_id` must both be of the expected format and length, and the lengths should additionally match the prepended length indicators, and `user_id` should match the current authenticated session).
   1. Generates `key_id`, a globally unique identifier as follows:
        1. Generates `randomness`, 256 bits of randomness using `rng`.
        1. Computes `key_id = Hash(domain_sep, len||user_id, tag, randomness)`, where `len` is one byte representing the length of the user identifier `user_id`.
        1. This functionality should fail (with negligible probability) if the generated identifier is not unique among the key server's stored identifiers. 
        1. Sends `key_id` to the client over the secure channel.
1. The client:
    1. Computes `Enc(storage_key, key, user_id||tag||key_id)` and sends the resulting ciphertext to the key server over the secure channel.
    1. [Stores](#store-a-secret-locally) the ciphertext and associated data locally.
1. The key server:
    1. Runs a validity check on the received ciphertext (i.e., the ciphertext must be of the expected format and length, and the length should additionally match the prepended length indicators).
    1. Stores a tuple containing the received ciphertext, `tag`, and `user_id` in the server database.
        - [TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/28): Add a reference to the security requirements for server storage when complete.
    1. Sends an ACK to the client.

TODO: retries? what is the client behavior after receipt of ACK?

## Cryptographic and Supporting Operations
### External dependencies
This includes:
- Cryptographic Hash Function `Hash`. 
- CSPRNG, `rng`.
- A symmetric AEAD scheme that consists of:
    - An encryption function `Enc` that takes a pair `(key, msg, data)`, where `key` is the symmetric key, `msg` is the message to be encrypted, and `data` is OPTIONAL associated data, and outputs a ciphertext.
    - A decryption function `Dec` that takes a pair `(key, ciphertext, data)`, where `key` is the symmetric key,`ciphertext` is the a ciphertext to be decrypted, and `data` is OPTIONAL associated data, and outputs a plaintext.

See [here](current-development-phase.md#cryptographic-protocol-and-implementation-dependencies) for our selections.

### Generate a secret
Input:
  - A length `len` in bytes.
  - A seeded CSPRNG `rng`.

Output: 
   - A unique 16-byte tag, `key_tag`.
   - An element `arbitrary_key`, where `arbitrary_key` is a randomly generated secret length `len` in bytes.
   
Protocol:
  1. Generate `arbitrary_key` as 32 bytes of randomness output from `rng`.
  1. Generate `tag` as 16 bytes of arbitrary data, prepended with one bye that represents the length of the tag. This tag MUST be unique among secrets stored by the local client. It can be generated deterministically (e.g. incrementing by 1 for each new secret) or pseudorandomly.
  1. Output `tag` and `arbitrary_key`.

### Store a secret locally

For now, simple clear-text storage is acceptable.
- [TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/39) Include appropriate requirements for client-side secure storage and generate relevant issues in key-mgmt.



