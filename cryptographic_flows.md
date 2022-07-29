
# Operations on Arbitrary Secrets

## Setup
The asset owner MUST have successfully completed [Registration](https://github.com/boltlabs-inc/key-mgmt-spec/issues/11) to generate and store arbitrary secrets at the server.
- [TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/11#issuecomment-1194099508): Registration details needs to be fleshed out.
- [TODO]() Authentication details need to be fleshed out.

We will assume the following:

   - `session_key`, A key that is derived during authentication and is shared between the client and the key server. This key can be used as a key for the AEAD scheme `Enc`. In the following, when the client and key server send each other messages, it is assumed that these messages are encrypted using `Enc` under `session_key`. The key server should additionally reject all messages sent over this channel that fail verification checks.

## Generate and Store a Secret
- [TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/43): Determine failure and retry behavior for this protocol: what is the client behavior after receipt of ACK?

This client-initiated functionality generates a secret locally and stores the result both locally and remotely:

Input:
- `user_id`, a 128-bit globally unique identifier (GUID) representing the identity of the asset owner.

Output:
- `key_id`, a 128-bit unique identifier representing a key, computed as the (possibly truncated) output of `Hash` over user and scheme parameters and a randomly generated salt.

Protocol:
1. The client:
   1. [Opens a request session](systems-architecture.md#request-session) for the given user id `user_id`. 
   1. Calls [`retrieve_storage_key`](#retrieve-storagekey-functionality), the output of which is `storage_key`.
   1. Sends a request message to the key server over the session's secure channel. This message MUST indicate the desire to store a secret remotely and contain `user_id`.
1. The key server:
   1. Runs a validity check on the received request and `user_id`(i.e., there must be a valid open request session, the request must conform to the expected format, and `user_id` must be of the expected format and length, and should match that of the open request session). If this check fails, the server MUST reject the request.
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


## Retrieve a Secret
This client-initiated functionality retrieves a secret from the system.

Input:
- `user_id`, a 128-bit globally unique identifier (GUID) representing the identity of the asset owner.
- `key_id`, a 128-bit unique identifier representing a key.

Output:
- `arbitrary_key`, the arbitrary secret that is backed up remotely.

Protocol:
1. The client:
   1. [Opens a request session](systems-architecture.md#request-session) for the given user id `user_id`. 
   1. Calls [`retrieve_storage_key`](#retrieve-storagekey-functionality), the output of which is `storage_key`. The implementation SHOULD keep this key in memory only and not write to disk.
   1. [Retrieves](#client-side-storage) the ciphertext `ciphertext` associated to `key_id` from local storage. 
        1. If successful, computes `arbitrary_key = Dec(storage_key, ciphertext, user_id||key_id)`, outputs `arbitrary_key`, and closes the request session. 
        1. Otherwise, continues.
   1. Sends a request message to the key server over the open session's secure channel. This message MUST indicate the desire to retrieve the backed up secret and contain `user_id` and `key_id`.
1. The key server:
    1. Runs a validity check on the received request, `user_id`, and `key_id`. That is,
        1. There must be a valid open request session, 
        1. The request must conform to the expected format and content, 
        1. The `user_id` must be of the expected format and length, and should match that of the open request session,
        1. The `key_id` must be associated with the given `user_id` in the key server's database.
       If this check fails, the server MUST reject the request.
    1. [Retrieves](#server-side-storage) the associated ciphertext and associated data for the given pair `(user_id, key_id)` from its database and sends this ciphertext, together with the associated data, to the client.
1. The client:
    1. Computes `arbitrary_key = Dec(storage_key, ciphertext, user_id||"storage key")`, where `ciphertext` is the received ciphertext from the key server.
    1. Deletes `storage_key` from memory.
    1. Closes the open request session.
    1. [Stores](#client-side-storage) `ciphertext` in its local storage.
    1. Outputs `arbitrary_key`.

Usage guidance: The calling application SHOULD enable the asset owner to copy-paste the password to the system clipboard for one-time use and then makes a best effort to delete this secret from memory.
  

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

### `retrieve_storage_key` functionality
This functionality allows the client to request and receive the key `storage_key` from the key server, where `storage_key` is a symmetric key for the implementation's selected AEAD used for remote storage.

Inputs:
- An open request session.
- `user_id`, a universally unique identifer that represents the user.
- `export_key`, a symmetric key from the open request session.

Outputs:
- `storage_key`, a symmetric key for an AEAD scheme.

Protocol:
1. The client:
    1. If the client does not have an [open request session](systems-architecture.md#opening-and-using-a-request-session) with the key server, calling this function should output an error.
    1. Sends a request message to the key server over the open session's secure channel. This message MUST indicate the desire to retrieve the storage key and contain `user_id`.
1. The key server:
    1. Runs a validity check on the received request and `user_id` (i.e., there must be a valid open request session, the request must conform to the expected format and content, and `user_id` must be of the expected format and length, and should match that of the open request session). If this check fails, the server MUST reject the request.
    1. [Retrieves](#server-side-storage) the associated storage key ciphertext for the given user from its database and sends this ciphertext to the client.
1. The client:
    1. Computes `Dec(export_key, ciphertext, user_id||"storage key")`, where `ciphertext` is the received ciphertext from the key server, and outputs `storage_key`.
       - TODO: update this to be consistent with the derived key used instead of `export_key` here.

Usage guidance: Code that calls the `retrieve_storage_key` functionality SHOULD NOT write the output `storage_key` to disk and should make a best effort to drop this key from temporary memory after use of this key is completed.

### Client-side storage

For now, simple clear-text storage is acceptable.
- [TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/39) Include appropriate requirements for client-side secure storage and generate relevant issues in key-mgmt.

### Server-side storage
- [TODO](https://github.com/boltlabs-inc/key-mgmt-spec/issues/28). Include appropriate requirements for server-side secure storage and generate relevant issues in key-mgmt.



