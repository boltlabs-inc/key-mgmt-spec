
# Generate a secret

The local client component can generate a secret.
1. Generate the secret: using a cryptographically secure pseudo-random number generator with a random seed, generate 256 bits of output.
2. Generate a unique tag to associate with the secret (unique among secrets stored by the local client). A tag is 16 bytes (128 bits) of arbitrary data. It can be generated deterministically (e.g. incrementing by 1 for each new secret) or pseudorandomly.
3. Output the tag and the secret.

# Generate and store a secret locally
The local client component can run this operation.
1. Execute the [generate](#generate-a-secret) protocol.
2. Store the tag and the secret as a pair.

# Generate and store a secret remotely
To store a secret remotely, the local client must already have opened a mutually authenticated session with the key server.

TODO (#4 #10 #27): Describe the authentication flow and outputs precisely. The outputs should include a symmetric encryption key and a user ID.

1. The local client runs the [generate](#generate-a-secret) protocol to get a tag and a secret.
2. The local client encrypts the secret under the symmetric encryption key.
3. The local client send the tag and encrypted secret over the open channel.
4. The key server generates a globally unique identifier for the key by generating 256 bits of randomness, appending the tag to it, and computing the SHA3-256 hash to get an ID for the secret. If the ID is not unique among the key server's stored IDs, the server selects new randomness and recomputes the hash.
5. The key server stores the user ID, the secret ID and encrypted secret. TODO (#28): Specify security requirements for server storage.
6. The key server sends the secret ID to the local client.
