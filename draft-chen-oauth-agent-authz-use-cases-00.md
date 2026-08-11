---
stand_alone: true
ipr: trust200902
cat: std
submissiontype: IETF
area: Security
wg: OAuth Working Group

docname: draft-chen-oauth-agent-authz-use-cases-01

title: Agent Authorization use cases and gap analysis
abbrev: Authorization use cases
lang: en
kw:
  - OAuth
  - Agent
  - AI
  - Artificial Intelligence
  - Authorization
  - Delegation
  - Revocation
  - Gap Analysis

author:
- name: Meiling Chen
  org: China Mobile
  city: BeiJing
  country: China
  email: chenmeiling@chinamobile.com
- name: Jia Chen
  org: China Mobile
  city: BeiJing
  country: China
  email: chenjia@chinamobile.com
- name: Jiankang Yao
  org: CNNIC
  city: BeiJing
  country: China
  email: yaojk@cnnic.cn
- name: Yuning Jiang
  email: jiangyuning2@h-partners.com
- name: Chunchi Peter Liu
  org: Huawei
  email: liuchunchi@huawei.com
    
informative:
  RFC2119:
  RFC8174:
  RFC7009:
  RFC6749:
  RFC7523:
  RFC8693:
  RFC9396:
  I-D.song-oauth-ai-agent-collaborate-authz:


--- abstract

This document provides a systematic analysis of these emerging agent-based use cases. It categorizes them into distinct scenarios, details their specific authorization requirements, and performs a comprehensive gap analysis against the existing OAuth 2.0 framework [RFC6749] and its common extensions. The analysis identifies fundamental mismatches, the goal of this document is to articulate these gaps clearly, providing a foundation for future work on new extensions within the OAuth Working Group to address the authorization needs of the next generation of ai agents.

--- middle

# Introduction

The OAuth 2.0 Authorization Framework [RFC6749] has become the de facto standard for delegated authorization on the internet. Its success is rooted in a well-defined model where a resource owner grants a third-party client limited access to their resources without sharing their credentials. This model primarily assumes a user-present, interactive flow where a static set of permissions (scopes) is approved upfront.

However, the landscape is rapidly evolving with the advent of sophisticated AI-driven "Agents". These are not simple clients but autonomous or semi-autonomous entities that perform complex, multi-step tasks on behalf of a user or another system. Their operational characteristics include:

*   **Delegated Autonomy:** Agents act with authority delegated from a principal (user or system) over extended periods.
*   **Complex Task Decomposition:** An agent might take a high-level instruction (e.g., "plan my business trip") and decompose it into numerous discrete actions involving multiple resource servers.
*   **Asynchronous & Long-Running Operations:** Tasks may run for hours, days, or indefinitely, often without direct, real-time supervision from the principal.
*   **Dynamic & Emergent Needs:** The exact permissions required may not be known at the start of a task but emerge as the agent plans and executes its steps.
*   **Composition & Chaining:** Agents may delegate sub-tasks to other agents, forming a chain of authority.
*   **Cross-Domain**: Chained or composited task orchestration often require calling of resources in different administrative domains.

This document does not propose new solutions or protocols. Instead, its purpose is to:
*   Define the key actors and concepts in agent-based authorization scenarios.
*   Describe a set of core use cases that exemplify the new challenges.
*   Conduct a gap analysis for each use case, identifying where the current OAuth 2.0 framework and its extensions fall short.
*   Summarize the key security considerations.

By clearly articulating these gaps, this document aims to provide a shared understanding of the problem space and stimulate focused work on developing interoperable solutions for the next generation of delegated authorization.

# Terminology

In addition to the terms defined in [RFC6749], this document uses the following terms:

**User (Resource Owner):**
: An entity capable of granting access to a protected resource. In the context of this document, the user is the person who delegates authority to an agent. Conforms to the definition of "Resource Owner" in [RFC6749].

**Authorization Server (AS):**
: The server that authenticates the User and issues access tokens to the Agent after obtaining the User's authorization. Conforms to the definition in [RFC6749].

**Resource Server (RS):**
: The server hosting the protected resources, capable of accepting and responding to protected resource requests using access tokens. Conforms to the definition in [RFC6749].

**Data Subject:**
: A natural person whom accessed data describes, and who is neither the Resource Owner nor a party to the authorization exchange.

**Delegation Chain:**
: A sequence of delegation events, where one actor grants a subset of its authority to another. For example: User -> Agent A -> Agent B.

**Intent:**
: A high-level description of a goal or task that the user wants the agent to accomplish (e.g., "book me a flight to Hawaii for next week").

**Tool:**
: A specific API or function that an agent can call to perform an action (e.g., a `search_flights` API, a `send_email` function).

**Grant Layer Authority:**
：The set of permissions established at the time of authorization, typically represented by scopes in an access token. It defines what an agent is allowed to do in principle (e.g., "The agent has the authority to book parks"). This authority is the primary focus of the core OAuth 2.0 framework and its extensions like RAR.

**Execution Layer Evidence:**
:A verifiable, non-repudiable record, generated at the moment a specific, critical action is taken. It serves as cryptographic proof of a principal's explicit consent to the concrete parameters of that single action (e.g., "The user explicitly approved booking 'Sunnyvale Park' for  $25 on this specific date"). This evidence proves *what* was done and under whose direct approval at runtime, not just what was *allowed* to be done.

#  Core Use Cases and Gap Analysis

This section explores different categories of use cases, providing a concrete example for each, and analyzes the gaps in the existing OAuth 2.x framework. For each use case, we examine what can be achieved with existing tools and, more importantly, what is missing.

## Category 1: Personal & Consumer Scenarios

### Use Case 1: Personal Digital Assistant

*   **Scenario Description:** A user gives a high-level, natural language command to their AI assistant. The assistant must decompose this command into multiple steps and interact with various services to fulfill the request.

*   **Example:** On Monday morning, Alice tells her AI assistant, "Help me plan a picnic for this Saturday." The assistant begins its work autonomously:
    1.  It first checks Alice's calendar for availability (requires `calendar.read`).
    2.  It then checks the weather forecast for Saturday (requires `weather.read`).
    3.  Seeing the weather is good, it spends some time researching nearby parks based on Alice's past preferences (requires `maps.search`and profile access).
    4.  On Tuesday afternoon, after identifying the perfect spot, the assistant determines it needs final authorization to book the picnic spot (requires parks.book) and add the event to Alice's calendar (requires calendar.write).
*   The assistant now presents an authorization request to Alice.

*   **Challenge: Context Collapse and the Erosion of Informed Consent:**
*   This scenario highlights a fundamental challenge to the traditional OAuth 2.0 model, which stems from the temporal decoupling of task initiation and authorization.
    *   **Context Collapse:** The authorization request Alice receives on Tuesday afternoon is a simple prompt listing scopes like parks.book and calendar.write. The original context—the "picnic planning" instruction from the previous day—is lost. To Alice, the request is at best confusing ("Why does this app want to book a park now?") and at worst, indistinguishable from a sophisticated phishing attack. The traditional OAuth consent screen, designed for immediate, in-context requests, fails to provide the necessary assurance for the user to make a safe and informed decision.
    *   **Inadequacy of Binary Choices:** Alice's only options are "Grant" or "Deny." However, her real question might be, "Which park did you choose?" or "Is there a fee?" The current framework provides no mechanism for this crucial dialogue. Denying the request causes the agent's task to fail, while granting it feels like signing a blank check.
    *   **Task-Level Revocation:** Alice should be able to say, "Cancel the picnic planning," and have all permissions and pending actions related to this specific task instantly revoked, without affecting other tasks the agent might be performing.

*   **Gap Analysis:**
*   This use case does not imply that OAuth itself should be responsible for parsing user intent (the agent's job) or orchestrating the task (the agent framework's job). Instead, it reveals a critical gap in the authorization experience when interacting with autonomous systems.

    *   **What Works (Partially):** The OAuth Authorization Code flow can be used to get initial permissions. Refresh Tokens can maintain the agent's session.
    *   **What's Missing (The Gap):**
        *   **No Standard for Authorization Context:** The core gap is the lack of a standardized mechanism to carry the justification for the request from the agent to the user via the Authorization Server. OAuth's model, designed for immediate user-initiated flows, implicitly relies on the user's short-term memory to provide context. This assumption breaks down in agentic workflows. There is no standard way for the agent to pass a cryptographically verifiable "context object" (e.g., "This is for the picnic you requested on Monday") that the AS can present to the user.
        *   **No Standardized Interactive Consent Flow:** There is no standard OAuth mechanism for an Authorization Server to facilitate a "clarification dialogue." The AS acts as a simple gatekeeper with a static grant/deny choice. It cannot "pause" the flow to allow the user to query the agent for more details (e.g., "Show me the park details") before consenting to the parks.book scope. This logic is currently left to complex, proprietary application-layer implementations.
        *   **Impractical Task-Level Revocation:** OAuth Token Revocation ([RFC7009]) revokes a single token. To achieve task-level revocation, the application would need to build and maintain a complex, non-standard mapping of tasks to all associated tokens. There is no standard way to issue a single command like "revoke all tokens and authority related to 'picnic-task-123'."
        *   **No Standard for Execution-Layer Evidence:** At the moment of financial commitment (the final "Book" step), a simple access token representing Grant-Layer Authority (e.g., a parks.book scope) is insufficient. The core requirement is for Execution-Layer Evidence: a non-repudiable, cryptographic proof that binds the user's explicit, real-time consent to the specific parameters of the transaction (e.g., "Book picnic spot at 'Sunnyvale Park', cost $25"). This evidence serves as proof of the human's decision at the moment of execution, proving what the user agreed to, not just that the agent possessed the authority to book something. The existing framework lacks a standard for generating or verifying such evidence.

### Use Case 2: Smart Home & Automation

*   **Scenario Description:** A central home hub agent manages various IoT devices based on pre-defined rules or real-time events.

*   **Example:** Bob sets up a "Good Morning" routine. When his alarm goes off at 7 AM on a weekday, the home hub agent is authorized to:
    1.  Slowly turn on the bedroom lights (device: `bedroom_light`, capability: `brightness`).
    2.  Set the thermostat to 70°F (device: `thermostat`, capability: `set_temperature`).
    3.  Start the coffee maker (device: `coffee_maker`, capability: `on_off`).

*   **Authorization Requirements:**
    *   **Persistent Delegation:** After a one-time setup, the agent must be able to perform these actions daily without Bob's intervention.
    *   **Fine-Grained Device & Capability Permissions:** The agent should be authorized to control the `brightness` of the `bedroom_light`, but not, for example, to unlock the `front_door`.
    *   **Conditional or Event-Driven Authorization:** The permissions should only be usable when specific conditions are met (e.g., `time=7AM` AND `day=weekday`).
    *   **Bulk Revocation:** When Bob sells his house, he needs a simple way to revoke all permissions for all devices from his home hub with a single action.

*   **Gap Analysis:**
    *   **What Works (Partially):** Refresh Tokens are well-suited for persistent delegation. Scopes can be defined with high granularity (e.g., `device:bedroom_light:brightness`).

    *   **What's Missing (The Gap):**
        *   **"Scope Explosion" and Usability:** In a home with hundreds of devices and capabilities, presenting the user with a list of thousands of scopes to approve is unmanageable. The OAuth consent screen was not designed for this scale.
        *   **No Standardized Policy Enforcement:** OAuth grants *what* an agent can do, but not *when* or *under what conditions*. The logic to enforce `time=7AM` is outside the protocol and must be custom-built into the agent or the device's resource server, leading to inconsistent and non-interoperable implementations.
        *   **No Standardized Bulk Revocation:** [RFC7009] is for revoking one token at a time. There is no standard API to "revoke all tokens for user Bob" or "revoke all tokens issued to the home hub client." This is a critical administrative and security gap, forcing reliance on proprietary AS-specific APIs.

### Use Case 3: Agent as User's Full Proxy to Access Third-Party Tools

*   **Scenario Description:** A user authorizes an agent to use a third-party service on their behalf. The service, however, was designed for human interaction and has no built-in awareness of automated agents, leading to potential misuse or policy violations.

*   **Example1:**  Charlie authorizes his personal research assistant agent to use his subscription to an academic paper database. The agent, acting as Charlie, begins downloading hundreds of papers for a literature review. The database service detects this high-frequency activity, flags it as a potential DDoS attack or data scraping, and temporarily locks Charlie's account, disrupting both the agent's task and Charlie's own access.
*   **Example2:** The database service later adds its own built-in AI features: an AI summarizer that processes users' saved papers. Charlie now faces three distinct classes of caller against the same API surface: his own interactive sessions, the service's built-in AI, and his external research agent. He may be comfortable with the built-in summarizer processing his library but not an external agent, or the reverse. His consent decision for one class says nothing about the others.
    

*   **Authorization Requirements:**
    *   **Agent-User Differentiation:** : The third-party service must be able to reliably distinguish between requests coming directly from Charlie and requests made by his agent.
    *   **Agent-Specific Policies:** The service needs a way to apply different policies (e.g., stricter rate limits, restricted API access) to the agent without impacting the human user's normal access rights.
    *   **Delegated Authority with Constraints:** The authorization given to the agent should be a constrained subset of the user's full permissions (e.g., "can search and download, but no more than 100 papers per hour").
    *   **Caller-Class Differentiation:** Beyond distinguishing the agent from its user, the service must be able to distinguish among classes of caller (at minimum the interactive human session, the application's own platform-native AI, and external user-delegated agents) in a way the caller cannot self-select.
    *   **Independent Per-Class Consent:** The user must be able to grant or deny consent for each caller class independently, with no inheritance between classes: consent for one class must not imply consent for any other.
    *   **An Attachment Point for Internal-AI Consent:** The application's own AI features typically never traverse the authorization layer at all, so there is currently nothing to attach a consent decision to for that class.

*   **Gap Analysis:**
    *   **What (partially) works:**  OAuth allows a user to delegate access to a client (the agent). The scope parameter can limit which APIs the agent can call.
    
        *   **What Works (Partially):** OAuth allows a user to delegate access to a client (the agent). The `scope` parameter can limit which APIs the agent can call.
    *   **What's Missing (The Gap):**
        *   **No Standard Agent Identifier:** There is no standard OAuth claim or parameter that explicitly signals "this request is from an automated agent." Resource servers are left to guess based on non-standard signals like User-Agent strings or unusual traffic patterns.
        *   **Inability to Express Constraints:** The standard `scope` mechanism is binary (permission is granted or not). It cannot express or enforce nuanced constraints like rate limits, data volume caps, or time-of-day restrictions as part of the authorization grant itself.
        *   **Confused Deputy Risk:** Without clear differentiation, the service provider cannot tell if high-volume activity is malicious (account takeover) or a legitimate but overly aggressive agent. Their only recourse is often to block the user's account entirely.
        *   **No Standard Caller-Class Representation:** Beyond the human-versus-agent distinction, there is no standard representation of a caller class, and no standard way to express consent that is evaluated per class without inheritance between classes.
        *   **Internal AI Is Invisible to Authorization:** Because the platform's own AI typically operates inside the application boundary, no standard consent mechanism applies to it at all, leaving users no way to express "external agents I delegate may process my data, but the platform's AI may not," or the reverse.

### Use Case 4: Agent as User's Proxy to Access Operating System Resources

*   **Scenario Description:** An agent running on a user's device (e.g., a laptop or smartphone) needs to perform tasks that require access to local resources like files, applications, or system settings.
 
*   **Example:** Diana asks her desktop agent, *"Clean up my downloads folder and free up disk space."* To do this, the agent needs permission to read and delete files within `/home/diana/downloads`. However, to be "helpful," the agent might also try to clear system-wide caches, which requires elevated, system-level permissions.
 
*   **Authorization Requirements:**
    *   **Fine-Grained Resource Access:** The agent should be granted access only to the specific files or settings needed for a task (e.g., a single folder), not the user's entire home directory or full system access.
    *   **Task-Scoped Permissions:** Permissions should be granted for the duration of a specific task ("clean downloads folder") and automatically revoked upon completion.
    *   **Secure Privilege Escalation:** If a task requires elevated permissions, there must be a secure, user-approved mechanism for the agent to request them just-in-time, rather than running with high privileges constantly.
 
*   **Gap Analysis:**
    *   **What Works (Partially):** Modern operating systems have their own permission models (e.g., app sandboxing, runtime permission prompts).
    *   **What's Missing (The Gap):**
        *   **Architectural Mismatch:** OAuth is a framework for network-based delegated authorization. It is not designed to manage local, process-level OS permissions. There is a fundamental mismatch between the two models.
        *   **Lack of a Standard Bridge:** There is no standard protocol to connect an agent's high-level intent (often determined in the cloud) to a secure, fine-grained grant of local OS permissions on the device.
        *   **Over-Privileging by Default:** Due to the lack of a standard bridge, developers often resort to a dangerous workaround: the agent's local component is installed with broad, persistent permissions (e.g., running with the user's full rights or even as a system service). This completely bypasses the principle of least privilege and creates a significant security risk.

### Use Case 5: First Connection to a Service with No Authorization Front Channel

*   **Scenario Description:** Scenario Description: A user wants their agent to access a service that has no authorization server relationship, its own or a delegated one, and exposes no browser-facing authorization endpoint. The agent runs in a headless or remote environment with no co-located browser. Neither side can present the front channel that existing flows assume.

*   **Example:** Dana subscribes to a note-taking service that exposes a plain REST API with no authorization server and no browser-facing authorization endpoint, as is common for services and MCP servers that predate or do not implement an authorization framework. She wants her personal assistant agent, which runs on a cloud host, to file notes for her. To make the first connection, Dana needs to:
1. Grant the agent scoped access (notes.write, but not account.manage).
2. Set a lifetime for that access and retain the ability to revoke it.
3. Get the resulting credential to the agent, which has no browser and cannot complete a redirect-based flow.

Today her only practical option is to copy a static API key into the agent's environment variables: long-lived, all-or-nothing, and invisible to her after setup.

*   **Authorization Requirements:**
    *   **Front-Channel-Free Bootstrapping:** The first-connection grant must be possible when neither the service nor the agent can present a browser-based interaction channel.
    *   **User-Authored Grant:** The user, not the agent, needs a way to initiate the grant and set its scopes and lifetime, since there is no authorization request for the agent to redirect her to.
    *   **Scoped, Revocable Credential:** The resulting credential must carry a constrained scope and lifetime and remain revocable by the user, in contrast to the static API key it replaces.

*   **Gap Analysis:**
    *   **What Works (Partially):** The Device Authorization Grant [RFC8628] removes the co-located browser requirement on the client side. Static API keys work operationally, which is why they are the de facto practice.
    *   **What's Missing (The Gap):**
    *   **A verification page is still assumed:** [RFC8628] presumes an authorization server with a browser-facing verification page, and the client initiates the request and proposes the scopes. It does not cover the case where no such page exists because the service has no front channel at all.
    *   **No standard first-connection grant:** There is no standardized way for the user to grant scoped, revocable access and for the agent to obtain the resulting credential when no front channel exists on either side. This precedes the flows in the existing use cases; Use Case 1's gap analysis, for example, notes that the Authorization Code flow can obtain the initial permissions, which assumes that front channel is available.
    *   **The workaround is the anti-pattern:** Environment-variable API keys are long-lived, over-broad, and invisible to the user after setup; these are the same over-privileging properties Section 5 warns against.

## Category 2: Enterprise & Business Process Scenarios

### Use Case 6: Complex Business Process Automation

*   **Scenario Description:** A task is passed through a chain of specialized agents, each performing one step of a larger business process.

*   **Example:** An automated insurance claim process:
    1.  **Agent A (Intake):** Receives a claim from a customer and is authorized to read the customer's policy. It passes the claim to the next agent.
    2.  **Agent B (Verification):** Receives the claim from Agent A. It is authorized to access external databases to verify the details of the incident. It then passes the verified claim to Agent C.
    3.  **Agent C (Adjudication):** Receives the verified claim from Agent B. It is authorized to run a risk assessment and approve or deny the claim. If approved, it delegates the final action to Agent D.
    4.  **Agent D (Payment):** Receives the approved claim from Agent C. It is authorized *only* to execute a payment to the customer's bank account for the approved amount.

*   **Authorization Requirements:**
    *   **Verifiable Delegation Chain:** The final agent (Payment Agent D) and the resource server (the bank's API) must be able to cryptographically verify the entire authorization path: `Customer -> Agent A -> Agent B -> Agent C -> Agent D`.
    *   **Principle of Least Privilege at Each Step:** Agent B should have no payment authority, and Agent D should have no access to the customer's policy details. The permissions must be strictly constrained at each step in the chain.
    *   **Auditable Context:** The entire process must be tied to a single, auditable `claim_id` that is securely passed along the chain.
    *   **Data Subject:** Where an agent accesses data describing a Data Subject who is not the Resource Owner, the authorization evidence MUST make the third-party access legible as such, and the catalogue MUST name whose authority permits it.
    *   **Execution-Layer Evidence for Final Action:** The final payment action by Agent D must not only be authorized by the delegation chain (Grant-Layer Authority) but must also generate verifiable Execution-Layer Evidence. This evidence must bind the specific payment details (amount, recipient) to the full, verifiable delegation chain and the original claim_id, creating an undeniable record that the specific transaction was legitimate and explicitly sanctioned.


*   **Gap Analysis:**
    *   **What Works (Partially):** OAuth 2.0 Token Exchange [RFC8693] introduces the `act` (actor) claim, which provides a primitive to show that one agent is acting on behalf of another. This is a foundational building block.
    *   **What's Missing (The Gap):**
        *   **No Native Support for Chains:** A standard OAuth token represents a simple, two-party delegation (User -> Agent A). It cannot natively represent a multi-step chain (`User -> A -> B -> C`). While the `act` claim from [RFC8693] helps, there is no standard for how to nest these claims to create a verifiable, multi-hop chain. This is a major architectural gap.
        *   **Lack of Standardized Context Passing:** There is no standard field in an OAuth token to carry the `claim_id` securely through the process. Developers resort to custom claims in a JWT, which harms interoperability.

### Use Case 7: Coordinated Task Group

*   **Scenario Description:** A coordinating agent decomposes a user's request into subtasks and assembles a group of specialized sub-agents to execute them. Unlike the delegation chain in Use Case 5, where each agent derives its authority from the previous agent hop by hop, here every member's authority stems from a single grant obtained centrally by the coordinator [I-D.song-oauth-ai-agent-collaborate-authz]. The execution order of subtasks (sequential, parallel, or mixed) is independent of this authorization structure.

*   **Example1:** A user asks a health assistant for real-time health advice. A leading agent resolves the request into three subtasks: collecting the user's health data, predicting the user's health status, and generating advice. It selects a sub-agent for each subtask, each needing access to different resource servers (a wearable data API, a prediction service, a guideline repository). Sub-agents may all be selected upfront, or incrementally during execution as intermediate results reveal which further subtasks are needed.
*   **Example2:** Cross-bank Coupon Agent of a leading payment company can coordinate multiple commercial banks' agents to compare the best promotional discount from all of the user's bank accounts for a single payment. It obtains a single batch authorization for query and payment permissions of all the user's bank accounts, then decomposes these permissions, restricting each sub-agent to query only its respective bank's promotional discounts.


*   **Authorization Requirements:**
    *   **One Grant, Many Members:** The coordinator must be able to obtain authorization once, on behalf of the whole group, rather than each member running its own flow against the authorization server and the user.
    *   **Member-Level Least Privilege:** Each member must be confined to its own subject-audience-scope binding within the group's overall authority.
    *   **Late Binding of Members:** Members selected mid-execution must be able to receive authority under the existing grant without a full re-authorization round trip.
    *   **Group Lifecycle:** When the task completes or the coordinator is compromised, the entire group's authority must be terminable as one unit.

*   **Gap Analysis:**
    *   **What Works (Partially):** Token Exchange [RFC8693] defines the privilege delegation relationship via the `may_act` claim, but it only defines one actor and cannot link that actor to a specific subset of permissions. Rich Authorization Requests [RFC9396] can express fine-grained permissions per request，but cannot bind them to different actors. 
    *   **What's Missing (The Gap):**
        *   **Per-Member Authorization Does Not Scale:** N members individually authenticating and obtaining tokens for what is logically one user grant means N authorization server interactions and potentially N consent prompts for a single request. There is no standard way for one party to request authorization on behalf of an enumerated set of clients.
        *   **No Group Construct in the Token Model:** Tokens describe one client acting for one subject. There is no standard representation of group membership, nor of per-member subject-audience-scope bindings under a common grant, that a resource server could verify.
        *   **No Late Binding of Members:** Members that cannot be enumerated at grant time have no standard way to be admitted to the group's authority, for example via a verifiable task assignment bound to the original grant.
        *   **No Group Lifecycle Management:** There is no standard way to terminate a group's authority atomically, nor an enforced rule that a member's effective permissions remain a subset of the group's.

### Use Case 8: Automated DNSSEC DS Record Maintenance Agent

*   **Scenario Description:** This case follows the multi-agent RRR (Registrant-Registrar-Registry) model defined in `draft-ietf-dnsop-ds-automation-09`. A corporate domain owner deploys chained agents to fully automate DS record updates during DNSSEC KSK rollovers, zone bootstrapping, and zone deletion, relying on CDS/CDNSKEY signals between child authoritative zones and parent registry systems.

*   **Example:** The whole workflow runs unattended under a one-time delegated permission from the domain registrant, forming a delegation chain: **Registrant --> Registrar Agent --> Registry Agent.**
    - **Zone Agent:** Publishes consistent CDS/CDNSKEY records on all name servers after key rollover.
    - **Registrar Agent:** Pulls and validates CDS data, forwards DS update requests to the registry.
    - **Registry Agent:** Applies DS records to the parent zone and sends operation notifications.

*   **Authorization Requirements:**
    - Cryptographically verifiable multi-hop delegation chain for a full audit trail.
    - Fine-grained permission limited to a fixed set of domains, only allowing DS updating without modifying other DNS records.
    - Support task-specific revocation: revoke all DS automation permissions without affecting other agent workflows.
    - Standard audit context identifier passed across all agent hops for compliance logging.

*   **Gap Analysis:**
    *   **What Works (Partially):** Client Credentials can assign identities to registry/registrar agents. RFC 8693 token exchange supports simple single-hop agent delegation.

    *   **What's Missing (The Gap)
        *   **Current DS automation does not use OAuth. OAuth only supports two-party delegation and lacks standard multi-hop RRR delegation syntax; proprietary JWT claims hurt interoperability.
        *   **OAuth has no standardized task/bulk revocation. Securing DS automation via OAuth would require repetitive single-token revocation in incidents.
        *   **No reserved OAuth JWT claim for cross-agent audit IDs, breaking consistent compliance logging for DS automation.

### Use Case 9: Managed Services

*   **Scenario Description:** The managed service model refers to an integrated end-to-end managed service solution delivered by a prime service integrator through the aggregation of infrastructure and services from multiple sub-providers. End users interact with a single, unified resource platform instead of multiple separate sub-modules. However, coordinating resources across different administrative domains introduces complex authorization challenges. 

*   **Example:**
    *   Telecom operator delivers a managed global secure SD-WAN orchestration service, which requires coordination of many localized regional telecom operators/ISPs and security service providers.
    *   Telecom operator delivers a managed smart home + security solution through a home-hub, which integrates controls over IPTV, streaming services, smart IoT (security cameras) etc.

*   **Authorization Requirements:**
    *   **Cross-domain Authorization:** The fast provisioning of an integrated service (take SD-WAN as an example) requires acquiring authorization from different service sub-providers in their respective administrative domains.
    *   **Batched Authorization:** The fast provisioning of the integrated service requires requesting a batch of authorizations from different service sub-providers.
    *   **Delegated Authorization:**
        *   Prime service integrator may need to conditionally delegate network configuration modification privileges for specific branches to the on-site O&M teams of local sub-providers. This delegation MAY be hierarchical.
        *   End-users are often times tenants that only have renting privileges instead of ownership rights. Thus technically the prime service integrator is delegating the access privileges to the end-user to access each modular sub-service. 

*   **Gap Analysis:**
    *   **What Works:**
        *   **Cross-domain Single Authorization:** The OAuth allows cross-domain authorization through {{RFC7523}} OAuth 2.0 authorization grant and {{?I-D.ietf-oauth-identity-chaining}} Identity Chaining. But it MUST require one access token at a time.
        *   **Single Domain Workflow:** The OAuth allows workflow orchestration through {{?I-D.ietf-oauth-transaction-tokens}} Transaction Token. It only allows workflow inside one trust domain.
     *   **What's Missing (The Gap):**
         *   **Batched Authorization:** How to request access permissions to a batch of resources from the user at once, while precisely delegating fine-grained privileges to the respective sub-agent responsible for executing each sub-task.
         *   **Cross-domain Workflow:** How to request access permissions across different trust domains, while ensuring secure propagation of important contextual information (claims, caller identity, rich authorization contexts...).


## Category 3: Security & Administrative Scenarios

### Use Case 10: Automated Security Incident Response

*   **Scenario Description:** A security agent detects a security threat and must take immediate, automated action to contain it.

*   **Example:** A security system detects that an employee's laptop has been compromised by malware. An automated security agent is triggered to:
    1.  Immediately revoke all active login sessions for that employee across all company applications (e.g., email, code repository, HR system).
    2.  Isolate the compromised laptop from the corporate network.
    3.  Temporarily suspend the user's account to prevent further unauthorized access.

*   **Authorization Requirements:**
    *   **Privileged, System-Level Authority:** The security agent needs broad, pre-approved authority to perform high-impact administrative actions.
    *   **Global Token Revocation API:** The agent must be able to make a single API call to the Authorization Server to "immediately revoke all access and refresh tokens associated with user ID `employee-123`."
    *   **Non-Repudiable Execution Evidence**: For post-hoc audits and security forensics, simply logging the action is insufficient. A critical requirement is the generation of **durable, non-repudiable, and potentially offline-verifiable evidence** at the moment of execution. This evidence must cryptographically attest to the specific action performed (e.g., "Agent `sec-ops-bot-01` isolated host `laptop-789` at `2024-08-21T10:00:00Z` under policy `POL-456` in response to alert `ALERT-XYZ`"). This formal proof is essential for accountability, especially when an automated agent takes high-risk actions like suspending a user account.

*   **Gap Analysis:**
    *   **What Works (Partially):** The OAuth Client Credentials grant is suitable for giving the security agent its own system-level identity and authority.
    *   **What's Missing (The Gap):**
        *   **Critically Inadequate Revocation API:** This is the most significant gap for this use case. The one-token-at-a-time revocation endpoint in [RFC7009] is completely insufficient for a security incident. The need to make potentially thousands of individual API calls to revoke tokens is too slow and unreliable during an active attack. The lack of a standardized bulk revocation API is a major operational and security failure point.
        *   **Absence of a Standard for Verifiable Action Records**: The OAuth framework focuses on granting and validating the *authority* to perform an action (i.e., possessing a valid token). It does not, however, define a standard mechanism for creating a **durable, cryptographically verifiable record of the action itself** at the moment of execution. In a security context, a simple log entry stating "action performed" is insufficient for high-stakes forensic analysis. What is missing is a formal, non-repudiable piece of evidence that binds the agent's identity, the specific action taken (e.g., "isolate host `laptop-789`"), the policy justification, and the timestamp into a single, verifiable artifact. This gap makes it difficult to construct an undeniable audit trail for automated, high-risk security operations.

#  Analysis of Existing OAuth Extensions

The OAuth 2.0 ecosystem is rich with extensions designed to address security and functionality gaps in the core specification. However, many of these powerful extensions were conceived before the rise of highly autonomous, dynamic, and often ephemeral AI agents. Their design assumptions, therefore, do not always align with the unique challenges presented by agent-centric architectures.

This section analyzes several key existing extensions and related concepts to evaluate their applicability to agent authorization use cases. For each, we identify its strengths, its limitations in agentic scenarios, and potential directions for evolution.

## Rich Authorization Requests (RAR) [RFC9396]

*   **Applicability and Strengths:** RAR represents a significant step beyond simple string-based scopes. It allows clients to request fine-grained, structured, and parameterized permissions. For example, instead of a generic `transaction` scope, a client can request authorization for a specific action like `{"type": "payment", "amount": "50", "currency": "USD", "recipient": "X"}`. This capability is invaluable for creating auditable and least-privilege grants, which is a core requirement for reining in agent capabilities.

*   **Limitations in Agentic Scenarios:** The primary limitation of RAR in agentic scenarios is its static nature. RAR defines the *structure* of a permission, but it assumes the *values* are known at the time of the authorization request. Autonomous agents often operate with non-deterministic logic; they discover the need for specific actions as they execute a task. An agent planning a picnic might not know the exact cost or booking details for a park shelter until it has already completed several other steps. This makes it difficult to request all necessary, fine-grained permissions upfront. In essence, RAR is a powerful Grant-Layer mechanism for defining the shape of authority. It cannot, by design, capture the user's specific, just-in-time consent for a transaction whose final parameters were only determined at the moment of execution. It describes the authority, not the human decision at runtime.

*   **Potential Directions for Evolution:** A potential direction is to evolve RAR or develop a complementary mechanism for "bounded capabilities". This would allow a user to grant an agent a budget or a set of constraints (e.g., "a maximum of $100 for picnic supplies within a 10-mile radius"). The agent could then use this grant to dynamically construct and justify specific RAR-formatted requests at runtime, with the authorization server validating each request against the pre-approved bounds.

## Demonstrating Proof-of-Possession (DPoP) [RFC9449]

*   **Applicability and Strengths:** DPoP enhances security by cryptographically binding access tokens to a specific client's public/private key pair. This effectively prevents token theft and replay attacks, as a stolen token is useless without the corresponding private key. This is a critical security baseline for any system where agents handle sensitive operations.

*   **Limitations in Agentic Scenarios:** The challenge arises in highly dynamic agent architectures. DPoP's model assumes a relatively stable client with a persistent key. This assumption breaks down when a primary agent needs to delegate a task to a dynamically created, ephemeral sub-agent, or when an agent migrates between compute environments. Each new instance would require a new token bound to its new key, creating significant overhead and complexity if it requires a full round-trip to the authorization server. DPoP is a critical Grant-Layer security control, binding a token to a client to prove the legitimacy of the actor possessing the authority. However, it says nothing about the legitimacy of the specific action being performed at the Execution Layer or the user's explicit consent for that action.

*   **Potential Directions for Evolution:** Future work could explore a "multi-hop DPoP" model. In this model, a parent agent holding a DPoP-bound token could securely derive a new, more constrained token for a sub-agent, binding it to the sub-agent's ephemeral key. This would create a verifiable chain of possession and delegation without requiring constant interaction with the central authorization server, making it more suitable for chained or group agent tasks.

## Client-ID Metadata Document (CIMD) [draft-ietf-oauth-client-metadata]

*   **Applicability and Strengths:** CIMD (and the broader concept of dynamic client registration) proposes a powerful shift from static, pre-registered clients to dynamic, discoverable ones. By defining `client_id` as a resolvable URI, an Authorization Server can fetch client metadata on-the-fly. This enables "plug-and-play" onboarding for new agents and greatly simplifies the lifecycle management of cryptographic keys by exposing them via a `jwks_uri`.

*   **Limitations in Agentic Scenarios:** Despite its flexibility, CIMD's current metadata vocabulary is client-application-centric, not agent-centric.
    1.  **Lack of Agent Context:** Standard fields (e.g., `redirect_uris`, `client_name`) fail to describe agent-specific attributes like its level of autonomy, risk profile, or operational policies.
    2.  **Public URI Dependency:** The reliance on a stable, public URI is problematic for ephemeral or internal sub-agents that may not have a persistent, publicly resolvable address.
    3.  **Missing Delegation Traceability:** While CIMD can verify an agent's identity, it does not provide a standard mechanism to describe its lineage or delegation path (i.e., "who created this agent?").

*   **Potential Directions for Evolution:** To bridge these gaps, CIMD could be extended with:
    1.  **Agent-Specific Metadata:** Introduce new fields like `agent_type`, `max_autonomous_budget`, or `human_in_the_loop_policy` to enable better risk assessment by Authorization Servers.
    2.  **Inline or Vouched-For Metadata:** Allow ephemeral agents to present their metadata directly in a request or have it "vouched for" and signed by a trusted parent agent, removing the public URI requirement.
    3.  **Trust Chain Assertions:** Incorporate fields like `parent_agent_id` or integrate concepts from specifications like OpenID Federation to build a verifiable delegation chain.

## Transaction Tokens

*   **Applicability and Strengths:** This refers to a common architectural pattern, often implemented using JWTs, where initial user context is encoded into a token and propagated through a series of internal microservices. This ensures consistent authorization and context within a trusted security domain.

*   **Limitations in Agentic Scenarios:** The fundamental gap is one of trust boundaries. This pattern is designed for use within a single, trusted system. It is ill-suited for multi-hop delegation scenarios involving untrusted third-party agents, as the tokens lack standard mechanisms for permission attenuation (reduction) or enforcement across organizational boundaries.

*   **Potential Directions for Evolution:** To be viable in agent ecosystems, this concept would need to evolve into a standardized "Attenuating Agent Token". Such a token would need a formal, interoperable mechanism for a parent agent to reduce permissions before passing it to a child agent, ensuring the child cannot exceed the parent's authority. Furthermore, embedding verifiable proofs of execution could enhance end-to-end auditability.
                 
# Summary of Major Gaps

The use cases above highlight several fundamental gaps between the needs of AI agents and the capabilities of the standard OAuth 2.x framework:

1.  **From 'Pre-Approval' to 'Continuous Dialogue': A Paradigm Shift.** OAuth's model is to get all permissions upfront. Agents need a continuous, interactive authorization model where permissions are granted dynamically and just-in-time as a task evolves from a high-level intent.

2.  **Lack of a Standardized Interactive Channel.** The framework has no built-in mechanism for an agent to "pause" and securely ask the user for an intermediate decision or to respond to a real-time authorization challenge from a resource server.

3.  **Inability to Represent Delegation Chains.** Standard tokens cannot securely represent a multi-step delegation chain (`User -> Agent A -> Agent B`). This is a critical blocker for automating complex, multi-agent business processes.

4.  **Insufficient Revocation Mechanisms.** The single-token revocation API is inadequate. The lack of standardized APIs for task-level and bulk (per-user or per-client) revocation is a major operational and security deficiency.

5.  **Authorization Is Modeled Per-Client, Not Per-Group.** OAuth has no notion of a set of clients acting as one task group under a single grant: no group membership representation, no admission of late-selected members, and no group-level lifecycle or atomic revocation.

6.  **The Grant-Layer vs. Execution-Layer Gap:** The existing OAuth framework excels at defining Grant-Layer Authority—what an agent is allowed to do. However, it lacks a standard mechanism for generating Execution-Layer Evidence—a non-repudiable, cryptographic proof of a user's explicit consent for a specific, high-risk action at the moment it occurs. This gap is critical for auditability and dispute resolution, as grant-layer tokens prove potential, not the legitimacy of a specific, executed transaction.

# Security Considerations

As we design new authorization mechanisms for agents, security must be the primary concern. The autonomy of agents amplifies the risk of any vulnerability.

*   **Risk of Over-Privileging:** The current lack of dynamic authorization tempts developers to request broad, long-lived permissions ("god tokens"), dramatically increasing the damage if an agent is compromised. Future solutions must make it easy to follow the Principle of Least Privilege.
*   **Delegation Chain Vulnerabilities:** Without a standard for secure delegation chains, custom implementations are prone to "Confused Deputy" attacks, where an agent is tricked into misusing its authority.
*   **Revocation Timeliness:** In a world of powerful, autonomous agents, the ability to instantly and completely revoke all permissions for a compromised user or agent is not a "nice-to-have"; it is an absolute necessity.
*   **Non-Repudiation:** For enterprise and B2B scenarios, actions taken by agents must be cryptographically auditable and non-repudiable, creating a strong digital paper trail.
*   **Coordinator as a Single Point of Authority:** In coordinated task groups, the leading agent both requests authorization on behalf of all members and assigns their tasks. If the coordinator is compromised, the blast radius covers the entire group; its authority to act as an applier for others must therefore be separately authenticated, authorized, and auditable.
*   **Durable Record of the Authorization Grant:** While agent actions must be non-repudiable, this is insufficient without a durable record of the authorization grant event itself. This record must capture the identity of the authorizing principal, the exact scopes granted, the time of the grant, and the policy version in effect, which disappear when a grant ends, this event record must persist to provide a verifiable source of truth for post-hoc audits and to answer the critical question: "What did the human actually authorize?".
*   **Agent-Operable Consent Surfaces:** AI agents can click "Approve" buttons themselves, so a simple click is no longer proof of human consent. We need approval methods that agents can't fake, like a confirmation on your phone or a fingerprint scan.

# IANA Considerations

This document has no IANA actions.

# Acknowledgements

The analysis and use cases in this document are derived from observations of emerging AI agent technologies and their application trends across various industries. Thanks are due to the OAuth community for their past and ongoing efforts in building a secure and interoperable authorization framework, upon which this work is built.

--- back

# References

[RFC6749]
: Hardt, J., Ed., "The OAuth 2.0 Authorization Framework", RFC 6749, DOI 10.17487/RFC6749, October 2012, <https://www.rfc-editor.org/info/rfc6749>.

[RFC7009]
: Lodderstedt, T., Ed., Dronia, S., and M. Scurtescu, "OAuth 2.0 Token Revocation", RFC 7009, DOI 10.17487/RFC7009, August 2013, <https://www.rfc-editor.org/info/rfc7009>.

[RFC8693]
: Jones, M., Nadalin, A., Campbell, B., Ed., Bradley, J., and C. Mortimore, "OAuth 2.0 Token Exchange", RFC 8693, DOI 10.17487/RFC8693, January 2020, <https://www.rfc-editor.org/info/rfc8693>.

[RFC9396]
: Lodderstedt, T., Richer, J., and B. Campbell, "OAuth 2.0 Rich Authorization Requests", RFC 9396, DOI 10.17487/RFC9396, May 2023, <https://www.rfc-editor.org/info/rfc9396>.

[I-D.song-oauth-ai-agent-collaborate-authz]
: Song, Y., Li, L., Jiang, Y., and F. Liu, "OAuth2.0 Extension for Multi-AI Agent Collaboration", Work in Progress, Internet-Draft, draft-song-oauth-ai-agent-collaborate-authz-01, 28 February 2026, <https://datatracker.ietf.org/doc/html/draft-song-oauth-ai-agent-collaborate-authz-01>.
