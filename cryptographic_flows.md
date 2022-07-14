
# Authenticate
The local client inputs a password and receives an "export key", which we will consider as a series of bytes, as output. The key server receives as output some user-specific authentication material, which it can compare to its stored data to verify authentication.


# Generate a secret

This interaction is initiated by the local client application, when the user requests to create a new self-custodial secret.

1. The calling application initiates generation of a new secret via the local client API. They specify the type of secret (e.g. arbitrary bytes, ECDSA key pair) and provide a communication session with the key server.

2. The local client [_authenticates_](#authenticate) with the key server and receives an export key. It transforms the export key into an [encyption scheme] key.

3. The local client indicates to the key server that they would like to generate a secret. The server sends random bytes.

4. The local client runs the key generation protocol. The input is a random number generator and the random bytes from the server. The protocol computes a deterministic, pseudorandom function over the inputs, the output of which is the arbitrary secret. 
If the secret is a type other than "arbitrary bytes," the key server may do additional processing on the output to make sure it meets the requirements of the secret type and, if appropriate, computes the corresponding public component. 

5. The local client encrypts the secret using user-specific authentication material from the communication session. They send the encrypted secret, the optional public component, and their user ID to the key server.

6. The key server selects a unique key ID (among its existing set of IDs for secrets). It verifies that the user ID matches the authenticated user, then stores the user ID, the key ID, the encrypted secret, and the (optional) public component in their database. 

7. The key server returns the key ID to the local client application.

8. The local client returns the public portions of the new secret, including the key ID and the public component (if any), to the calling application.
