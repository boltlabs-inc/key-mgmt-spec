# Lock Keeper Specification

This repository holds the current design specification for [Lock Keeper](https://github.com/boltlabs-inc/key-mgmt), a human-centric digital asset management system.

## Goals
We are designing and building Lock Keeper iteratively, so this specification is meant to be a single point of truth for our current development phase.

At a high level, we want to capture information that helps developers and the public understand the properties of our digital asset management system, as well as a formal specification of the underlying cryptographic protocols used. 

## Expected Audience
We want this repository to be a helpful reference for the following groups:
- Developers who are integrating our code into a larger system.
- Cryptographers and security engineers who are auditing our system design and/or code.

## In Scope
Here is a working list of what should be included in this repository:
- An overview of what the product is expected to accomplish in our current development phase. Because we are designing and implementing the product iteratively, this should be updated regularly.
- Integration guidance for developers, including any requirements or assumptions for the calling application(s) and how the calling application(s) fits in the overall system architecture, i.e., which components are external-facing. This will be helpful for iterating on our external-facing APIs and overall system functionality.
- A specification of our threat model and desired system security and privacy properties.
- A specification of the system's application-layer session handling and assumptions on the underlying transport layer. This will be helpful for design iteration, analysis, audits, and obtaining feedback from the community.
- A specification of the system's cryptographic protocol flows. This should include a description of underlying cryptographic assumptions and desired properties. This will be helpful for design iteration, analysis, audits, and obtaining feedback from the community.

## Out of Scope
We do not attempt to capture granular, non-cryptographic implementation and software architecture decisions. This type of development decision should be documented clearly in [Lock Keeper](https://github.com/boltlabs-inc/key-mgmt).






