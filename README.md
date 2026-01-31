# 🌐 AWS ↔ GCP Hybrid Cloud Connectivity using Site-to-Site VPN (HA + BGP)

This project demonstrates **enterprise-grade hybrid cloud networking** by establishing a **secure Site-to-Site VPN** between **Amazon Web Services (AWS)** and **Google Cloud Platform (GCP)** using **High Availability (HA) VPN** and **dynamic routing via BGP**.

The objective is to allow **private resources in AWS and GCP to communicate securely over private IPs**, as if both clouds were part of the same internal network.

---

## 📌 Why This Project?

Hybrid connectivity is a **core real-world cloud networking requirement** used in:

- Multi-cloud architectures  
- Cloud migration scenarios  
- Disaster recovery setups  
- Regulatory or data residency requirements  

This project helps you understand:

- How AWS and GCP VPN components map to each other
- Why BGP is required for scalable routing
- How routing and firewall rules affect traffic flow
- How to troubleshoot VPNs in real production environments

---

## 🏗️ Final Architecture Overview

+-------------------+ +-------------------+
| GCP VPC | | AWS VPC |
| 21.0.0.0/16 | | 10.0.0.0/16 |
| | | |
| GCP VM | | EC2 Instance |
| 21.0.1.10 | | 10.0.2.10 |
| | | |
+--------+----------+ +---------+---------+
| |
| HA VPN + BGP |
| (2 Interfaces, 4 Tunnels) |
| |
+--------+----------+ +---------+---------+
| Cloud Router | | Virtual Private |
| ASN: 65001 | | Gateway (VGW) |
| | | ASN: 64512 |
+-------------------+ +-------------------+




---

## 🧠 Key Concepts Used

| Concept | Why It Is Required |
|------|------------------|
| Site-to-Site VPN | Secure private connectivity between clouds |
| HA VPN (GCP) | 99.99% SLA and redundancy |
| BGP | Dynamic route exchange between AWS & GCP |
| ASN | Identifies routing devices in BGP |
| VGW (AWS) | Terminates VPN traffic on AWS |
| Cloud Router (GCP) | Manages BGP sessions |
| Route Propagation | Allows AWS to learn GCP routes dynamically |

---

## 📍 IP Addressing Plan

### AWS
| Resource | CIDR |
|------|------|
| VPC | 10.0.0.0/16 |
| Public Subnet | 10.0.1.0/24 |
| Private Subnet | 10.0.2.0/24 |

### GCP
| Resource | CIDR |
|------|------|
| VPC | 21.0.0.0/16 |
| Subnet | 21.0.1.0/24 |

---

## 🔐 Phase 1: Prerequisites & Permissions

### AWS IAM
Required permissions:
- `AmazonVPCFullAccess`
- `AmazonEC2FullAccess`
- `AWSVPNFullAccess`

### GCP IAM
Required roles:
- `Compute Admin`
- `Network Admin`

---

## ☁️ Phase 2: AWS Infrastructure Setup

### 1️⃣ Create VPC
**Why:** Logical network boundary in AWS.

- CIDR: `10.0.0.0/16`

---

### 2️⃣ Create Subnets
**Why:** Separate public access from private workloads.

- Public Subnet: `10.0.1.0/24`
- Private Subnet: `10.0.2.0/24`

---

### 3️⃣ Internet Gateway (IGW)
**Why:** Required for internet access in AWS.

- Attach IGW to AWS VPC

---

### 4️⃣ Route Table
**Why:** Controls traffic flow.

- Route: `0.0.0.0/0 → IGW`
- Associate with Public Subnet

---

## 🌐 Phase 3: GCP Infrastructure Setup

### 1️⃣ Create VPC
**Why:** Private network in GCP.

- Custom mode VPC
- Subnet: `21.0.1.0/24`

---

### 2️⃣ Firewall Rule
**Why:** GCP blocks traffic by default.

Allow:
- ICMP
- SSH

Source:
- `10.0.0.0/16` (AWS CIDR)

---

## 🔁 Phase 4: GCP HA VPN Setup

### 1️⃣ Cloud Router
**Why:** Required for BGP.

- ASN: `65001`
- Attached to GCP VPC

---

### 2️⃣ HA VPN Gateway
**Why:** Provides redundant VPN interfaces.

- Generates **2 public IPs**
- Each interface supports 2 tunnels

---

## 🔗 Phase 5: AWS VPN Configuration

### 1️⃣ Customer Gateways (CGW)
**Why:** Represents GCP on AWS side.

- Create **2 CGWs**
- Use GCP HA VPN public IPs
- ASN: `65001`

---

### 2️⃣ Virtual Private Gateway (VGW)
**Why:** VPN termination point in AWS.

- Attach to AWS VPC
- AWS ASN: `64512`

---

### 3️⃣ Site-to-Site VPN Connections
**Why:** Establish encrypted tunnels.

- Create **2 VPN connections**
- Routing: **Dynamic (BGP)**

Download configuration files:
- Contains PSK
- Inside IPs
- Tunnel IPs

---

## 🔄 Phase 6: Tunnel & BGP Configuration (GCP)

### VPN Tunnels
**Why:** Actual encrypted communication path.

- Total tunnels: **4**
- Peer IP: AWS tunnel IP
- Pre-Shared Key: From AWS config

---

### BGP Sessions
**Why:** Exchange routes automatically.

- Peer ASN: `64512`
- Inside IPs: From AWS config

---

## 🚦 Phase 7: Route Propagation (CRITICAL)

### AWS Route Table
**Why:** Without this, traffic will NOT flow even if VPN is UP.

Enable:
- Route Propagation
- Select VGW

Expected learned route:
21.0.0.0/16 → VGW


---

## 🧪 Phase 8: Verification

### Create Test Machines
- AWS EC2 in **private subnet** (no public IP)
- GCP VM in subnet `21.0.1.0/24`

---

### Connectivity Test

SSH into GCP VM:
```
gcloud compute ssh gcp-vm --zone us-central1-a
```
Ping AWS EC2 private IP:
```
ping 10.0.2.x
```
✅ Successful ping confirms hybrid connectivity.

❗ Common Troubleshooting Points

| Issue                 | Root Cause                     |
| --------------------- | ------------------------------ |
| VPN UP but ping fails | Route propagation disabled     |
| No response           | AWS Security Group blocks ICMP |
| No GCP route          | BGP not established            |
| Tunnel DOWN           | PSK / ASN mismatch             |

💰 Cost Awareness

Services that incur cost:

AWS Site-to-Site VPN

GCP HA VPN

Compute instances

⚠️ Always delete resources after testing.

🧹 Cleanup

Delete in correct order:

- VPN Tunnels
- VPN Gateways
- Cloud Router / VGW
- EC2 / VM
- Subnets
- VPCs
