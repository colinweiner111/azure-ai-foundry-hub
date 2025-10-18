# Azure AI Foundry (Hub Model) – Service-Provider Architecture with ExpressRoute & Private Endpoints

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE) ![Azure AI Foundry](https://img.shields.io/badge/Azure%20AI%20Foundry-Hub%20Model-008AD7) ![Private Link](https://img.shields.io/badge/Network-Private%20Link%20%2F%20ExpressRoute-5e527f) ![Mermaid](https://img.shields.io/badge/Diagram-Mermaid-1f425f)

This repository documents a **secure, multi-customer architecture** for Azure AI Foundry using the **Hub Model**, integrated with **ExpressRoute**, **Private Endpoints**, and **Private DNS**.

---

## Architecture Diagram (Mermaid)

```mermaid
flowchart LR
    subgraph OnPrem[Customer Networks]
      A1[Agency A LAN]
      A2[Agency B LAN]
    end

    subgraph ER[ExpressRoute Private Peering]
      ERGW[ER Circuit]
    end

    subgraph HubVNet[Hub VNet]
      PEHUB[Private Endpoint: Foundry Hub]
      PESTG[Private Endpoint: Storage]
      PEKV[Private Endpoint: Key Vault]
      PEACR[Private Endpoint: ACR]
      DNS[Private DNS Zones - privatelink]
      FW[Azure Firewall or NVA]
    end

    subgraph Projects[Azure AI Foundry Hub and Projects]
      HUB[Hub - AML Workspace]
      PRJA[Project A]
      PRJB[Project B]
    end

    A1 --- ERGW --- HubVNet
    A2 --- ERGW
    HubVNet ---|VNet Peering| Projects

    HUB --- PEHUB
    HUB --- PESTG
    HUB --- PEKV
    HUB --- PEACR
    DNS --- HubVNet
    PRJA --> HUB
    PRJB --> HUB
```
