# Glossary
**Affiliated asset creator**: An entity with an established relationship with the Service Provider, who wishes to leverage the SP for distribution and use of assets.

**Affiliated transaction creator**: An entity with an established relationship with the Service Provider, who wishes to leverage the SP for processing of transactions that requires key use.

**Asset fiduciary**: An entity with an established relationship with the Service Provider, who has the responsibility and power to examine key use requests and approve or deny these requests.

**Asset owner**: A human that owns digital assets and wishes to store related secrets using Lock Keeper, via a full-featured digital asset manager built on top of Lock Keeper. 

**Channel**: An open channel of communication between a client and a server. Used to facilitate the execution of protocols or workflows via individual messages.

**Client**: Software that runs on an asset owner's device and initiates workflows involving secrets.

**Demo repository**: A [repository](https://github.com/boltlabs-inc/key-mgmt-demo) that demonstrates usage of the client and server libraries.

**Endpoint**: A gRPC method that establishes a session and determines which protocol or workflow will be executed within that session.

**gRPC**: A Remote Procedure Call (RPC) framework developed by Google.

**Key fiduciary**: An entity to whom the asset owner has granted key use authority.

**Key server**: A server that enables both secure storage and use of secrets.

**Message**: A collection of data required to advance a protocol by one step. Messages are sent across channels between a client and server.

**Protocol**: A multi-message process that accomplishes a particular task within a workflow. *Example: Generate a secret*

**Request**: Any operation undertaken by the client that involves communication with the key server, e.g., register, generate and store a secret, retrieve a secret.

**Service provider**: The entity using Lock Keeper to build a full-featured digital asset manager.

**Session**: A logical interaction between a client and one or more servers at the application layer. Sessions use one or more channels to communicate with messages in order to execute protocols.

**Workflow/Flow**: A set of steps that accomplish an action on behalf of a user. Completing a workflow requires the execution of one or more protocols. *Example: Generate and store a secret*
