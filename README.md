
# Lock Keeper Specification

This repository holds the current design specification for [Lock Keeper](https://github.com/boltlabs-inc/key-mgmt), a human-centric digital asset management system.

## Contents

[Problem Statement](#problem-statement)<br>
[Repository Overview](#repository-overview)<br>
[Repository Navigation](#repository-navigation)<br>

## Problem Statement

Humans need to own and control digital assets; doing so securely requires humans to leverage cryptographic key management in order to generate, store, retrieve, and use arbitrary secrets, or keys, associated with these digital assets, while preventing key theft and key misuse. 

In practice, a person’s experience of their digital asset portfolio is often mediated by an organization, or _Service Provider (SP)_. The service provider provides an asset manager to its community members. The asset manager is a specialized key management system that helps people manage and use the secret keys corresponding to their digital assets.

A key point is that the asset manager must meet the needs of both community members and the service provider. The service provider might itself be composed of several entities offering a variety of services and utilities, each with their own set of goals and requirements with respect to how digital assets are created and handled. In order to meet compliance and regulatory frameworks, service providers are particularly concerned with the processes that approve, sign, and transmit transactions (i.e., messages that move assets) on behalf of users. The needs of these various actors sometimes conflict, and a challenge of design in this space includes identifying and selecting the tradeoffs that achieve a balanced, beneficial system for all stakeholders.

We plan to build the core key management technology for a _digital asset management system (DAMS)_, which we call _Lock Keeper_. We stress that this design does not encompass the full details of a digital asset manager, although we do include a wallet application proof of concept (PoC) for reference. Lock Keeper can be used by a Service Provider to create a feature-complete digital asset manager. 

We focus on:

- The design of the core key management technology, namely generation, storage, and use of keys.

- Providing integration guidance and requirements for the Service Provider. This includes establishing the expected interfaces that the core key management technology has with other components of an asset manager. These other components include:

    - Wallet application PoC/reference implementation: This component is responsible for managing end customer account data, handling blockchain transactions (or other messages to be signed), authentication of the origin of submitted transactions from entities other than the end customer, and providing a front-end interface for the end customer to interact with their digital assets. We are currently building a PoC application for demonstration purposes.

    - Policy Engine component: This component is responsible for determining approvals and rejections of key use requests. Lock Keeper integrates a minimal policy engine component that provides lightweight audit capabilities via cryptographic signatures on approval decisions, but complex policy logic must be managed in an external policy engine. We anticipate future development of an in-house policy engine that provides more complex, flexible, and cryptographically rigorous policy enforcement, and as such we sometimes include more detail as to how we expect this policy engine component to interact with other (non-Lock-Keeper) system components.


## Repository Overview
### Goals
We are designing and building Lock Keeper iteratively, so this specification is meant to be a single point of truth for our current development phase.

At a high level, we want to capture information that helps developers and the public understand the properties of our digital asset management system, as well as a formal specification of the underlying cryptographic protocols used. 

### Expected Audience
We want this repository to be a helpful reference for the following groups:
- Developers who are integrating our code into a larger system.
- Cryptographers and security engineers who are auditing our system design and/or code.

### In Scope
Here is a working list of what should be included in this repository:
- An overview of what the product is expected to accomplish in our [current development phase](current-development-phase.md) and the [current functionality](cryptographic-flows.md) of the repository. Because we are designing and implementing the product iteratively, this should be updated regularly.
- Integration guidance for developers, including any requirements or assumptions for the calling application(s) and how the calling application(s) fits in the overall system architecture, i.e., which components are external-facing. This will be helpful for iterating on our external-facing APIs and overall system functionality.
- A specification of our [threat model and desired system security and privacy properties](design-goals.md).
- A specification of the [system architecture](systems-architecture.md), including application-layer session handling and assumptions on the underlying transport layer. This will be helpful for design iteration, analysis, audits, and obtaining feedback from the community.
- A specification of the system's [cryptographic protocol flows](cryptographic_flows.md). This should include a description of underlying cryptographic assumptions and desired properties. This will be helpful for design iteration, analysis, audits, and obtaining feedback from the community.

### Out of Scope
We do not attempt to capture granular, non-cryptographic implementation and software architecture decisions. This type of development decision should be documented clearly in [Lock Keeper](https://github.com/boltlabs-inc/key-mgmt).

## Repository Navigation
[Design Philosophy and Threat Model](design-philosophy-and-threat-model.md)<br>
[Stakeholders and System Goals](stakeholders-and-goals.md)<br>
[Current Development Phase](current-development-phase.md) <br>
[System Architecture](systems-architecture.md)<br>
[System Functionality](cryptographic-flows.md)<br>
[Where to Find Code](repository-list.md)<br>
[Glossary](glossary.md)

