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

- Advantage 1
- Advantage 2
- Advantage 3

### Disadvantages

- Disadvantage 1
- Disadvantage 2
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

- Advantage 1
- Advantage 2
- Advantage 3

### Disadvantages

- Disadvantage 1
- Disadvantage 2
- Disadvantage 3

---
