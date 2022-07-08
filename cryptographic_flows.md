
# Authenticate
This interaction is intiated by the local client application.

1. The calling application requests to create a new open session via the local client API. They specify the user ID and password.

2. The local client initiates OPAQUE with the key server. They form a credential request based on the password and send it with the user ID to the key server.

3. The key server validates the credential request. If it is for a user that doesn't exist, they terminate the connection. Otherwise, they form and send a credential response. OQ: does this happen even if the password is wrong?

4. The local client recieves the credential response. To complete registration, they form and send a credential finalization. From the finalization, they extract a session key.

5. The key server recieves the finalization and checks it (?). OQ: Does the server get any output?


# Generate a secret

This interaction is initiated by the local client application, when the user requests to create a new self-custodial secret.

1. The calling application initiates generation of a new secret via the local client API. They specify the type of secret (e.g. arbitrary bytes, ECDSA key pair) and provide a communication session with the key server.

2. The local client forwards the request to key server. Optionally, the local client augments the request with some random bytes.
Possible failures: the session authentication fails or times out. The networking layer times out.

2. The key server selects a unique ID (among its existing set of secret IDs) and assigns it to the secret.

4. The key server runs the key generation protocol: the key server generates some random bytes and combines it with the random bytes provided by the local client to create the secret.
If the secret is an ECDSA key pair, the key server may do additional processing on the secret to make sure it meets the requirements of an ECDSA private key and computes the corresponding public key. 

5. The key server encrypts the secret using user-specific authentication material from the communication session.
The key server stores the user ID, the key ID, the encrypted secret, and the (optional) public component in their database. 

6. The key server returns the public portions of the key to the local client application, including the key ID and the public component (if any).

7. The local client returns the public portions to the calling application.

## Aspects not included here
- Use permissions: there is no support for delegated keys at this point.
- Policy engine: without a policy engine, we don't have to specify shared vs unilateral control.
- Remote client application: the remote client can create passive secrets, but it's not incorporated here and will use a slightly different flow
- Enclaves: The key server runs all operations on normal, untrusted hardware.
