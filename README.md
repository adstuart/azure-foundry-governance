# Azure AI Foundry Architecture Decision

## Decision Topic
**Single Azure AI Foundry Resource vs Multiple Foundry Resources**

This document evaluates the architectural decision between deploying **a single Azure AI Foundry resource** or **multiple Foundry resources** across an environment.

The comparison evaluates trade-offs across operational, security, networking, and lifecycle considerations.

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
- Advantage 2
- Advantage 3

### Disadvantages

- Max 250 projects in a single Foundry resource
- Single BYOVNet for agents. All agents reside in same Subnet. Increases reliance on VNet Peering and Firewall transit for agent traffic.
- Disadvantage 3

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
- Advantage 2
- Advantage 3

### Disadvantages

- /24 recommended per VNet in BYOVNet model, run out of IP quickly
- IP requirement could drive interest in Managed VNet (rather than BYOVNet) and therefore you need to consider how managed VNet hosted agents, reach centralised models (Private endpoints, AI Gateway, Azure Firewall/Managed-VNet etc)
- Disadvantage 2
- Disadvantage 3

---

# Associated links

- Foundry limits https://learn.microsoft.com/en-us/azure/foundry/foundry-models/quotas-limits
- 
