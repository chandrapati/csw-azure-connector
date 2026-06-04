# Cisco Secure Workload — Azure Connector Guide

> **Disclaimer:** Community reference guide by Cisco Solutions Engineering. Always consult [official Cisco Secure Workload documentation](https://www.cisco.com/c/en/us/products/security/tetration/index.html) for authoritative guidance.

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

> **No virtual appliance required.** The Azure connector runs as a service within the CSW cluster.

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
- Requires Azure VNet flow logs (Version 2) enabled and stored in an **Azure Storage Account**
- Retention: minimum 2 days (connector pulls every minute)
- Storage account must have **storage account key access enabled**

### 3.3 Segmentation
- CSW policies → Azure NSG rules applied to subnets and NICs
- **Warning:** All existing NSG rules are removed when enforcement is enabled. Back up before enabling.

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

1. CSW UI: **Manage > Connectors > Cloud > + Add Connector > Azure**
2. Select capabilities to enable

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

If enabling:
1. Back up current NSG rules first
2. Enable Labels (required)
3. Enable Segmentation — CSW takes ownership of NSG rules on the VNet subnets/NICs

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
1. CSW policies (label-based) are translated to NSG rules
2. NSG rules applied to subnets and NICs in the configured VNet
3. CSW continuously reconciles — adds/removes rules as policies or workloads change

### Policy example
```
Consumer: DevVMs     (scope: tag/Environment = Development)
Provider: ProdSQL    (scope: tag/Environment = Production AND tag/Tier = Database)
Action:   DENY ALL
```
CSW creates NSG inbound rule denying DevVMs from ProdSQL — enforced natively by Azure.

### Warning
- **All existing NSG rules are removed** from VNet-associated NSGs when enforcement is enabled
- Export and save NSG rules before enabling enforcement:
  ```bash
  az network nsg show --name <nsg-name> --resource-group <rg> > nsg-backup.json
  ```

---

## 10. Verification

### Check connector status
**Manage > Connectors > Cloud > [Azure Connector]**
Status should show **Active** with last sync time.

### Check inventory enrichment
1. **Inventory > Workloads** → search by Azure VM IP
2. Confirm Azure tags appear as CSW labels

### Check NSG rules (if segmentation enabled)
Azure Portal → **Network Security Groups > [CSW-managed NSG]**
Confirm CSW-programmed rules are present.

---

## 11. Limits

| Metric | Limit |
|--------|-------|
| Subscriptions per connector | Multiple |
| VNets per cluster | One connector per VNet |
| Flow log version | Version 2 required |
| Flow log retention | Minimum 2 days |
| Storage account key access | Must be enabled |

---

## 12. Troubleshooting

| Symptom | Check |
|---------|-------|
| Authentication failure | Verify client ID, tenant ID, secret; confirm App Registration is active |
| No VNets discovered | Check subscription ID; verify ARM template applied with correct permissions |
| Flow logs not appearing | Confirm Version 2 flow logs; storage account key access enabled; storage accessible from CSW cluster |
| Segmentation not enforcing | Confirm Labels enabled; verify ARM role has NSG write permissions |
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
