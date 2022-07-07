
# Generate a secret

This interaction is initiated by the local client application, when the user requests to create a new self-custodial or delegated secret.

A similar operation happens when the service provider requests to create a new passive secret via the remote client application. This operation is slightly different and is _not_ addressed here.

1. The calling application initiates generation of a new secret via the local client API. They specify the use permission (self-custodial, delegated), the use restrictions (unilateral control, shared control), and the type of secret (e.g. arbitrary bytes, ECDSA key pair).
2. The local client authenticates with every key server.
2. The local client forwards the request to every key server. Optionally, the local client augments each request with some random bytes.
2. __[Some entity]__ assignes an ID to the secret. (See open questions)
3. If the secret is specified to have shared control, each key server forwards the request (without the augmented random bytes) to the policy engine. The policy engine approves or rejects creation of a new secret for the user.
If a key server receives a rejection from the policy engine, they forward it back to the user and the protocol ends.
4. Within their encalves, the key servers jointly run the secret generation protocol. This _might_ look like: within their enclave, each key server generates some random bytes and combines it with the random bytes provided by the local client to create their share of the secret.
If the secret has a corresponding joint computation by all key servers to create a public component, the key servers run that computation. Each key server should receive the public component as output.
5. Each key server stores the output of the secret generation protocols in their database. In particular, they encrypt the secret using enclave-specific key material, then pass it out of the enclave to be stored.
6. Each key server returns the public portions of the key to the local client application, including the key ID, the public component (if any).
7. The local client returns the public portions to the calling application.

## Open questions
- How is an ID for a secret generated? Which parties participate to select an ID?
- What if some key server fails to respond to the secret generation request? Is there a redundancy or re-sharing operation that can later make sure that server gets a share of the key?
- If some server doesn't have a key share, does any entity keep track of which servers have shares of which keys? What is the mechanism to avoid duplicate key IDs if not every server knows about every key?


