# Azure AI Foundry (Hub Model) – Service-Provider Architecture with ExpressRoute & Private Endpoints

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE) ![Azure AI Foundry](https://img.shields.io/badge/Azure%20AI%20Foundry-Hub%20Model-008AD7) ![Private Link](https://img.shields.io/badge/Network-Private%20Link%20%2F%20ExpressRoute-5e527f) ![Mermaid](https://img.shields.io/badge/Diagram-Mermaid-1f425f)

This repository documents a **secure, multi-customer architecture** for Azure AI Foundry using the **Hub Model**, integrated with **ExpressRoute**, **Private Endpoints**, and **Private DNS**. It’s designed for **service providers** hosting multiple customer projects within the same Azure tenant.

---

## 🧭 Overview

This architecture pattern allows each internal customer (department, agency, or division) to consume Azure AI Foundry capabilities through a centralized **Hub**, with shared network isolation and governance. All traffic flows privately through **ExpressRoute**, avoiding public endpoints.

---

## 🧩 Architecture Diagram (Mermaid)

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
      DNS["Private DNS Zones - privatelink"]
      FW["Azure Firewall / NVA (optional)"]
    end

    subgraph Projects[Azure AI Foundry Hub + Projects]
      HUB[Hub (AML Workspace)]
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

---

## ⚙️ Deployment Steps

### 1️⃣ Prerequisites
- ExpressRoute private peering established to Hub VNet.
- Required resource providers registered: `Microsoft.MachineLearningServices`, `Microsoft.Network`, `Microsoft.ContainerRegistry`, etc.
- Azure CLI ≥ 2.63 or PowerShell Az ≥ 11.

### 2️⃣ Create the Hub
```bash
az ml workspace create   --name aif-hub-prod-wus3   --resource-group rg-prod-aif-hub   --location westus3
```

### 3️⃣ Create Private Endpoints
```bash
# Hub (AML Workspace)
az network private-endpoint create   -g rg-prod-net -n pe-aifhub   --subnet snet-pe   --private-connection-resource-id $(az ml workspace show -n aif-hub-prod-wus3 -g rg-prod-aif-hub --query id -o tsv)   --group-ids amlworkspace   --connection-name peconn-aifhub
```
Repeat for **Storage**, **Key Vault**, and **ACR**.

### 4️⃣ Configure Private DNS Zones
```bash
for Z in   privatelink.api.azureml.ms   privatelink.notebooks.azure.net   privatelink.vaultcore.azure.net   privatelink.blob.core.windows.net   privatelink.azurecr.io; do
  az network private-dns zone create -g rg-prod-dns -n $Z
  az network private-dns link vnet create -g rg-prod-dns -z $Z -n link-$Z     -v $(az network vnet show -g rg-prod-net -n vnet-hub-prod-wus3 --query id -o tsv) --registration-enabled false
 done
```

### 5️⃣ Disable Public Access (after validation)
```bash
az ml workspace update -n aif-hub-prod-wus3 -g rg-prod-aif-hub --public-network-access Disabled
az keyvault update --name kv-prod-aif-hub --resource-group rg-prod-aif-hub --public-network-access Disabled
az acr update --name acrprodwus3 --resource-group rg-prod-aif-hub --public-network-enabled false
az storage account update --name stgprodwus3 --resource-group rg-prod-aif-hub --public-network-access Disabled
```

### 6️⃣ Validate
- ✅ DNS resolves to private IPs (10.x.x.x)
- ✅ Hub and project access functional via Private Link
- ✅ Public access disabled

---

## 🔒 Governance & Operations
- Use AAD groups for role assignments per project.
- Enforce Azure Policy: `Deny-PublicNetworkAccess` + `Deploy-PrivateEndpoint`.
- Log diagnostics to central Log Analytics.
- Apply Defender for Cloud and cost tags (`env`, `agency`, `project`).

---

## 🧱 Repository Structure Example
```
/azure-ai-foundry-hub/
├── README.md                   ← this file
├── /scripts/                   ← optional automation scripts
├── /diagrams/                  ← exported PNGs or .mmd files
└── LICENSE
```

---

## 🪶 License
Use MIT or Microsoft Open Source license model as appropriate.

---

## ✅ References
- [Configure Private Link for Azure AI Foundry Hub](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/hub-configure-private-link)
- [Azure AI Foundry Architecture Baseline](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-azure-ai-foundry-landing-zone)
- [Private Link + ExpressRoute FAQ](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-faq#can-i-access-azure-paas-services-over-an-expressroute-connection)
