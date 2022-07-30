# Where to Find Code

## Lock Keeper Libraries
Our libraries exist in the [`key-mgmt`](https://github.com/boltlabs-inc/key-mgmt) repository.
- [`local-client`](https://github.com/boltlabs-inc/key-mgmt/tree/develop/dams-local-client): This is a Rust library that can be integrated into a human-facing calling application that runs on an asset owner's device. All operations on the key server are initiated by this client. See [here](systems-architecture.md#local_client) for more details.
- [`key-server`](https://github.com/boltlabs-inc/key-mgmt/tree/develop/dams-key-server): This library can be integrated into a host server in order to provide end-to-end encrypted storage of secrets. See [here](systems-architecture.md#key_server) for more details.

## Demo Repository
A [repository]() that demonstrates usage of the `local-client` and `key-server` libraries.<br>
- [TODO #93](https://github.com/boltlabs-inc/key-mgmt/issues/93) Add link to location above.

This demo contains two parts:
1. A sample human-facing calling application that integrates the `local-client` library. That is, this application provides an interface between the asset owner and the key-mgmt `local-client` API.
1. A demonstration setup for the key server.
1. A Dockerfile for testing the sample calling app with the `local-client` library and key server on any machine.