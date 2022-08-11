# Design Philosophy and Threat Model
[Philosophy](#philosophy)<br>
[Threat Model](#threat-model-and-mitigations)<br>

## Philosophy
An informal and non-exhaustive list:

- [Zero Trust](https://csrc.nist.gov/publications/detail/sp/800-207/final).
    - defense in depth 
    - graceful failures 
    - distribution of trust, no single point of failure 
    - least privilege 
    - minimize attack surface
- Acknowledgment of human fallibility, importance of usability
- Trust-but-verify, open source, abort & identify malicious behavior
- Consideration of challenges to security posed by integration with the service provider’s asset management product: 
    - Our software should be conducive to integration in a way that protects against threats not in our immediate scope; 
    - The importance of providing clear, actionable guidance for organizational partners
- Extensibility, modularity 
    - Including allowing for transition to more robust threat models, e.g., fault-tolerant Byzantine servers
    - Generalizability
    - Keep components modular so that we can serve different types of community members and service providers with different constraints/end goals. (e.g., Do not preclude asset owner from running their own server; allow asset owners to choose from a menu of custodial options; allow service providers to choose from a menu of deployment options)
- Deployment flexibility:
    - Support on-premises HSMs and in-cloud servers
    - Allow use of external cloud infrastructure while minimizing trust required by using secure processors and/or secure hardware.
    - Allow distribution of trust by using multiple geographic/legal jurisdictions and/or external cloud/service providers

## Threat Model and Mitigations

The system should be secure against both external and internal actors, as detailed below. We ensure compatitibiliy of our technology with mitigations against the following types of adversaries and attacks. Many of the appropriate mitigations are outside the implementation scope of our core technology, but we advise the Service Provider to consider the following types of attacks and mitigation measures when deploying a digital asset manager built with Lock Keeper.

A core principle in the development of Lock Keeper is to provide guidance and make decisions that enable the consuming application to create a secure and usable product.

[External Actors](#external-actors)<br>
[Internal Actors](#internal-actors) <br>
[Software Dependencies](#software-dependencies)<br>
[Additional Threat Considerations](#additional-threat-considerations)

### External actors

#### Passive and active network-level adversaries
We advise consideration of the following:

1. Denial of service (DoS) attacks, including both temporary and permanent (e.g., volumetric, loss of server): For deployment,  standard DoS mitigations at both the cryptographic and the networking level are advisable, including mechanisms that account for server compromise and failure, namely
    1. Server over-provisioning; 
    1. Server replication; 
    1. Automated volumetric DoS protection; and
    1. Cold backups, where/if appropriate.
1. Attacks on the security of communication channels, including eavesdropping and monster-in-the-middle: We ensure our secure channel implementation provides confidentiality, integrity, and mutual authentication. Other desirable, relatively standard properties include forward secrecy and backward secrecy. The Service Provider should similarly ensure security of communication channels.
1. Malicious external compromise of server infrastructure: In line with the principle of graceful system failures, we consider possible compromise of the server infrastructure, including of routers and load balancers. Where possible, the system should be robust up to some designated threshold of compromise. Mitigations include:
    1. Infrastructure providers and compromised servers should not be able to mount effective brute force attacks on (partial) key material. Mitigations include:
        1. The use of enclaves on servers: sensitive data and operations should take place on server enclaves or, optionally, on the asset owner’s device.
        1. The use of threshold/MPC protocols to cryptographically enforce limits on the probability of a successful key recovery/key misuse attack by a subset of infrastructure providers/servers.
    1. Periodic faults/failures of infrastructure should not render the system unusable. In particular, single points of failure should be avoided. This can be achieve by a system that allows for distribution of trust via flexible deployment across:
        1. On-premises HSMs, if appropriate and desired.
        1. Multiple providers of external cloud infrastructure, e.g., Microsoft Azure, AWS, GCP.
        1. Use of multiple geographic/legal jurisdictions for infrastructure deployment.


#### Infrastructure providers

If the service provider enters into a partnership agreement with an infrastructure provider, e.g., AWS, GCP, Microsoft Azure, then we assume these entities are honest-but-curious.

#### Other humans and malware

Malware and device compromise is a consideration of Lock Keeper, particularly with respect to confidentiality assurance:

1. Lock Keeper should provide confidentiality of the key and other sensitive user data on device and in transit:
    1. A (whole) secret should never be held by any single entity, except possibly by the asset owner after an explicit, authorized request. Certain types of keys may not have full recovery functionality enabled, e.g., if the asset is considered a shared asset with the service provider. Threshold cryptography/MPC solutions should be leveraged to this end for key generation, distribution, and for signing/decryption.
    1. The asset owner must authenticate in order to recover, use, or otherwise interact with their key. Care must be taken that the authentication mechanism is robust against impersonation attacks, e.g., by use of threshold cryptography/MPC solutions and/or a distributed PAKE.
1. The developers integrating Lock Keeper should receive integration guidance on how to avoid decisions that undermine key or user data confidentiality.

Humans with whom the digital asset owner interacts physically, e.g., family members, housemates, friends, and coffee shop dwellers, or virtually. Specifically, the software integrating Lock Keeper should include mitigations for the following types of threats:
1. Physical threats to confidentiality, e.g., for shoulder-surfing attacks, be mindful of the community member’s context when asking for the input of sensitive information. Consider additional strategies, such as provision of educational materials to individual community members, and use of concrete, actionable advice and nudges for how to handle sensitive system information.
1. Social engineering attacks: Avoid any scenario in which the asset owner is expected to send their key material to another party. Consider additional strategies, such as provision of educational materials to individual community members.

### Internal actors

Internal actors include the Lock Keeper software itself, any (developers of) organizational software that integrates Lock Keeper, server infrastructure and infrastructure providers (including cloud service providers) and sub-entities of the service provider, including individuals who work for these organizations, as well as the digital asset owner. We consider:

#### Servers

We do not fully trust servers, but we also do not expect servers to behave maliciously as part of normal system behavior. In our context, servers are generally run by sub-entities of the service provider and certain policy decisions may be made by a human operating in an official capacity; we consider the covert adversarial model for these sub-entities, including any humans-in-the-loop. That is, we consider adversaries who are willing to cheat, but not willing to be caught. 

The system should therefore be robust against compromise of a subset of the servers and be capable of detecting malicious behavior, aborting the given operation, removing/fixing/punishing the affected server, and retrying the operation. 

Mitigations overlap with mitigations against compromise of servers by external entities and also include:

1. The incorporation of internal audit and logging mechanisms, where possible, backed by cryptography.
1. Cryptographic enforcement of the policy engine (as a system extension).
1. The use of cryptographic protocols that allow for identification of server misbehavior, e.g., in the context of secret sharing, preference is given to protocols that allow for validation of provided shares and identification of any servers that provide an incorrect share.

#### Humans and other stakeholders

Humans are fallible:

1. The asset owner may lose their device(s) and may forget any supposedly human-memorable secrets; the asset owner may additionally not choose any human-memorable secrets appropriately.
1. The asset owner may accidentally misuse/misconfigure system software. This is arguably out of scope, but guidance should be provided to organizations making use of our software as to appropriate defaults.
1. The asset owner may inadvertently reveal system secrets, e.g., due to lack of an appropriate mental model of the system’s security properties or in response to a social engineering attack. This threat vector is arguably out of scope, but guidance should be provided to organizations making use of our software.
1. Stakeholders in the service provider ecosystem may accidentally misuse/misconfigure Lock Keeper:
    1. APIs should be misuse-resistant.
    1. APIs should be well-documented.
    1. Support should be provided for the initial integration and configuration efforts of Lock Keeper.
1. Stakeholders in the service provider ecosystem can attempt to steal or misuse assets:
    1. Policy engine enforcement should be robust and allow for the identification of a stakeholder that participates in an inappropriate signature creation, i.e., the system should incorporate logging and audit mechanisms. The potential use of enclaves should be taken into account for this purpose.
    1. Stakeholders should be subject to the principle of least privilege.

#### Software dependencies

Software libraries, including our own, are a potential threat vector. These libraries must be trusted to behave honestly while the system is in use, but trust should be granted sparingly and software selection should be _transparent_.

We should publish both our reasoning for specific software selection and our level of confidence in the selected components. Our dependencies should be vetted and compatible with a trust-but-verify approach, which includes:

1. A preference for software audited by reputable companies.
1. A preference for reputable, open-source software that is well-documented and contains thorough testing infrastructure, including appropriate unit and integration tests.
1. Inclusion of best practices to ensure the integrity of downloaded software and applications, including updates. Anyone running the software should be able to ascertain the veracity of the software and the integrated system should provide guidance on how to do so.

### Additional Threat Considerations

We focus on the core cryptographic key management design of a larger digital asset management system. As such, we do not provide comprehensive integration plans for security solutions that are a consideration of the deployment environment and are not part of the core cryptographic system. Service Providers SHOULD give careful treatment to deployment and operational security.

As stated above, the design of Lock Keeper is meant to be modular and compatible with other security and privacy-enhancing solutions. Adversaries and attacks that we do not consider, but can be addressed by system extensions or the Service Provider, if desired, include:
- Passive attacks by external adversaries. Lock Keeper does not include anti-censorship or privacy-preserving networking layers. That is, we do not consider traffic analysis by an external adversary as a successful attack on our system. Mitigation may be achieved through the integration of a privacy-preserving networking layer such as Tor or Nym.
- Active attacks by global or near-global external adversaries. Lock Keeper does not consider denial of service attacks by global or near-global adversaries. If censorship is a concern, we recommend integration of anti-censorship technologies such as Tor or Nym. We recommend integrating standard network analysis techniques for intrusion detection, denial-of-service resilience, and achievement of service level guarantees as part of the deployment plan.  
- Compromise of stakeholder devices. Lock Keeper does not currently consider extended mitigation efforts against compromise of devices owned and operated by system stakeholders, including asset owners and institutional entities. We do, however, include mitigation efforts at the cryptographic protocol layer in this document, namely the use of threshold cryptography to minimize the amount of confidential information available on any one device. We recommend considering a system extension and/or deployment plan that integrates:
    - Trusted execution environments or other secure hardware on asset owner and other stakeholder devices.
    - Multiple external cloud infrastructure providers in order to achieve a distribution of trust.

