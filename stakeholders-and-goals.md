# Stakeholders and System Goals

Our system serves both direct and indirect stakeholders. _Direct stakeholders_ refer to entities who directly interact with our code, either through using library code or by making explicit requests to our APIs.  _Indirect stakeholders_ are entities who do not interact directly with our code, but still have stake in what the code does and how it works. That is, indirect stakeholders are not involved in _building_ the full digital asset management application, but instead interact with the asset manager provided by the Service Provider. 

In the following, we first define our stakeholders and consider their needs, and then present our system design goals, which attempt to balance the (sometimes conflicting) needs of our stakeholders.

[Stakeholders](#stakeholders)<br>
[System Goal Summary](#system-goal-summary)<br>

## Stakeholders
[The Service Provider](#the-service-provider)<br>
[Asset Owners](#asset-owners)<br>
[Affiliated Asset Creator](#affiliated-asset-creator)<br>
[Affiliated Transaction Creator](#affiliated-transaction-creator)<br>
[Key Fiduciary](#key-fiduciary)<br>
[Asset Fiduciary](#asset-fiduciary)<br>

### The Service Provider

Our direct stakeholder is the _Service Provider_ who seeks to provide a full digital asset management system. As such, the direct users of Lock Keeper are the _system developers_ working to actually integrate our software and define APIs for other, auxiliary developers (e.g., internal service provider developers, external partner developers building system-level architecture); these users directly call Lock Keeper code or use Lock Keeper APIs.

From the perspective of the service provider, system goals include:
- _Confidentiality, Integrity, and Availability_: Any secret values held on behalf of the digital asset owner must be accessible only to the extent necessary in order to achieve the desired functionality and only by the entity that offers the given functionality in question. Within these constraints, the (correct) secret values should always be accessible.
- _Usability_: 
    - The cryptographic/product API should be usable by developers without an extensive background in cryptography, e.g., the API should be misuse-resistant and well-documented.
    - The cryptographic/product API should allow the service provider to provide usable options for meaningful control of digital assets to their customers, including key recovery options. That is, the service provider’s goals are aligned with those of the digital asset owner’s above, but the service provider may have other requirements that conflict with this.
- _Scalability_: The system should be efficient and scale gracefully with the number of users with whom the entity interacts.
- _Transparency_: System processes should provide audit and logging support, particularly with respect to use of secret keys and/or values (i.e., signatures created). To the extent possible, misbehaving entities should be identifiable to allow for system recovery.
- _Trust assumptions_: The organization is security conscious and wants to limit trust and risk as much as possible. That is, the SP:
    - Is willing to trust audited and “reputable” software libraries and applications to perform the functions as stated, but would prefer software to be open source.
    - Assumes that internal actors are sometimes dishonest. e.g., Internal actors may try to deny participating in the signing of transactions, attempt to commit fraud, or otherwise obtain information to which they should not have access.
    - If using internal cloud infrastructure, is willing to optimistically trust server infrastructure, but wants organizational transparency.
    - If using external cloud infrastructure, is willing to trust this cloud infrastructure to be honest-but-curious. May prefer to distribute trust across multiple jurisdictions/entities. To the extent possible, wants to be able to identify and recover from misbehaving servers.
- _Risk appetite_: The service provider wants to limit exposure and liability for digital assets as much as possible. In particular, the SP wants to be compliant with all regulatory frameworks to which the service provider is or may be subject.

### Asset Owners
Humans who interact with various apps in the Service Provider ecosystem and "own" associated digital assets. We are concerned with the subset of asset owners who wish to store the keys that correspond to these digital assets in Lock Keeper. These stakeholders are served by Lock Keeper indirectly as they interact with the full asset management application, which in turn integrates Lock Keeper as a building block. 

Asset owners do not have to directly call our code to use their asset management application. However, we want to be careful our design is intentional with respect to this category and to provide clear integration guidance for Service Providers (i.e., the entity that builds the full asset management application).

From the perspective of the human, or digital asset owner, the system goals include:
- _Confidentiality, Integrity, and Availability_: The digital asset secret key must be accessible only by the human in question. The (correct) key should always be accessible to the human. The user should be able to recover their key even if they lose their devices and should have some flexibility in determining their threat model with respect to recovery.
- _Usability_: The human should be able to interact with the key without having specialized hardware, understanding cryptographic techniques, or having a perfect memory. The system should accept that some users are non-technical and that all humans are error-prone. In particular, the system should function as intended even if human-chosen randomness has insufficient entropy. Additionally, humans should have some flexibility in determining their own preferences with respect to tradeoffs between usability, availability, and other security and privacy goals.
- _Scalability_: The system should be efficient and scale gracefully with the number of users. 
- _Transparency_: Asset owners should have the ability to audit use of secret keys and/or values (e.g., signatures created). 
- _Trust assumptions_: The human may be security conscious and want to limit trust as much as possible. That is,
    - Willing to trust audited and “reputable” software libraries and applications to perform the functions as stated, but would prefer software to be open source. 
    - Willing to trust server infrastructure to be honest-but-curious. May prefer to distribute trust across multiple jurisdictions/entities.
    - May be technically sophisticated enough to want the ability to verify Lock Keeper software.

### Affiliated Asset Creator
An entity that creates digital assets for distribution in conjunction with the Service Provider. An asset creator's goals with respect to the system are consistent with the Service Provider's goals, with an emphasis on usability and performance.

### Affiliated Transaction Creator
This entity uses the Service Provider’s asset manager to submit transactions (that impact assets) for processing, including signing. 
A transaction creator's goals with respect to the system are consistent with the Service Provider's goals, with an emphasis on usability and performance.

### Key Fiduciary
A _key fiduciary_ is an entity with automated key use authority for a given key. For example, for a signing key, a key fiduciary may submit messages to the system in an automated fashion in order to obtain a signature under the given key. In Lock Keeper, this authority MUST be granted directly by the asset owner.

Keys that have named key fiduciaries are a type of _delegated_ key. In other words, we treat these entities as having a type of key custody, because they have broad ability to use the key in question. This type of entity does not necessarily have a built-in ability to recover the secret key material and the use of secure processors may be leveraged to prevent this type of key access; this is a design choice of the underlying system architecture. More nuanced treatment of key fiduciaries is a promising area for future extensions.

Key fidicuciaries have similar goals to the Service Provider, emphasizing the following:
- Wants access to a usable and performant audit capability that allows the key fiduciary to inspect approvals for use of a delegated key.
- Wants delegated key use to require unforgeable proof of approval by the key fiduciary, so that responsibility for key use is unambiguous.
- Wants a seamless method for automated submission of key use requests to the system.

### Asset Fiduciary
An entity who has veto power over requested asset transactions (e.g. Service Provider compliance entities).  This entity can approve or deny transaction signature requests from key fiduciaries. 

Asset fiduciaries have similar goals to the Service Provider, emphasizing the following:
- Wants access to a usable and performant audit capability that allows the asset fiduciary to inspect key activity and key use.
- Wants key use to require unforgeable proof of approval by the asset fiduciary, so that responsibility for key use (e.g., approved transactions) is unambiguous.
- Wants a seamless method to integrate both automated and manual external policy enforcement on key use.

## System Goal Summary

We want our design to be a flexible building block for realizing an asset manager. This includes support for arbitrary secrets, multiple custody options for keys, and optionality for shared control of key use.

[Support for Arbitrary Secrets](#support-for-arbitrary-secrets)<br>
[Self-custody and Delegated-custody Keys](#self-custody-and-delegated-custody-keys)<br>
[Optionality for Shared Control of Key Use](#optionality-for-shared-control-of-key-use)<br>


### Support for Arbitrary Secrets
Lock Keeper should include underlying cryptographic support for storing and retrieving arbitrary secrets. This preserves our ability to extend the system in a flexible manner that meets the needs of a range of service providers.  

### Self-custody and Delegated-custody Keys
We plan to support both _self custody_ and _delegated custody_ of keys. In the former, the service provider does not have access to the secret key material. In the latter, the service provider does have access to the secret key, in the sense that the system allows the service provider to use the key in an automated fashion without active participation by the end customer. That is, in our basic system, we will allow for asset owners to designate the service provider as a key fiduciary.

### Optionality for Shared Control of Key Use
We plan to support for both individual key use control by the asset owner and _shared key use control_ between the asset owner and the service provider. We define shared key control as a key that requires both the asset owner and the service provider to approve a particular use of the key in question, e.g., both stakeholders must agree that a transaction should be signed and the resulting signature is cooperatively generated. The service provider is responsible for their own risk assessment and compliance with regulatory frameworks, and should determine their desire for shared control accordingly. Any per-asset use controls MUST be realized through a combination of smart contracts and decision-making that is internal to the Service Provider. 

In the near-term, Lock Keeper will realize shared key use control via a lightweight policy control and auditing mechanism. This will be a minimal, barebones policy engine functionality realized using a combination of cryptographic techniques and secure processor technology. In this system, asset fiduciaries approve or deny key use via cryptographic signatures, and these decisions are enforced using secure processors. The actual process by which asset fiduciaries decide whether to approve a given key use is external to Lock Keeper.

