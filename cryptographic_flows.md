
# Authenticate
The local client inputs a user ID and a password and receives an "export key", which we will consider as a valid symmetric encryption key, as output. The key server receives as output some user-specific authentication material, which it can compare to its stored data to verify authentication.

Both parties recieve an open channel as output, which satisfies mutual authentication, confidentiality, and integrity.


# Generate a secret

The local client component can generate a secret. Using a cryptographically secure pseudo-random number generator with a random seed, generate 256 bits of output.

Generate a unique tag to associate with the secret (unique among secrets stored by the local client).

# Generate and store a secret locally
The local client component can run the [generate](generate-a-secret) protocol. Then, store the tag and the secret as a pair.

# Generate and store a secret remotely
The local client must
1. Run the [authenticate](authenticate) protocol to retrieve an export key and an open channel.
2. Run the [generate](generate-a-secret) protocol to get a tag and a secret.
3. Use the export key to encrypt the secret.
4. Send the tag and encrypted secret over the open channel.

The key server generates a unique secret ID for the key by generating 256 bits of randomness and hashing it with the tag to get an ID for the secret (if the ID is not unique among the key server's stored IDs, try again).
Then, store the user ID (from authentication), the secret ID and encrypted secret.

To conclude, the key server sends the secret ID to the local client.
