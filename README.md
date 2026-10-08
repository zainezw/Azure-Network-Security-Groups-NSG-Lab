# Azure Network Security Groups (NSG) Lab

Securing an Azure virtual machine with a Network Security Group so that remote desktop (RDP) access is allowed **only from a single trusted IP address**, while all other inbound traffic is denied by default.

Based on [Cloud Security Projects for Beginners – Project 2](https://github.com/0xrajneesh/Cloud-Security-Projects-For-Beginners/blob/main/Project-2-Implementing-Network-Security-Groups(NSGs)-on-Azure.md), built on an **Azure for Students** subscription with several hardening and cost-control changes beyond the original guide.

---

## Objectives

- Build an isolated virtual network and subnet
- Deploy a Windows Server VM with **no public ports open at creation**
- Create a custom NSG and write a least-privilege inbound rule
- Attach the NSG to the VM's network interface
- Control cloud spend and clean up all resources afterward

## Architecture

```
Internet
   │
   │  RDP 3389/TCP — allowed ONLY from <home-public-IP>/32
   ▼
┌─────────────────────────── myVNet 10.0.0.0/16 ───────────────────────────┐
│                                                                          │
│   ┌──────────────── mySubnet 10.0.1.0/24 ────────────────┐               │
│   │                                                      │               │
│   │   myNSG  ──attached to──►  myvm905 (NIC, 10.0.1.4)   │               │
│   │                                  │                   │               │
│   │                                myVM                  │               │
│   │                     Windows Server 2019 Datacenter   │               │
│   └──────────────────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────────────────┘
Region: South Africa North  •  Resource group: rg-nsg-lab
```

## Environment

| Item | Value |
|---|---|
| Subscription | Azure for Students ($100 credit, no credit card) |
| Resource group | `rg-nsg-lab` |
| Region | South Africa North |
| VNet / subnet | `myVNet` 10.0.0.0/16 / `mySubnet` 10.0.1.0/24 |
| VM | `myVM`, Windows Server 2019 Datacenter, Standard_DS1_v2 |
| NIC | `myvm905`, private IP 10.0.1.4 |
| NSG | `myNSG` |

---

## Lab resources

All resources deployed for the lab, viewed from the Azure portal home page.

![Lab resources](1.png)

## Pre-lab: Cost controls

Before deploying anything, I created a **$10 monthly budget** on the subscription with email alerts at 50% and 90% of actual spend and 100% of forecasted spend. Everything in the lab was placed in a single resource group (`rg-nsg-lab`) so the entire environment could be removed in one action.

## Exercise 1 – Virtual network and subnet

Created `myVNet` with address space `10.0.0.0/16` and a single subnet, `mySubnet`, at `10.0.1.0/24`. Azure Bastion, Azure Firewall, and DDoS Protection were left disabled, as they are paid services not needed for this lab.

![VNet overview](2.png)

## Exercise 2 – Virtual machine

Deployed `myVM` (Windows Server 2019 Datacenter) into `mySubnet`. Two settings were deliberately changed from the portal defaults:

- **Public inbound ports: None** — the wizard offers to open RDP to the entire internet. Leaving it closed means the VM is unreachable until a deliberate rule is written.
- **NIC network security group: None** — prevents Azure from auto-creating a second NSG, so `myNSG` is the only control governing traffic.

Auto-shutdown was enabled, and the disk, NIC, and public IP were set to delete with the VM.

![VM overview](3.png)

## Exercise 3 – Network security group

Created `myNSG` in the same region as the VM. A new NSG ships with default rules, the most important being **DenyAllInBound (priority 65500)**: until an allow rule is added, nothing from the internet can reach the VM.

![NSG overview with all rules](4.png)

## Exercise 4 – Security rules

### Inbound

| Priority | Name | Port | Protocol | Source | Destination | Action |
|---|---|---|---|---|---|---|
| **1000** | **Allow-RDP** | **3389** | **TCP** | **`<home-public-IP>/32`** | **Any** | **Allow** |
| 65000 | AllowVnetInBound | Any | Any | VirtualNetwork | VirtualNetwork | Allow |
| 65001 | AllowAzureLoadBalancerInBound | Any | Any | AzureLoadBalancer | Any | Allow |
| 65500 | DenyAllInBound | Any | Any | Any | Any | Deny |

NSG rules are evaluated from the lowest priority number upward, and evaluation stops at the first match. RDP from my IP matches rule 1000 and is allowed; traffic from any other source falls through to 65500 and is denied.

The `/32` suffix scopes the rule to exactly one address.

![Allow-RDP rule details](5.png)

### Outbound

Left at defaults. `AllowInternetOutBound` (65001) lets the VM reach the internet for updates and activation.

![Outbound rules](6.png)

## Exercise 5 – Associate the NSG with the NIC

Associated `myNSG` with network interface `myvm905`. The NSG overview confirms **1 inbound custom rule** and **association with 1 network interface**.

![NSG associated with NIC](7.png)

## Cleanup

Deleted the `rg-nsg-lab` resource group, which removed all 7 resources (VM, OS disk, NIC, public IP, VNet, NSG, and auto-shutdown schedule) in a single operation.

![Delete resource group](8.png)
![Deletion confirmed](9.png)

---

## Challenges and how I solved them

**VM size not available in East US.** The default size and B-series sizes returned `NotAvailableForSubscription`. Azure for Students subscriptions have regional capacity restrictions. I moved the deployment to
