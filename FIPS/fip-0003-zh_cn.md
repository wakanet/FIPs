---
fip: "0003"
title: Filecoin Plus Principles
author: Alex Feerst (@feerst), jbenet (@jbenet), JV (@jnthnvctr), Tim Boucher (@tim-murmuration), Zargham (@mzargham), ZX (@zixuanzh)
discussions-to: https://github.com/filecoin-project/FIPs/issues/12
status: Active
type: Standards
created: 2020-10-13
---

## Simple Summary 简述
This proposal rebrands the Verified Clients program as Filecoin Plus - to represent the addition of a layer of social trust and all the use cases that will be enabled on the existing Filecoin protocol. Verified Clients are now Filecoin Plus Clients. Verifiers are now Filecoin Plus Notaries. This proposal separates the program into layers (Principles, Mechanisms, Operations, Markets) with different change frequencies and governance structures. The primary goal of this proposal is to describe different layers of institutional governance, establish the Principles of the program that all other layers must follow and demonstrate, clarify interfaces between the program and the technical layer, and introduce a work-in-progress repository where more specific Mechanisms and Operations can be iterated on a more frequent basis. Future updates to the Principles and technical implementation should only be made with another FIP.

该提案将验证客户端计划重新命名为Filecoin Plus，以表示添加了一层社会信任和现有Filecoin协议将启用的所有用例。已验证的客户端现在是Filecoin Plus客户端。验证者现在是Filecoin Plus公证人。该提案将该计划划分为具有不同变更频率和治理结构的层次（原则、机制、运营、市场）。本提案的主要目标是描述不同的机构治理层，确定所有其他层必须遵循和演示的项目原则，澄清项目和技术层之间的接口，并引入一个在建工程存储库，其中可以更频繁地迭代更具体的机制和操作。原则和技术实施的未来更新应仅与另一个FIP一起进行。

![governance-layers](https://user-images.githubusercontent.com/16908497/95934306-2c096200-0d96-11eb-821d-dcc7186819c3.jpg)

## Abstract
Filecoin Plus is a program that aims to maximize the amount of useful storage that Filecoin can and will support.  As a renaming and clarification of the Verified Clients program, Filecoin Plus should allow the network to trust some clients to bring honest demand to the network. The program consists of four components: Principles, Mechanisms, Operations, and Markets.  The principles are explicitly described in the document.  For the remaining components, their role and motivation are described, as well as the mechanisms by which the Filecoin community can update these components.

## Change Motivation
Filecoin Plus’s mission is to build a decentralized network of useful storage.

### A Network of Useful Storage
Filecoin Plus is a pragmatic solution to the technically challenging problem of verifying that a particular set of data is useful in a permissionless, incentive-compatible, and pseudonymous network. By adding a layer of social trust and providing leverage to storage clients, Filecoin Plus makes the network more decentralized and accelerates the proliferation of high-quality services on the network. Offering additional reward incentives for storing a Filecoin Plus Client’s deals also ensures reward subsidies go where they are needed most – encouraging legitimate use of the network.

### A Spectrum of Decentralization
It is common to see sweeping statements characterizing a particular blockchain network as decentralized. However, there are many dimensions to decentralization for any network, such as token production, geographic distribution, ownership, governance, etc. In addition to decentralizing in these ways, Filecoin will also require distribution of power over a range of participants, including miners (present and future, large and small), storage clients, application developers, researchers, and token holders to make the economy successful. In terms of storage content, diversity in storage variety and level of service on Filecoin will also make the economy more attractive. Filecoin Plus does not mean to specify how the economy will evolve but rather set up the right structure to allow economic diversity and robustness to emerge.

## Specification
The specification for the program describes the Principles layer, as well as an overview of the full program’s mechanisms and operations.
### Principles
We introduce the guiding principles of Filecoin Plus below. These fundamental goals and ideals serve to focus the Filecoin Plus program.

#### Decentralization and Diversity
As mentioned above, Filecoin Plus should strive to promote greater decentralization of control and points of failure (e.g. token production, ownership, governance, etc) and diversity of characteristics (e.g. product offering, geographic distribution, storage content, etc). This is in the interest of all participants of the Filecoin Economy in the long run and consistent with the fundamental principles of the Filecoin network.

#### Transparency and Accountability
Observing Filecoin Plus participant (Root Key Holder, Notaries, Clients) behavior should be simple and transparent. Any decisions regarding Filecoin Plus – from approval of Root Key Holders and Notaries, to DataCap allocation decisions, to governance changes – should be easily auditable by someone with no special privileges within the network, and take place within the framework set forth by community-driven governance. All actors are responsible for abiding by the governance framework and subject to repercussions if found to be in violation of the governing principles. 

#### Community Governance
The Filecoin community should collectively decide on the Principles, Mechanisms, Operations, and Markets that will govern the election, rotation, and actions of the Root Key Holders, Notaries, and Filecoin Plus Clients participating in this process. 

#### Low-Cost Dispute Resolution
Disputes will inevitably arise over matters such as the selection of a particular Notary, or the validity of a verification decision. It will be necessary to have clear criteria and procedures for resolving disputes where different parties disagree as to whether a particular action or actions diverge from the Principles. The criteria will be published in an easily discoverable place, as should basic procedural rules governing the adjudication of disputes. Dispute resolution should be relatively cheap, quick, and result in a clear outcome, while providing sufficient due process to support the fairness of the outcome.

#### Limited Trust Earned over Time
Parties in Filecoin Plus (Root Key Holders, Notaries, and Clients) all start with a limited amount of trust and power. Additional trust and power need to be earned over time through good-faith execution of their responsibilities and transparency of their actions. Participants found to be violating the governing principles or underperforming relative to expectations are subject to a reduction in trust and power, rotation, or retirement.

#### Terms of Service
Filecoin Plus Clients agree to do so in compliance with the terms of service of the storage provider. Procedures and agreements should be in place to support compliance. 

#### A Useful Storage Network
The ultimate goal of Filecoin Plus is to accelerate the timeframe over which storage goods and services on Filecoin become more useful and attractive. The program plays an essential role in shaping the product offering on the network in collaboration with storage miners.

### Mechanism Overview
Mechanisms are specific instantiations and representations of the Principles. While Principles are not expected to change frequently and can only be modified through an FIP, mechanisms and operational processes can evolve on a more frequent basis. This FIP describes the high-level role and responsibilities and the interface between the operations of the physical world and on-chain protocol. The decision-making and further refinement of the Mechanism, Operations, and Markets layers are delegated to community governance which will be defined and managed in this [repository](https://github.com/filecoin-project/Notary-governance).

#### Roles and Responsibilities

##### Community Governance
There are five main stakeholder roles in the Filecoin ecosystem: clients, miners, developers, token holders, and ecosystem partners. While an entity can play multiple stakeholder roles at the same time, the voice and opinions of each stakeholder group should be heard and weighed in making community decisions.

The scope of community governance involves shaping the rubrics, rules and processes, rather than specific cases. This includes defining mechanisms that reflect the goals and visions of the Principles; recommending standards and processes for selecting and evaluating Root Key Holders, Notaries, and Filecoin Plus Clients; and dispute resolution framework.

##### Root Key Holders
Root Key Holders are signers to a Multisig on chain with the power to grant and remove Notaries. For any action to take place, a majority of the selected Root Key Holders must sign in order for the Multisig to execute. The responsibilities of the Root Key Holder are to: 
Exercise decisions made by the community governance on-chain.
Take action on finalized decisions in a timely manner.

The role of the Root Key Holder is not to make subjective decisions - rather to act as an executor for decisions made by the community on-chain. The action space and power of Root Key Holders are also limited by design. Any deviation from this mandate will make the offending Root Key Holders immediately subject to removal through a FIP.

##### Notaries
Notaries are selected to act as fiduciaries for the Filecoin network. Notaries are entrusted with DataCap to be spent in a manner that aligns with the Principles. DataCap, when allocated to a client, can be spent by the client to store with Miners with these deals carrying a higher deal quality multiplier. The responsibilities of the Notaries are as follows: 
Allocate DataCap to clients in order to subsidize reliable and useful storage on the network.
Verify that clients receive a DataCap commensurate with the level of trust that is warranted based on information provided. 
Ensure that in the allocation of the DataCap no party is given excessive trust in any form that might jeopardize the network.
Follow operational guidelines, keep record of decision flow, and respond to any requests for audits of their allocation decisions.

Specific processes and criteria for becoming a Notary and operational guidelines are described in the [repository](https://github.com/filecoin-project/notary-governance). Notaries are given autonomy in their decision making process, but may be asked to substantiate any previous decisions before receiving future DataCap allocations to distribute. 

##### Clients
Clients are active participants of the network with DataCap allocation for their use cases. Clients can use their DataCap to incentivize miners to provide additional features and levels of services that meet their specific requirements. In doing so, storage related goods and services on Filecoin are made more valuable and competitive over time. Clients are vetted by Notaries to ensure the client receives DataCap commensurate with their reputation and needs, and that the Client responsibly allocates that DataCap.
Obtain verification and DataCap allocation from a Notary.
Deploy DataCap responsibly in accordance with the Principles.
Follow operational guidelines, keep record of decision flow, and respond to any requests for audits of their allocation decisions.

Specific details on the suggested framework for responsible DataCap allocation are described in the [repository](https://github.com/filecoin-project/notary-governance). It is expected that clients who intend to receive greater amounts of DataCap may be asked to provide evidence for responsible spending of their previous allocation before receiving more.

#### Technical Interface
The Filecoin Plus Mechanism interfaces with on-chain protocol through the [Verified Registry Actor](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/verifreg/verified_registry_actor.go) where specific actions are defined. Root Key Holders can add Notaries with a DataCap allocation or remove them based on decision outcomes in the repository. Notaries can add and allocate a portion of their DataCap to clients. 	

When a DataCap is allocated to a client, it is a one-time credit from the Notary coming from the Notary’s allocation. The DataCap amount is at the discretion of the Notary and must be judiciously allocated. Once DataCap has been transferred from a Notary to a Filecoin Plus Client, there is no way to remove the DataCap. Hence, a general principle is that trust is limited initially and gradually earned over time. 

When a client makes a deal, they have the option of passing a flag that identifies their deal as coming from a Filecoin Plus Client.  This deal carries a 10x deal quality multiplier and hence greater quality adjusted power for the miner that chooses to accept. The Filecoin Plus Client’s DataCap diminishes as deals are made in this way. Clients can receive more DataCap by re-applying to a Notary when they need an additional allocation. 

Clients can discover available Notaries, as well as their areas of expertise, regions of operations, and verification protocols, by viewing the [repository](https://github.com/filecoin-project/notary-governance) where the Notaries applications are approved or by visiting external websites. 


#### Diagram of Interactions

![interaction-diagram](https://user-images.githubusercontent.com/16908497/95934470-80144680-0d96-11eb-87ec-c792e56e87d9.jpg)

The above diagram (non-exhaustively) details the interactions between the various actors in Filecoin Plus. Specifically, it illustrates action space available to different roles and decision making processes, as well as which aspects will ultimately make it on chain. Below we describe a sample end to end flow going from becoming a Notary to allocating DataCap to a client, 

A potential Notary interested in helping to advance the utility and adoption of Filecoin may apply to receive a DataCap allocation that they might further redistribute to high quality clients. To do so, the potential Notary must file an issue following the template application to answer basic questions about their qualifications to be a Notary, their intended markets of operation, contact information, their desired DataCap, and proposed allocation plan. 

The potential Notary’s application is scored by editors of the repository based on how well it meets the criteria specified [here](https://github.com/filecoin-project/notary-governance/tree/main/notaries/templates#overview). Once approved, a Notary will be approved for a DataCap allocation - at which point a Root Key Holder will be notified to deploy the allocation on chain. Evolution of the rubric and guideline is done through Pull Requests in the repository.

Once the Notary has a DataCap allocation, they will respond to inbound requests from clients in their preferred method (publicly via Github, privately over email, etc) - vetting clients to ensure the appropriate amount of DataCap is allocated commensurate with the information provided by the client, in accordance with the approved self-defined allocation plan, and generally in a manner in accordance with the Filecoin Plus Principles. Exemplary Notaries will keep diligent records of their allocation decisions and rationale for those decisions. 

Once a client has received an allocation, they have the flexibility to spend that DataCap however they see fit to best advance their use case within the Terms of Service of the hosting node. Miners who choose to receive “verified” deals from clients will receive a 10x deal quality multiplier. When a client is in need of further DataCap, they can re-apply to a Notary for an additional allocation. Note that a Notary may request information from the client about how a previous allocation was made prior to granting a new DataCap.

### Operations Overview
Operational processes and guidelines are iterated, described, and executed in the repository. All participants follow the guideline and template of the repository. Applications to become Notaries take place in the repository and editors of the repository will score each application based on the agreed upon rubrics in a public and transparent way. Root Key Holders then execute the decision made by the community. All governance actions with regard to the operations happen within the repository. Community can file Pull Requests and participate in regular meetings to discuss the rubrics, templates, and guidelines in the repository to reflect and serve the values and goals outlined by the Principles. Changes to the Principles, however, require participants to follow the FIP process. Disputes and follow ups also happen in the repository. Given that DataCap is a one-time allocation, unresolved disputes will impact future allocation of participants - just as successful allocation will improve eligibility for future allocations. 

- [Notary Application Rubric](https://github.com/filecoin-project/notary-governance/tree/main/notaries/templates#notary-rubric)
- [Notary Allocation Plan Rubric](https://github.com/filecoin-project/notary-governance/tree/main/notaries/templates#notary-allocation-plan-rubric)
- [Notary Application Template](https://github.com/filecoin-project/notary-governance/blob/main/notaries/templates/sample-notary-application.md#notary-application-template)


## Design Rationale
The Filecoin Plus program is separated into different institutional governance layers to drive consensus and allow for emergence as the program iterates. The Principles are meant to be high-level but easily agreed upon by all participants. Both the Principles and the Technical Interface are not meant to change often and doing so will require an FIP. Mechanisms and Operations, however, can iterate more frequently in the repository as the community collectively figures out ways to best demonstrate and serve the goals and values of the Principles. Markets, on the other hand, change much more frequently with diverse needs and offerings.

Detailed rubrics and recommended guidelines are widely used in Mechanisms and Operations to encourage diversity and emergence in a principled and objective way. The majority of the governance activities should happen on these metrics, rubrics, and guidelines. 

Lastly, as the first technical iteration of the program, DataCap is a one-time allocation, rather than a recurring allowance. This creates some inconvenience but also allows the program to minimize abuse as trust can be limited to a smaller amount and subsequently increase based on track record.

## Backwards Compatibility
Technical interface has been implemented and is backward compatible by default.

## Test Cases
N/A

## Implementation
Technical aspects have already been implemented and further iterations on mechanisms and operations can take place in the Filecoin Plus [repository](https://github.com/filecoin-project/notary-governance).

## Security Considerations
There is no change to the protocol on a technical level and pledge is proportional to quality-adjusted power. Thus, security properties remain unchanged.

## Incentive Considerations
There are many incentive considerations and this is where humans are placed in the protocol of a technical network. This is essential for a goal-driven network of utility. On the macro level, Filecoin Plus incentivizes the network as a whole to acquire and subsidize clients with real business use cases and needs. Sealing as fast as one can becomes less important relative to acquiring real storage use cases. Clients, on the other hand, can demand services and applications that are not yet present on the network or provided by storage miners, hence shaping the products and services on Filecoin.

Given that there might be cases of abuse, transparency and accountability with limited trust will go a long way. More details around how this works in practice can iterate, adapt, and evolve in the repository as long as it reflects the goals and values of the Filecoin Plus Principles.

## Product Considerations
The ultimate goal of Filecoin Plus is to accelerate the proliferation of products and compliant use cases on Filecoin with both geographic and use case diversity. Filecoin Plus provides the framework and incentive for business development to happen on Filecoin. After all, adoption is more than just technology. Client needs and legitimate use cases will shape the goods and services produced on Filecoin.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
