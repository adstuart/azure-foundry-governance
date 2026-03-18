# Azure AI Foundry Architecture Decision

**Work in Progress**

## Decision Topic
**Single Azure AI Foundry Resource vs Multiple Foundry Resources**

- This document evaluates the architectural decision between deploying **a single Azure AI Foundry resource** or **multiple Foundry resources** across an environment.
- In both scenarios it is imagined that the requirements are to accommodate both:
    - a) a central repository of models and LLM access governed by central IT
    - b) areas where individual teams can build out Agent-based workloads
- The comparison evaluates trade-offs across operational, security, networking, and lifecycle considerations.

> NB. This is focused on Microsoft Foundry as of March 2026. Not the older AI Foundry with the concept of Hubs etc. See https://journeyofthegeek.com/2026/02/22/microsoft-foundry-the-evolution-revisited/

---

# Options

## Option A — Single Foundry Resource

A single Azure AI Foundry resource shared across environments, teams, or workloads. 

```text
Central AI Subscription
└── Resource Group
    └── Azure AI Foundry
        ├── Models
        ├── Agents
        └── Projects
            └── Central AI Project
            └── BU A Project
            └── BU B Project
```

### Advantages

- Single BYOVNet only requires one /24, less IP required
- All projects in a single Foundry resource can share a PTU Deployment (could also be a disadvantage if worried about noisy neighbour). Note, PTU Reservations (not deployments) CAN be shared across Foundry resources. In all cases you cannot share PTU deployment/reservation across regions.
- Not strictly an advantage, but worth noting, the Foundry resource itself as a "bucket" is not chargeable. (You pay for the models, agents and tools)
- *Simpler ownership model — a single central IT / cloud platform team can own and govern the Foundry resource, models, and tooling without negotiating boundaries with BUs*
- *Easier to enforce consistent model governance (approved model list, throughput limits, auditability) from a single control plane*
- *Natural fit for organisations with strong central IT culture (e.g., banking, regulated industries) where a single team already controls cloud infrastructure*

### Disadvantages

- Max 250 projects in a single Foundry resource
- You only get a Single BYOVNet for agents if using this private customer VNet model. All agents reside in same Subnet. Increases reliance on VNet Peering and Firewall transit for agent traffic if calling in/out of other VNets.
- All projects, prior to publishing, share a common managed-identity for agents. (Agents get their own Entra ID if published).
- All projects share some Foundry level datastores/connections. This has security and chargeback implications and complexity. E.g.
    - Agent conversations (Cosmos DB) (DB vs container level identity separation)
    - Vector Stores (AI Search) (Need to pay special attention to RBAC at Index vs Search resource levels)
    - Storage Accounts 
- *Risk of noisy-neighbour effects across BU projects sharing the same Foundry resource (PTU, datastores, identity) — harder to isolate blast radius*
- *Chargeback complexity — shared datastores (Cosmos DB, AI Search) make it difficult to attribute costs per BU without additional tagging/metering*



---

## Option B — Multiple Foundry Resources

Multiple Azure AI Foundry resources separated by environment, workload, or team.

```text
Central AI Subscription
└── Resource Group
    └── Azure AI Foundry
        ├── Models
        ├── Agents
        └── Projects
     
BU A Subscription
└── Resource Group
    └── Azure AI Foundry
        ├── Models
        ├── Agents
        └── Projects

BU B Subscription
└── Resource Group
    └── Azure AI Foundry
        ├── Models
        ├── Agents
        └── Projects
```

### Advantages

- 250 projects per Foundry resource
- Advocated approach in MS Docs. [Baseline Microsoft Foundry chat reference architecture in an Azure landing zone](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-landing-zone#:~:text=Instead%2C%20this%20architecture%20treats%20the%20workload%20as%20the%20owner%20of%20the%20Foundry%20resource%2C%20which%20is%20the%20recommended%20approach)
- Each BU gets their own BYOVNet for agents if using this private customer VNet model. Isolation between Agents of different BU. If Agent subnet is in same VNet as other BU workloads, no VNet Peering charges.
- All BU get their own unique shared identity for "work in progress" Agents. I.e. prior to publishing, it is easy to isolate Agent identity. (Agents get their own Entra ID after publishing in all cases)
- All projects have their own specific Foundry level datastores/connections. This has security and chargeback benefits, but cost considerations:
    - Agent conversations (Cosmos DB) (DB vs container level identity separation) >> Per BU
    - Vector Stores (AI Search) (Need to pay special attention to RBAC at Index vs Search resource levels) >> Per BU
    - Storage Accounts >> Per BU
- *Aligns with federated operating models (e.g., large diversified conglomerates) where BUs have autonomy over their own infrastructure and budgets*
- *Each BU owns their agents end-to-end (development, testing, evaluation, security), with central teams providing guardrails via approved templates and Azure Policy*
- *Cleaner chargeback — each Foundry resource and its backing datastores are scoped to a single BU subscription, making cost attribution straightforward*

### Disadvantages

- /24 recommended per VNet in BYOVNet model, run out of IP quickly
- IP requirement could drive interest in Managed VNet (rather than BYOVNet) and therefore you need to consider how managed VNet hosted agents, reach centralised models (Private endpoints, AI Gateway, Azure Firewall/Managed-VNet etc)
  - Note that Managed VNet uses one Azure Firewall per instance if using outbound FQDN filtering = cost consideration
- Each Foundry resource requires its own PTU deployments (can come from same PTU reservation)
- *BU Foundry instances need a mechanism to consume centrally governed models — this introduces a dependency on AI Gateway / APIM (see section below) with associated networking and cost overhead (Private APIM premium tier, failover, scaling)*
- *More moving parts to govern — central team must provide and maintain approved Foundry deployment templates, Azure Policy guardrails, and a model consumption gateway*

---

# *AI Gateway / APIM — Cross-Foundry Model Access*

Regardless of Option A or B, most enterprises want **centralised model governance** (approved model list, throughput limits, cost controls, auditability) combined with **distributed consumption** by BU teams.

In Option B this becomes an explicit architectural concern — BU Foundry instances need to call models hosted in the central Foundry. The emerging pattern is:

```text
BU Foundry Instance ──► Azure API Management (AI Gateway) ──► Central Foundry Models
```

### Key Considerations

- **APIM as enforcement & observability layer** — rate limiting, token metering, logging, and policy enforcement sit in APIM, consistent with how enterprises already govern APIs
- **Private APIM** — required for private-network deployments; requires Premium tier with associated cost
- **Networking** — Private endpoints from BU VNets/Managed VNets to APIM, plus APIM to central Foundry model endpoints. Consider failover, latency, and cross-region implications
- **Scaling** — AI agent workloads can generate high volumes of API calls; APIM capacity planning is critical
- **Cost** — Premium APIM + networking (Private Link, VNet peering) adds infrastructure cost on top of model consumption costs

### Associated Links

- [AI Gateway landing zone](https://learn.microsoft.com/en-us/ai/playbook/technology-guidance/generative-ai/dev-starters/genai-gateway)
- [APIM AI Gateway capabilities](https://learn.microsoft.com/en-us/azure/api-management/api-management-ai-gateway-overview)

---

# *Shadow AI / Compliance Controls*

A governance concern that exists **regardless of whether you choose Option A or B**: the risk of developers bypassing approved Foundry models and calling third-party AI providers directly (e.g., external commercial APIs, open-source model endpoints).

Policy alone is insufficient — **technical controls** are required:

| Control | Purpose |
|---|---|
| **Azure Policy** | Prevent deployment of unapproved AI resources / model endpoints |
| **Defender for Cloud** | Detect anomalous outbound calls to AI provider endpoints |
| **Network controls** | Restrict outbound internet access from agent subnets; allowlist only approved endpoints |
| **Purview / DLP** | Monitor and prevent sensitive data from being sent to unapproved AI services |
| **Azure Firewall / NSGs** | Enforce egress filtering at the network level |

This should be treated as a **separate but parallel governance workstream** — it applies to both centralised and distributed Foundry topologies.

---

# *Ownership & Operating Model*

### Who Owns the Foundry?

| Ownership Model | When It Fits |
|---|---|
| **Central IT / Cloud Platform Team** | Strong central governance culture, regulated industries (banking, healthcare) |
| **Chief Data Office** | Organisation where AI/ML is a data office function, CDO drives AI strategy |
| **Federated BU Ownership** | Large diversified organisations where BUs operate semi-independently |

In practice, a hybrid is common: **central teams own governance, tooling, and model access** while **BUs own their Foundry instances and agents** (deployed via approved templates, charged back for usage).

### Agent Ownership

Agents themselves are typically **owned by product / application teams**, who are responsible for:
- Development and testing
- Evaluation and monitoring (accuracy, safety, groundedness)
- Security (data access, identity, prompt injection mitigation)
- Lifecycle management (versioning, deprecation)

Central teams provide the **platform and guardrails**, not the agents.

---

# *Zooming Out — Foundry as Playground vs. Production Deployment*

The Option A vs. B discussion above focuses on Foundry resource topology, but it's important to zoom out and recognise that **Foundry is often used as a development and prototyping environment** — a playground where teams build, test, and evaluate agents before promoting them to production.

When it comes to **production deployment**, the architecture may look completely different. Many enterprises will choose to run their agent code **outside of Foundry entirely**, on their own compute — for example, on an enterprise-grade AKS cluster — while still consuming Foundry-hosted models via API.

```text
Development / Prototyping              Production
┌─────────────────────────┐            ┌─────────────────────────┐
│   Azure AI Foundry      │            │   Enterprise AKS        │
│   ├── Models            │            │   ├── Agent containers  │
│   ├── Agent playground  │  ──────►   │   ├── Custom middleware │
│   └── Evaluation tools  │            │   └── Observability     │
└─────────────────────────┘            └────────────┬────────────┘
                                                    │
                                        ┌───────────▼───────────┐
                                        │  Foundry Models (via  │
                                        │  AI Gateway / APIM)   │
                                        └───────────────────────┘
```

In this model:
- **Foundry** remains the control plane for model governance, evaluation, and experimentation
- **Production agent runtime** lives on your own infrastructure (AKS, App Service, Container Apps, or on-premises), giving you full control over scaling, networking, security, and SLA
- **Model access** is still governed centrally via AI Gateway / APIM, regardless of where the agent code runs

### Microsoft Agent Framework

The [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/overview/) is an open-source, cross-language (Python and .NET) framework designed for building and orchestrating production-ready AI agents. It is particularly relevant here because:

- **Cloud-agnostic runtime** — agents built with the Agent Framework can run anywhere: AKS, on-premises Kubernetes, VMs, or container services. There is no hard dependency on Foundry for the agent runtime itself
- **Foundry integration** — while the runtime is portable, agents can still consume Foundry-hosted models, tools, and evaluation capabilities via standard APIs
- **Production-grade features** — built-in support for multi-agent orchestration, persistent state, human-in-the-loop, OpenTelemetry observability, and type safety — features that enterprises need when moving beyond the playground
- **Foundry Local** — for scenarios requiring full data sovereignty, Foundry Local allows hosting both the agent runtime and models entirely on your own infrastructure

### Why This Matters for Governance

This "playground vs. production" distinction has direct governance implications:

- The **Foundry topology decision** (Option A vs. B) primarily governs the **development and model access** layer
- The **production deployment decision** may be governed by entirely different teams (platform engineering, SRE) using different infrastructure patterns
- Organisations should plan for both — Foundry governance for model access and experimentation, and standard application governance (CI/CD, RBAC, network policies) for the production agent runtime

---

# So what should I do?

Conclusion and recommendations

*There is no single "right" answer — the best choice depends on your organisation's size, regulatory posture, operating model, and AI maturity. The goal is to **choose with eyes open**, understanding the trade-offs.*

### *Lean toward Option A (Single Foundry) if:*

- *You have a **small number of teams** building AI workloads (well under the 250 project limit)*
- *Your organisation has a **strong central IT culture** with a single team governing cloud infrastructure*
- *You want to **minimise infrastructure complexity** and networking overhead*
- ***IP address space** is constrained and you want to limit BYOVNet /24 consumption*
- *You are in **early stages of AI adoption** and want to start simple before scaling out*

### *Lean toward Option B (Multiple Foundry Resources) if:*

- *You are a **large or diversified organisation** with semi-autonomous BUs (each with their own subscriptions and budgets)*
- *You need **strong isolation** between BU workloads (identity, networking, datastores)*
- ***Chargeback clarity** is a hard requirement — each BU needs clean cost attribution*
- *You expect to **scale beyond 250 projects** or want headroom for growth*
- *Your BUs have **different compliance or data residency requirements** that benefit from separate Foundry instances*
- *You are willing to invest in **AI Gateway / APIM infrastructure** to bridge central models to distributed Foundry instances*

### *In all cases:*

- *Establish **centralised model governance** — an approved model catalogue with access controls, regardless of topology*
- *Implement **Shadow AI technical controls** — Azure Policy, Defender, network egress filtering, Purview/DLP*
- *Define a clear **ownership model** — who owns the platform vs. who owns the agents*
- *Use **Infrastructure as Code** (Bicep/Terraform) with approved templates so BUs can self-serve within guardrails*

---

# Associated links

- Foundry limits https://learn.microsoft.com/en-us/azure/foundry/foundry-models/quotas-limits
- *[AI Gateway landing zone](https://learn.microsoft.com/en-us/ai/playbook/technology-guidance/generative-ai/dev-starters/genai-gateway)*
- *[APIM AI Gateway capabilities](https://learn.microsoft.com/en-us/azure/api-management/api-management-ai-gateway-overview)*
- *[Baseline Microsoft Foundry chat reference architecture](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-landing-zone)*
- *[Microsoft Agent Framework overview](https://learn.microsoft.com/en-us/agent-framework/overview/)*
