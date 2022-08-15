# Glossary
**Affiliated asset creator**: An entity with an established relationship with the Service Provider, who wishes to leverage the SP for distribution and use of assets.

**Affiliated transaction creator**: An entity with an established relationship with the Service Provider, who wishes to leverage the SP for processing of transactions that requires key use.

**Asset fiduciary**: An entity with an established relationship with the Service Provider, who has the responsibility and power to examine key use requests and approve or deny these requests.

**Asset owner**: A human that owns digital assets and wishes to store related secrets using Lock Keeper, via a full-featured digital asset manager built on top of Lock Keeper. 

**Client**: Software that runs on an asset owner's device and initiates workflows involving secrets.

**Demo repository**: A repository that demonstrates usage of the client and server libraries. <br>
- [TODO #93](https://github.com/boltlabs-inc/key-mgmt/issues/93): Add location after set-up.

**Endpoint**: A gRPC method that establishes a session and determines which protocol or workflow will be executed within that session.

**Key fiduciary**: An entity to whom the asset owner has granted key use authority.

**Key server**: A server that enables both secure storage and use of secrets.

**Protocol**: A multi-message process that accomplishes a particular task within a workflow. *Example: Generate a secret*

**Service provider**: The entity using Lock Keeper to build a full-featured digital asset manager.

**Session**: An open channel of communication between a client and a server. Used to facilitate the execution of protocols or workflows.

**Workflow/Flow**: A set of potentially non-linear steps that accomplish an action on behalf of a user. Completing a workflow requires the execution of one or more protocols. *Example: Generate and store a secret*
