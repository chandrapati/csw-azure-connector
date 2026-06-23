# Cisco Secure Workload — Azure Connector Guide

> **Disclaimer:** Community reference guide by Cisco Solutions Engineering. Always consult the [official Cisco Secure Workload documentation](https://www.cisco.com/c/en/us/products/security/secure-workload/index.html) — specifically [Configure and Manage Connectors → Azure Connector](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-and-manage-connectors-for-secure-workload.html) — for authoritative guidance.

## Table of Contents
1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Capabilities](#3-capabilities)
4. [Prerequisites](#4-prerequisites)
5. [Step A — Azure App Registration & Permissions](#5-step-a--azure-app-registration--permissions)
6. [Step B — Configure the Azure Connector in CSW](#6-step-b--configure-the-azure-connector-in-csw)
7. [Step C — VNet and Capability Configuration](#7-step-c--vnet-and-capability-configuration)
8. [AKS Integration](#8-aks-integration)
9. [Segmentation with Azure NSGs](#9-segmentation-with-azure-nsgs)
10. [Verification](#10-verification)
11. [Limits](#11-limits)
12. [Troubleshooting](#12-troubleshooting)
13. [Related Resources](#13-related-resources)

---

## 1. Overview

The **Azure connector** in Cisco Secure Workload automatically ingests workload inventory and tags from **Azure Virtual Networks (VNets)**, streams **Azure VNet flow logs**, and enforces segmentation policy using **Azure native Network Security Groups (NSGs)** — all without deploying CSW agents on every Azure VM.

The connector supports **multiple Azure subscriptions** from a single connector instance, providing unified visibility across a complex Azure footprint.

### What the Azure connector provides

| Capability | Description | Use case |
|------------|-------------|---------|
| **Label ingestion** | Azure VM and NIC tags → CSW labels (real-time sync) | Scope automation |
| **Flow log ingestion** | VNet flow logs from Azure Storage → CSW for ADM | Traffic visualization |
| **Segmentation** | CSW policies → Azure NSG rules (auto-programmed) | Cloud-native enforcement |
| **AKS integration** | K8s node/pod/service metadata from AKS | Container workload visibility |

> **No virtual appliance required.** Unlike the NetFlow / ERSPAN / ISE connectors, the Azure cloud connector does **not** run on a Secure Workload Ingest or Edge virtual appliance — Secure Workload connects to the Azure Resource Manager APIs directly.

> **Beyond VMs:** The Azure connector also discovers and ingests Standard Load Balancers, Application Gateways, Private Link Services, Private Endpoints, Azure SQL Servers, and Azure Function Apps — modeling frontend IPs/rules as services and backend pools as machines. Only resources associated with the **configured VNet** are discovered (resources in other VNets in the same subscription are not).

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Azure Subscription                                  │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  VNet (East US)                                               │   │
│  │                                                               │   │
│  │  Azure VMs with tags:                                         │   │
│  │    Environment=Production, Tier=App, Team=Finance             │   │
│  │                                                               │   │
│  │  VNet Flow Logs ─────────────────────► Azure Storage Account  │   │
│  │                                                               │   │
│  │  NSGs on subnets/NICs ◄── CSW programs rules                  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                  ▲ Azure Resource Manager API                         │
│                  │                                                    │
│  Azure AD App Registration (client ID + secret/cert)                  │
│  Custom Role with least-privilege permissions (ARM template)          │
└──────────────────────────────────────────────────────────────────────┘
                   ▲
                   │ Azure REST API
                   │
┌──────────────────┴───────────────────────────────────────────────────┐
│  Cisco Secure Workload (SaaS or On-Prem)                              │
│  Azure Connector (no virtual appliance needed)                        │
│                                                                       │
│  • Polls Azure RM API for tag changes                                 │
│  • Reads VNet flow logs from Storage Account                          │
│  • Programs NSG rules when enforcement enabled                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. Capabilities

### 3.1 Label Ingestion
- Azure VM tags + Network Interface tags → CSW workload labels
- Tags from both VMs and NICs are merged when both are configured
- Real-time synchronization as Azure tags change

### 3.2 VNet Flow Log Ingestion
- Requires Azure **VNet flow logs (Version 2)** enabled and stored in an **Azure Storage Account**
- Retention: minimum 2 days (connector pulls new flow data every minute)
- Storage account must have **storage account key access enabled**, and be reachable from the Secure Workload cluster
- **Flow-log limitations to set expectations:** Azure VNet flow logs capture **TCP and UDP only — no ICMP** flow data, and do **not** capture **TCP flag** information

### 3.3 Segmentation
- Requires Gather Labels to be enabled
- CSW policies → Azure NSG rules applied to subnets and NICs. Azure NSGs support both Allow and Deny rules
- When segmentation is enabled for a VNet, the **subnet-level NSG is set to allow-all** and CSW overwrites the **interface-level (NIC) NSGs**; an interface NSG is created automatically if one is not already present. A subnet with **no NSG** is **not** enforced
- **Warning:** Enabling segmentation **removes all existing rules** from the associated NSGs. CSW automatically backs them up first (see §9); take your own backup as defense-in-depth

### 3.4 AKS Integration
- Node, pod, and service metadata from Azure Kubernetes Service clusters

---

## 4. Prerequisites

### Azure requirements
- [ ] Azure subscription ID
- [ ] **Azure AD App Registration** created (Application/Client ID, Directory/Tenant ID, Client Secret or Certificate)
- [ ] **ARM template applied** to grant required permissions to the App Registration (generated by CSW connector wizard)
- [ ] Storage account key access **enabled** on storage accounts used for flow logs
- [ ] For flow logs: VNet flow logs enabled (Version 2), published to Azure Storage Account
- [ ] For segmentation: Existing NSG rules backed up

### CSW requirements
- [ ] CSW cluster administrator access
- [ ] Target VRF/tenant configured for Azure workloads
- [ ] Each VNet belongs to exactly **one** Azure connector (across all connectors in the cluster)

---

## 5. Step A — Azure App Registration & Permissions

### A1 — Create an Azure AD App Registration

1. Azure Portal → **Azure Active Directory > App registrations > New registration**
2. Name: e.g., `csw-connector-app`
3. Supported account types: **Single tenant**
4. Click **Register**
5. Note:
   - **Application (client) ID**
   - **Directory (tenant) ID**

### A2 — Create client credentials

**Option A — Client secret (simpler):**
1. App registration → **Certificates & secrets > New client secret**
2. Set expiry (12 or 24 months — plan for rotation)
3. Copy the **secret value** immediately (shown only once)

**Option B — Certificate (more secure):**
1. Generate a certificate and upload the public key under **Certificates & secrets > Certificates**
2. Use the private key in CSW connector configuration

### A3 — Apply the ARM template for permissions

The CSW connector wizard generates an **Azure Resource Manager (ARM) template** that creates a custom role with exactly the permissions needed:

1. Start the connector wizard in CSW (Step B below)
2. Select your desired capabilities
3. Download the ARM template from the wizard
4. Apply in Azure via Portal or CLI:
   ```bash
   az deployment sub create \
     --location eastus \
     --template-file csw-connector-role.json \
     --parameters principalId=<App-Registration-Object-ID>
   ```
5. The custom role is applied at the subscription level

---

## 6. Step B — Configure the Azure Connector in CSW

### B1 — Navigate to connector configuration

1. CSW UI: **Manage > Workloads > Connectors**, then choose the **Azure** cloud connector and start the wizard
2. Select capabilities to enable. **For a POV, enable Gather Labels + Ingest Flow Logs first and leave Segmentation off** until you are ready to enforce (see §9)

### B2 — Enter Azure credentials

| Field | Value | Where to find |
|-------|-------|---------------|
| **Application (Client) ID** | App Registration client ID | Azure AD > App registrations > Overview |
| **Directory (Tenant) ID** | Azure AD tenant ID | Azure AD > Overview |
| **Client Secret** | Secret value from Step A2 | Certificates & secrets (saved at creation) |
| **Subscription ID** | Azure subscription ID | Subscriptions blade |

> The Azure connector supports **multiple subscriptions** — you can add additional subscription IDs after initial configuration.

### B3 — VNet discovery

After credentials are entered, CSW discovers all VNets across the configured subscriptions.

---

## 7. Step C — VNet and Capability Configuration

For each selected VNet:

### Labels / Inventory
Select VM tags and NIC tags to ingest. All tags become CSW labels:
```
Azure tag: Environment=Production
CSW label: Environment = Production
```

### Flow Log Ingestion

Ensure VNet flow logs are enabled in Azure:
1. Azure Portal → **Monitor > Network Insights > Traffic > Flow logs > + Create**
2. Select target VNet, set to **Version 2**, choose Storage Account as destination
3. Enable retention: 2+ days
4. In CSW connector: provide the **Storage Account name** and access credentials

### Segmentation
> **Best practice (per Cisco): do _not_ enable Segmentation during initial configuration.** Gather labels and flows first, build/analyze policies in a workspace, enable enforcement on that workspace, then return to the connector to enable Segmentation for the VNet.

If/when you enable segmentation:
1. Confirm **Gather Labels** is enabled (required dependency)
2. Take your own backup of the NSG rules (CSW also auto-backs them up — see §9)
3. **Enable enforcement in the workspace _before_ enabling Segmentation for the VNet.** Otherwise **all traffic on that VNet is allowed**
4. Edit the connector and enable Segmentation for the target VNet — CSW takes ownership of the subnet/NIC NSGs

### Apply
Click **Test and Apply**.

---

## 8. AKS Integration

Enable Kubernetes integration for AKS clusters in the VNet:

1. Check **Enable Kubernetes / AKS** in the VNet configuration
2. CSW collects: pod IPs, namespaces, labels, service IPs, node labels
3. Metadata appears in CSW inventory:
   ```
   pod/name = payment-api-7d4b9f
   pod/namespace = production
   node/pool = nodepool-finance
   service/name = payment-svc
   ```

---

## 9. Segmentation with Azure NSGs

### How it works
1. CSW policies (label-based) are translated to NSG rules. Azure NSGs support both **Allow and Deny** rules
2. CSW sets the **subnet-level NSG to allow-all** and programs the **interface-level (NIC) NSGs** in the configured VNet (creating an interface NSG if none exists). Subnets with no NSG are not enforced
3. CSW continuously reconciles — adds/removes rules as policies or workloads change

### Ordering (important)
- **Enable enforcement on the workspace _first_.** Enabling Segmentation on a VNet not covered by an enforcement-enabled workspace results in **all traffic being allowed** on that VNet
- Set the workspace **Catch-All to Deny** to block everything not explicitly allowed

### Automatic backup and restore of NSGs
- **Automatic backup** — when you enable Segmentation for a VNet, CSW automatically backs up the associated NSGs (current state). Only one backup state is kept per VNet; restores use the **most recent** backup state
- **Automatic restore** — when you **disable** Segmentation, CSW restores the NSGs **it modified** to that most-recent backup; unrelated config is left untouched
- Toggle via **Manage > Workloads > Connectors > Azure Connector > Resources > Resources Tree**, then select the VNet and set the **Segmentation** radio button
- Still take an independent backup as defense-in-depth before the first enforcement test:
  ```bash
  az network nsg show --name <nsg-name> --resource-group <rg> > nsg-backup.json
  ```

### Policy example
```
Consumer: DevVMs     (scope: tag/Environment = Development)
Provider: ProdSQL    (scope: tag/Environment = Production AND tag/Tier = Database)
Action:   ALLOW TCP 1433   (only the flows you intend to permit)
Catch-All (scope default): DENY
```
CSW programs NSG rules permitting only the explicitly-allowed flows; everything else is dropped by the Catch-All — enforced natively by Azure.

---

## 10. Verification

### Check connector status
**Manage > Workloads > Connectors**, then select the Azure connector. Review the per-VNet rows and confirm a recent sync.

### Check inventory enrichment
1. On the Azure connector page, click a **VNet** row, then click an **IP address** to open its **Inventory Profile**
2. Confirm Azure tags appear as CSW labels

### Check enforcement / NSG rules (if segmentation enabled)
1. Check enforcement state under **Defend > Enforcement Status** (see *Enforcement Status for Cloud Connectors*)
2. In the Azure Portal → **Network Security Groups > [CSW-managed NSG]**, confirm CSW-programmed rules are present

---

## 11. Limits

| Metric | Limit |
|--------|-------|
| Subscriptions per connector | Multiple |
| Connectors per VNet | Exactly one — a VNet can belong to only one Azure connector |
| Flow log version | Version 2 required |
| Flow log protocols | TCP/UDP only — **no ICMP**; **no TCP flags** captured |
| Flow log retention | Minimum 2 days |
| Storage account key access | Must be enabled |
| Discovery scope | Only resources in the configured VNet |

---

## 12. Troubleshooting

| Symptom | Check |
|---------|-------|
| Authentication failure | Verify client ID, tenant ID, secret; confirm App Registration is active |
| No VNets discovered | Check subscription ID; verify ARM template applied with correct permissions |
| Flow logs not appearing | Confirm Version 2 flow logs; storage account key access enabled; storage accessible from CSW cluster |
| Segmentation not enforcing | Confirm Gather Labels enabled and the workspace is enforcing; verify ARM role has NSG write permissions; confirm the subnet has an NSG (subnets without an NSG are not enforced) |
| VNet unexpectedly allows all traffic | Segmentation was enabled on a VNet whose workspace is not enforcing, or the Catch-All policy is not set to **Deny** |
| Concrete policy shows **SKIPPED** | Generated rule count exceeds Azure NSG limits — consolidate policies (e.g., larger subnets) |
| AKS pods not visible | Confirm AKS RBAC allows CSW App Registration to read cluster resources |

---

## 13. Related Resources

| Repository | Description | Best for |
|------------|-------------|---------|
| [csw-aws-connector](https://github.com/chandrapati/csw-aws-connector) | AWS VPC label ingestion, Security Group enforcement | AWS workloads |
| [csw-gcp-connector](https://github.com/chandrapati/csw-gcp-connector) | GCP VPC label ingestion, firewall enforcement | GCP workloads |
| [CSW-Agent-Installation-Guide](https://github.com/chandrapati/CSW-Agent-Installation-Guide) | Deploy CSW agents inside Azure VMs for deep visibility | Agent-based visibility |
| [CSW-Policy-Lifecycle](https://github.com/chandrapati/CSW-Policy-Lifecycle) | Policy discovery → enforcement workflow | Policy management |
| [csw-splunk-integration](https://github.com/chandrapati/csw-splunk-integration) | CSW syslog alerts → Splunk | SecOps alerting |

---
*Community reference — Cisco Solutions Engineering. Not an official Cisco product document.*
