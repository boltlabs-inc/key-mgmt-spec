# Registration
An asset owner that has not previously interacted with the key server MUST register. Registration proceeds as follows:

1. The client:
    1. Generates `user_id`, a globally unique identifier (GUID). 
        - TODO: Specify how. 
    1. [Opens a registration session](systems-architecture.md#opening-a-registration-session) with the key server.
    1. Runs the [generate protocol](cryptographic_flows.md#generate-a-secret) to get a symmetric key `storage_key` for [`Enc`](#external-dependencies) of length 32 bytes.
        - TODO: Include warnings to never store this key or pass it out to calling application.
    1. Computes a ciphertext `encrypted_storage_key = Enc(export_key, storage_key, user_id||"storage key")`. 
        - TODO: change to a key derived from the `export_key` instead. Include a domain separator.
    1. Sends a request message to the key server over the registration session's secure channel. This message MUST indicate the desire to complete registration and contain `user_id` and the ciphertext `encrypted_storage_key`.
    1. Deletes `storage_key` from memory.
        - TODO: Update this when additional requests are allowed.
1. The server:
    1. Checks that `user_id` matches that of the authenticated user in the open registration session. If this check fails, the server MUST abort.
    1. Checks that the received ciphertext is of the expected format and length.
    1. [Stores](#server-side-storage) the received ciphertext in a record associated with `user_id`.
1. The client and server close the session upon completion of the request.

### Retrieve `storage_key` functionality
- TODO: The client will also need a corresponding`retrieve_storage_key` function. This request should take place over an [open request session](systems-architecture.md#opening-and-using-a-request-session).




