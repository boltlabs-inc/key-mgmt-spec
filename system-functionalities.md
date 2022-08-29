# System Functionalities
This page contains protocol descriptions for Lock Keeper system functionalities.

## Contents
1. [Register](#register) <br>
1. [Retrieve Audit Logs](#retrieve-audit-logs) <br>
1. [Operations on Arbitrary Secrets](#operations-on-arbitrary-secrets) <br>
    1. [Generate and Store a Secret](#generate-and-store-a-secret) <br>
    1. [Retrieve a Secret](#retrieve-a-secret) <br>
    1. [Import a Secret](#import-a-secret) <br>
1. [Operations on Signing Keys](#operations-on-signing-keys) <br>
1. [Cryptographic and Supporting Dependencies](#cryptographic-and-supporting-operations) <br>
    1. [External Dependencies](#external-dependencies) <br>
    1. [Generate a Secret Helper](#generate-a-secret) <br>
    1. [Generate a Key Identifier](#generate-a-key-identifier) <br>
    1. [`retrieve_storage_key` protocol](#retrieve_storage_key-protocol) <br>
    1. [Client-side storage](#client-side-storage) <br>
    1. [Server-side storage](#server-side-storage) <br>

## Register
An asset owner that has not previously interacted with the key server MUST register. Registration proceeds as follows:

Input:
- Client input:
    - `user_credentials`, which consists of credentials for use in opening [authenticated sessions](systems-architecture.md#networking).
        - `account_name`, bytes that represent the asset owner's human-memorable account information, e.g., email address; and
        - `password`, bytes that represent the asset owner's human-memorable secret authentication information.
- Server input: none

Output:
- Client output:
    - A success indicator.
- Server output:
    - A success indicator.

1. The client [opens a registration session](systems-architecture.md#opening-a-registration-session) with the key server using credentials `user_credentials`. The client receives as output an open secure channel and a user identifier `user_id`.
1. The client:
    1. Derives `master_key`, a symmetric key of length `len` bytes for [`Enc`](#external-dependencies), from  `export_key`, as follows:
        1. Length `len` should be the default for `Enc`.
        1. Set `master_key = KDF(export_key, "OPAQUE-derived Lock Keeper master key", len)`.
    1. Runs the [generate protocol](cryptographic_flows.md#generate-a-secret) to get a symmetric key `storage_key` for [`Enc`](#external-dependencies) of length 32 bytes.
        - The `storage key` MUST NOT be saved, stored, or used in any context outside this protocol. It must not be passed to the calling application.
    1. Computes a ciphertext `encrypted_storage_key = Enc(master_key, storage_key, user_id||"storage key")`.
    1. <a name="complete-registration"></a> Sends a request message to the key server over the registration session's secure channel. This message MUST indicate the desire to _complete registration_ and contain `user_id` and the ciphertext `encrypted_storage_key`.
    1. Deletes `storage_key` and `master_key` from memory.
        - [TODO #51](https://github.com/boltlabs-inc/key-mgmt-spec/issues/51): Update this when additional requests are allowed.
1. The server:
    1. Checks that `user_id` matches that of the authenticated user in the open registration session. If this check fails, the server MUST abort.
    1. Checks that the received ciphertext is of the expected format and length.
    1. [Stores](#server-side-storage) the received ciphertext in a record associated with `user_id`.
    1. Stores the registration registration request in the [audit log](#audit-logs).
    1. Outputs a success indicator.
1. The client and server close the session upon completion of the request.
1. The client outputs a success indicator to the calling application.

## Retrieve Audit Logs
The asset owner can request audit logs from the key server.

Input:
- Client Input:
    - `user_credentials`, which consists of credentials for use in opening [authenticated sessions](systems-architecture.md#networking).
        - `account_name`, bytes that represent the asset owner's human-memorable account information, e.g., email address; and
        - `password`, bytes that represent the asset owner's human-memorable secret authentication information.
    - `type`, one of:
        - `"system only"`, which indicates the asset owner wants a record of registration, logins, and audit log requests;
        - `"key only"`, which indicates the asset owner wants a record of requested key use operations with respect to one or more keys; or
        - `"all"`, which indicates the asset owner wants both system and requested key use operations.
    - `key_identifiers`, an OPTIONAL list of key identifiers. If no key identifier is provided, both the `"all"` and `"key only"` options above will return logs for all keys.
    - `after_date`, an OPTIONAL date-time input to ask for logs after a certain date.
    - `before_date`, an OPTIONAL date-time input to ask for logs before a certain date. If both `after_date` and `before_date` are provided, logs from within that date range will be returned.
- Server Input: none.

Output: 
- Client output:
    - `summary_record`, which contains the requested history, including timestamps.
- Server output: 
    - A success indicator.

Protocol:

1. The client:
   1. [Opens a request session](systems-architecture.md#request-session) for the given credentials `user_credentials`. The client receives as output an open secure channel and a user identifier `user_id`.
   1. Sends a request message to the key server over the session's secure channel. This message MUST indicate the desire to retrieve audit logs, specify the type of audit requested (i.e., `"system only"`, `"key only"`, or `"all"`) and contain `user_id`.
1. The key server:
   1. Runs a validity check on the received request and `user_id`(i.e., there must be a valid open request session, the request must conform to the expected format, and `user_id` must be of the expected format and length, and should match that of the open request session). If this check fails, the server MUST reject the request.
   1. Retrieves all log records relevant to the client's request, summarises these records in `summary_record`, and sends `summary_record` to the client over the secure channel.
   1. Creates and stores an [audit log](#audit-logs) entry for the current request, including the outcome of the validity check.
   1. Outputs a success indicator.
 1. The client:
    1. Closes the session.
    1. Outputs `summary_record` to the calling application.


## Operations on Arbitrary Secrets
### Generate and Store a Secret
- [TODO #43](https://github.com/boltlabs-inc/key-mgmt-spec/issues/43): Determine failure and retry behavior for this protocol: what is the client behavior after receipt of ACK?

This client-initiated functionality allows for two ways of generating and storing a secret: 
- local generation with remote backup; and
- remote-only generation and storage.

#### Local secret generation with remote backup
This client-initiated functionality generates a secret locally and stores the result both locally and remotely.

Input:
- Client input:
    - `user_credentials`, which consists of credentials for use in opening [authenticated sessions](systems-architecture.md#networking).
        - `account_name`, bytes that represent the asset owner's human-memorable account information, e.g., email address; and
        - `password`, bytes that represent the asset owner's human-memorable secret authentication information.
- Server input: none.

Output:
- Client output:
    - `key_id`, a 128-bit unique identifier representing a secret, computed as the (possibly truncated) output of `Hash` over user and scheme parameters and a randomly generated salt.
- Server output:
    - A success indicator.

Protocol:
1. The client:
   1. [Opens a request session](systems-architecture.md#request-session) for the given credentials `user_credentials`. The client receives as output an open secure channel and a user identifier `user_id`.
   1. Calls [`retrieve_storage_key`](#retrieve_storage_key-protocol), the output of which is `storage_key`.
   1. Sends a request message to the key server over the session's secure channel. This message MUST indicate the desire to store a secret remotely and contain `user_id`.
1. The key server:
   1. Runs a validity check on the received request and `user_id`(i.e., there must be a valid open request session, the request must conform to the expected format, and `user_id` must be of the expected format and length, and should match that of the open request session). If this check fails, the server MUST reject the request.
   1. Runs the [Generate a key identifier](#generate-a-key-identifier) subprotocol, the output of which is a globally unique identifier `key_id`.
   1. Sends `key_id` to the client over the secure channel.
1. The client:
    1. Runs the [generate](#generate-a-secret) protocol on input `(32, rng, user_id||key_id)` to get a secret `arbitrary_key`.
    1. Computes `Enc(storage_key, arbitrary_key, user_id||key_id)` and sends the resulting ciphertext to the key server over the secure channel.
    1. [Stores](#client-side-storage) the ciphertext and associated data `user_id||key_id` locally.
1. The key server:
    1. Runs a validity check on the received ciphertext (i.e., the ciphertext must be of the expected format and length).
    1. [Stores](#server-side-storage) a tuple containing the received ciphertext, `user_id`, and `key_id` in the server database.
    1. Stores the current request information, including the outcome of the validity check, in an [audit log](#audit-logs) associated with the given user.    
    1. Sends an ACK to the client.
    1. Outputs a success indicator.
1. The client:
    1. Closes the session.
    1. Outputs `key_id` to the calling application.

#### Remote-only secret generation and storage
This client-initiated functionality sends a request to the key server to generate and store a secret remotely.

Input:
- Client input:
    - `user_credentials`, which consists of credentials for use in opening [authenticated sessions](systems-architecture.md#networking).
        - `account_name`, bytes that represent the asset owner's human-memorable account information, e.g., email address; and
        - `password`, bytes that represent the asset owner's human-memorable secret authentication information.
- Server input: none.

Output:
- Client output:
    - `key_id`, a 128-bit unique identifier representing a secret, computed as the (possibly truncated) output of `Hash` over user and scheme parameters and a randomly generated salt.
- Server output:
    - A success indicator.

Protocol:
1. The client:
    1. [Opens a request session](systems-architecture.md#request-session) for the given credentials `user_credentials`. The client receives as output an open secure channel and a user identifier `user_id`.
    1. Sends a request message to the key server over the session's secure channel. This message MUST indicate the request to generate and store a secret remotely, and contain `user_id`.
1. The key server:
    1. Runs a validity check on the received request and `user_id`(i.e., there must be a valid open request session, the request must conform to the expected format, and `user_id` must be of the expected format and length, and should match that of the open request session). If this check fails, the server MUST reject the request.
    1. Runs the [generate a key identifier](#generate-a-key-identifier) subprotocol, the output of which is a globally unique identifier `key_id`.
    1. Runs the [generate](#generate-a-secret) protocol on input `(32, rng, user_id||key_id)` to get a secret `arbitrary_key`.
    1. [Stores](#server-side-storage) a tuple containing `arbitrary_key`, `user_id`, and `key_id` in the server database.
    1. Sends `key_id` to the client over the secure channel.
    1. Stores the current request information, including the outcome of the validity check, in an [audit log](#audit-logs) associated with the given user.    
    1. Outputs a success indicator.
1. The client:
    1. [Stores](#client-side-storage) the received `key_id` locally.
    1. Closes the session.
    1. Outputs `key_id` to the calling application.

### Retrieve a Secret 
This client-initiated functionality retrieves a secret from the system and passes use-specific information to the calling application.

Input:
- Client input:
    - `user_credentials`, which consists of credentials for use in opening [authenticated sessions](systems-architecture.md#networking).
        - `account_name`, bytes that represent the asset owner's human-memorable account information, e.g., email address; and
        - `password`, bytes that represent the asset owner's human-memorable secret authentication information.
    - `key_id`, a 128-bit unique identifier representing a secret.
    - `context`, one of `NULL`, `"local only"`, or `"export"`, which should indicate the asset owner's intended use of the secret:
        - `NULL`: This option captures internal use of this workflow, i.e., there is no specific asset owner intent specified.
        - `"local only"`: This option captures immediate uses of the secret that are either local to the calling application or the asset owner's system. 
        - `"export"`: This option captures other, potentially non-local uses of the secret, i.e., the value is expected to be passed and persist outside of the calling application.
- Server input: none.

Output:
- Client output:
    - Either a success indicator OR `arbitrary_key`, the arbitrary secret that is backed up remotely.
- Server output:
    - A success indicator.

Protocol:
1. The client:
   1. [Opens a request session](systems-architecture.md#request-session) for the given credentials `user_credentials`. The client receives as output an open secure channel and a user identifier `user_id`.
   1. Calls [`retrieve_storage_key`](#retrieve_storage_key-protocol), the output of which is `storage_key`. The implementation SHOULD keep this key in memory only and not write to disk.
   1. [Retrieves](#client-side-storage) the ciphertext `ciphertext` associated to `key_id` and the associated data `associated_data` from local storage. 
        1. If successful, computes `arbitrary_key = Dec(storage_key, ciphertext, associated_data)`, outputs `arbitrary_key`, and closes the request session. 
        1. Otherwise, continues.
   1. Sends a request message to the key server over the open session's secure channel. This message MUST indicate the desire to retrieve the backed up secret and contain `user_id` and `key_id`.
1. The key server:
    1. If the client closes the request session, outputs a success indicator and halts. Otherwise, continues.
    1. Runs a validity check on the received request, `user_id`, and `key_id`. If this check fails, the server MUST reject the request. The validation check includes the following:       
        1. There must be a valid open request session, 
        1. The request must conform to the expected format and content, 
        1. The `user_id` must be of the expected format and length, and should match that of the open request session,
        1. The `key_id` must be associated with the given `user_id` in the key server's database.
        1. The intended use must match one of the expected options.
    1. [Retrieves](#server-side-storage) the associated ciphertext, `ciphertext` and associated data, `associated_data`, for the given pair `(user_id, key_id)` from its database and sends `ciphertext` and `associated data` to the client.
    1. Stores the current request information, including the outcome of the validity check, in an [audit log](#audit-logs) associated with the given user.  
    1. Outputs a success indicator.
1. The client:
    1. Computes `arbitrary_key = Dec(storage_key, ciphertext, associated_data)`, where `ciphertext` is the received ciphertext from the key server and `associated_data` is the associated data received from the server.
    1. Deletes `storage_key` from memory.
    1. Closes the open request session.
    1. [Stores](#client-side-storage) `ciphertext` in its local storage.
    1. If `context` is set to `NULL`, outputs a success indicator to the calling application and halts.
    1. Otherwise, 
        1. If `context` is set to `"local only"`, outputs `arbitrary_key` to the calling application.
        1. If `context` is set to `"export"`, the client computes `exported key` as `len || secret`, as described above, and outputs `exported_key` to the calling application.

Usage guidance: The calling application SHOULD support the intended use of the asset owner in as security conscious manner as possible. That is:
- If the `context` is set to `"local only"`, the calling application SHOULD enable the asset owner to copy-paste the password to the system clipboard for one-time use and then makes a best effort to delete this secret from memory.
- If the `context` is set to `"export"`, the calling application SHOULD make a best effort to delete the exported secret from memory immediately after use, e.g., after passing the exported secret (or a value derived from this exported secret) out of the calling application itself.

Non-normative notes: 
- The context `context` is meant to allow the asset owner the ability to store state at the server as to the intended use of their secret and does NOT provide assurance that the intended use was respected. The calling application SHOULD make a best effort to support the intended use in as conservative a manner as possible:
    - The context `"local only"` indicates that secret material is meant to be passed to the calling application, and possibly the asset owner's device system, and used in a limited fashion. The calling application SHOULD allow either no or very restricted use of this value outside of the calling application itself, e.g., passing the value in order to allow one-time use of the system clipboard is acceptable, but passing this material over a network connection is not.
    - The context `"export"` indicates that the secret material is meant to be passed to and out of the calling application, and persist _outside of the calling application_ itself. This is meant to capture uses like the asset owner creating an external backup of the system or as part of removing the secret from the key server entirely.
    - The context `NULL` is for internal testing and system extensibility purposes and does not currently provide a function for the asset owner.

### Import a Secret

This functionality allows an asset owner to _import_ a secret to the system. The imported secret is stored both locally and remotely at the key server.

Input:
- Client input:
    - `user_credentials`, which consists of credentials for use in opening [authenticated sessions](systems-architecture.md#networking).
        - `account_name`, bytes that represent the asset owner's human-memorable account information, e.g., email address; and
        - `password`, bytes that represent the asset owner's human-memorable secret authentication information.
    - `secret`, the secret that is being imported, which is of the form ``len || secret_material``, where `len` is 1 byte that represents the length of the secret material `secret_material` in bytes.
- Server input: none.

Output:
- Client output:
    - `key_id`, a 128-bit unique identifier representing a key, computed as the (possibly truncated) output of `Hash` over user and scheme parameters and a randomly generated salt.
- Server output:
    - A success indicator.

Protocol:
1. The client:
   1. [Opens a request session](systems-architecture.md#request-session) for the given credentials `user_credentials`. The client receives as output an open secure channel and a user identifier `user_id`.
   1. Calls [`retrieve_storage_key`](#retrieve_storage_key-protocol), the output of which is `storage_key`.
   1. Sends a request message to the key server over the session's secure channel. This message MUST indicate the desire to store an imported secret remotely and contain `user_id`.
1. The key server:
   1. Runs a validity check on the received request and `user_id` (i.e., there must be a valid open request session, the request must conform to the expected format, and `user_id` must be of the expected format and length, and should match that of the open request session). If this check fails, the server MUST reject the request.
   1. Runs the [Generate a key identifier](#generate-a-key-identifier) subprotocol, the output of which is a globally unique identifier `key_id`. 
   1. Sends `key_id` to the client over the secure channel.
1. The client:
    1. Computes `Enc(storage_key, secret, user_id||key_id||"imported key")` and sends the resulting ciphertext to the key server over the secure channel.
    1. [Stores](#client-side-storage) the ciphertext and associated data `user_id||key_id||"imported key"` locally.
1. The key server:
    1. Runs a validity check on the received ciphertext (i.e., the ciphertext must be of the expected format and length).
    1. [Stores](#server-side-storage) a tuple containing the received ciphertext, and the associated data `user_id||key_id||"imported key"`in the server database.
    1. Stores the current request information, including the outcome of the validity check, in an [audit log](#audit-logs) associated with the given user.    
    1. Sends an ACK to the client.
    1. Outputs a success indicator.
1. The client:
    1. Closes the session.
    1. Outputs `key_id` to the calling application.

Non-normative note: The additional context `"imported key"` provides assurance and usage information for the asset owner and does not provide any additional assurance for the key server. That is, the asset owner, even after recovering from a lost device, is still able to retrieve and leverage this contextual information in deciding how to use the given secret.

## Operations on Signing Keys

All of the [protocols for arbitrary secrets](#operations-on-arbitrary-secrets) should be supported. The only difference is the type of key created and used. That is, calling the generation functionality should allow the asset owner to create one of the following ECDSA key types:
- ECDSA on secp256 curve; or
- EdDSA on ed25519.

Signing keys have an additional supported operation, namely, the creation of a signature.

Implementation guidance: We pick these parameters in order to provide functionality compatible with EVM-based blockchains. However, we expect to support multiple blockchains and signing primitives in the future.

## Cryptographic and Supporting Operations
#### External dependencies
See [the current development phase](current-development-phase.md#cryptographic-protocol-and-implementation-dependencies) for our selections. Dependencies include:

- Cryptographic Hash Function `Hash`. 
- CSPRNG, `rng`.
- A symmetric AEAD scheme that consists of:
    - An encryption function `Enc` that takes a pair `(key, msg, data)`, where `key` is the symmetric key, `msg` is the message to be encrypted, and `data` is OPTIONAL associated data, and outputs a ciphertext.
    - A decryption function `Dec` that takes a pair `(key, ciphertext, data)`, where `key` is the symmetric key,`ciphertext` is the a ciphertext to be decrypted, and `data` is OPTIONAL associated data, and outputs a plaintext.
- [A key derivation function (KDF)] that takes a tuple `(input_key, context, len)`, where `input_key` is the input key material, `context` is an optional context and application-specific information, and `len` is the length of the output keying material in bytes.
   

Inter-dependency constraints include:
- The length of the a key for `Enc` must be no more than 255 times the length of the output of `Hash`.

#### Generate a secret helper
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

Implementation guidance: Care should always be taken when seeding a CSPRNG to ensure that the seed has sufficient entropy. Using a PRNG that reseeds periodically may be advisable.

#### Generate a key identifier 
This subprotocol is run by the key server to generate a 32-byte globally unique identifier, `key_id`, as follows:
1. Generates `randomness` as 32 bytes of randomness output from `rng`.
1. Computes `key_id` as `Hash(domain_sep, 16, user_id, 32, randomness)`, truncated to 128-bits if necessary, where `domain_sep` is a static string acting as a domain separator, prepended with its length in bytes.

This functionality should fail (with negligible probability) if the generated identifier is not unique among the key server's stored identifiers. 


#### `retrieve_storage_key` protocol
This functionality allows the client to request and receive the key `storage_key` from the key server, where `storage_key` is a symmetric key for the implementation's selected AEAD used for remote storage.

Input:
- Client Input:
    - An open request session.
    - `user_id`, a universally unique identifer that represents the user.
    - `export_key`, a symmetric key from the open request session.
- Server Input:
    - An open request session.

Output:
- Client Output:
    - `storage_key`, a symmetric key for an AEAD scheme.
- Server Output:
    - A success indicator.

Protocol:
1. The client:
    1. If the client does not have an [open request session](systems-architecture.md#opening-and-using-a-request-session) with the key server, calling this function should output an error.
    1. Sends a request message to the key server over the open session's secure channel. This message MUST indicate the desire to retrieve the storage key and contain `user_id`.
1. The key server:
    1. Runs a validity check on the received request and `user_id` (i.e., there must be a valid open request session, the request must conform to the expected format and content, and `user_id` must be of the expected format and length, and should match that of the open request session). If this check fails, the server MUST reject the request.
    1. [Retrieves](#server-side-storage) the associated storage key ciphertext, `ciphertext` for the given user from its database and sends `ciphertext` to the client.
    1. Outputs a success indicator.
1. The client:
    1. Derives `master_key`, a symmetric key of length `len` bytes for [`Enc`](#external-dependencies), from `export_key`, as follows:
        1. Length `len` should be the default for `Enc`.
        1. Set `master_key = KDF(export_key, "OPAQUE-derived Lock Keeper master key", len)`.
    1. Computes `storage_key = Dec(master_key, ciphertext, user_id||"storage key")`, where `ciphertext` is the received ciphertext from the key server.
    1. Deletes `master_key` from memory.
    1. Outputs `storage_key`.

Usage guidance: Code that calls the `retrieve_storage_key` protocol SHOULD NOT write the output `storage_key` to disk and should make a best effort to drop this key from temporary memory after use of this key is completed.

#### Client-side storage

For now, simple clear-text storage is acceptable.
- [TODO #39](https://github.com/boltlabs-inc/key-mgmt-spec/issues/39): Include appropriate requirements for client-side secure storage and generate relevant issues in key-mgmt.

#### Server-side storage
- [TODO #28](https://github.com/boltlabs-inc/key-mgmt-spec/issues/28). Include appropriate requirements for server-side secure storage and generate relevant issues in key-mgmt.
- All data stored by the server MUST be encrypted by a key known only to the server.
- System requirements may change to include the usage of secure enclaves, in which case, the encryption key MUST be known only to the enclave.

##### Audit Logs
The server storage should include a per-user audit log that tracks system registration and logins, key use requests, and audit log requests. A single log entry contains the following information about each action: action, actor, date, outcome (e.g. success or failure), any related key identifier.



