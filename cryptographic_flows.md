
# Generate a secret

The local client component can generate a secret. Using a cryptographically secure pseudo-random number generator with a random seed, generate 256 bits of output.

Generate a unique tag to associate with the secret (unique among secrets stored by the local client). A tag is 16 bytes (128 bits) of arbitrary data. It can be generated deterministically (e.g. incrementing by 1 for each new secret) or pseudorandomly.

# Generate and store a secret locally
The local client component can run the [generate](#generate-a-secret) protocol. Then, store the tag and the secret as a pair.

# Generate and store a secret remotely
To store a secret remotely, the local client must already have opened a mutually authenticated session with the key server.

TODO (#4 #10 #27): Describe the authentication flow and outputs precisely. The outputs should include a symmetric encryption key and a user ID.

2. Run the [generate](#generate-a-secret) protocol to get a tag and a secret.
3. Use the symmetric encryption key to encrypt the secret.
4. Send the tag and encrypted secret over the open channel.

The key server generates a globally unique identifier for the key by generating 256 bits of randomness, appending the tag to it, and computing the SHA3-256 hash to get an ID for the secret. If the ID is not unique among the key server's stored IDs, the server selects new randomness and recomputes the hash.
Then, the key server stores the user ID , the secret ID and encrypted secret.
TODO (#28): Specify security requirements for server storage.

To conclude, the key server sends the secret ID to the local client.
