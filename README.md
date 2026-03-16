# Azure AI Foundry Architecture Decision

**Work in Progress**

To explore:
- AI Gateway / APIM mapping implications.

## Decision Topic
**Single Azure AI Foundry Resource vs Multiple Foundry Resources**

- This document evaluates the architectural decision between deploying **a single Azure AI Foundry resource** or **multiple Foundry resources** across an environment.
- In both scenarios it is imagined that the requirements are to accomodate both:
    - a) a central repositry of models and LLM access governed by central IT
    - b) areas where individual teams can build out Agent-based workloads
- The comparison evaluates trade-offs across operational, security, networking, and lifecycle considerations.

> NB. This is focused on Foundry as of March 2026. Now the older Foundry with hubs etc. See https://journeyofthegeek.com/2026/02/22/microsoft-foundry-the-evolution-revisited/

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

- Single BYOVNet only requires on /24, less IP required
- All projects in a single Foundry resource can share a PTU Deployment (could also be a disadvantage if worried about noisey neighbour). Note, PTU Reservations (not deployments) CAN be shared across Foundry resources. In all cases you cannot share PTU deployment/reservation across regions.
- Advantage 3

### Disadvantages

- Max 250 projects in a single Foundry resource
- Single BYOVNet for agents. All agents reside in same Subnet. Increases reliance on VNet Peering and Firewall transit for agent traffic.
- All projects, prior to publishing, share a common managed-identity for agents. (Agents get their own Entra ID if published).
- All projects share some Foundry level datastores. This has security and chargeback implicatons and complexity. E.g.
    - Agent conversations (Cosmos DB) (DB vs container level identity seperation)
    - Vector Stores (AI Search) (Need to pay special attention to RBAC at Index vs Search resource levels)
    - Storage Accounts 



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
- Adovcated approach in MS Docs. [Baseline Microsoft Foundry chat reference architecture in an Azure landing zone](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-landing-zone#:~:text=Instead%2C%20this%20architecture%20treats%20the%20workload%20as%20the%20owner%20of%20the%20Foundry%20resource%2C%20which%20is%20the%20recommended%20approach)
- Advantage 3

### Disadvantages

- /24 recommended per VNet in BYOVNet model, run out of IP quickly
- IP requirement could drive interest in Managed VNet (rather than BYOVNet) and therefore you need to consider how managed VNet hosted agents, reach centralised models (Private endpoints, AI Gateway, Azure Firewall/Managed-VNet etc)
  - Note that Managed VNet uses one Azure Firewall per instance if using outbound FQDN filtering = cost consideration
- Disadvantage 2
- Disadvantage 3

---

# So what should I do?

Conclusiona and recommendations

---

# Associated links

- Foundry limits https://learn.microsoft.com/en-us/azure/foundry/foundry-models/quotas-limits
- 
