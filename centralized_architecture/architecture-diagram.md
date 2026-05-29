# Centralized Ingress + Egress/East-West Architecture Diagram

## Message for Daniel

Hey Daniel, heads up - after I reviewed the original architecture and told you it was good, I ended up syncing with Anvesh and got some more networking SSA insights. Based on that conversation I made some significant changes to the architecture. The key shift is splitting into two separate firewalls (ingress VPC-attached, egress TGW-native) and removing the spoke NLBs. Here's the full updated architecture for you to recreate in Lucid.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                        INTERNET                                          │
└─────────────────────────────┬───────────────────────────────────┬───────────────────────┘
                              │ Inbound (SSH/SFTP)                 │ Outbound (egress return)
                              ▼                                    ▼
┌─────────────────────────────────────────────┐    ┌─────────────────────────────────────┐
│         INGRESS VPC (100.64.0.0/16)         │    │       EGRESS VPC (10.100.0.0/16)    │
│                                             │    │                                     │
│  ┌───────────────────────────────────────┐  │    │  ┌─────────────────────────────┐    │
│  │  Internet Gateway                     │  │    │  │  Internet Gateway            │    │
│  └──────────────────┬────────────────────┘  │    │  └──────────────┬──────────────┘    │
│                     │                        │    │                 │                    │
│  ┌──────────────────▼────────────────────┐  │    │  ┌──────────────▼──────────────┐    │
│  │  IGW Ingress Route Table              │  │    │  │  Public Subnet              │    │
│  │  100.64.1.0/24 → FW Endpoint AZ1     │  │    │  │  0.0.0.0/0 → IGW            │    │
│  │  100.64.2.0/24 → FW Endpoint AZ2     │  │    │  │  10.0.0.0/8 → TGW           │    │
│  └──────────────────┬────────────────────┘  │    │  │                              │    │
│                     │                        │    │  │  ┌────────┐  ┌────────┐     │    │
│  ┌──────────────────▼────────────────────┐  │    │  │  │NAT GW  │  │NAT GW  │     │    │
│  │  Firewall Subnet(s)                   │  │    │  │  │  AZ1   │  │  AZ2   │     │    │
│  │  ┌──────────────────────────────────┐ │  │    │  │  └────┬───┘  └────┬───┘     │    │
│  │  │  INGRESS NETWORK FIREWALL       │ │  │    │  └───────┼───────────┼──────────┘    │
│  │  │  (VPC-attached)                  │ │  │    │          │           │               │
│  │  │  - Log-only rules (default)     │ │  │    │  ┌───────▼───────────▼──────────┐    │
│  │  │  - Alerts on SSH, TLS, ICMP     │ │  │    │  │  TGW Subnet(s)              │    │
│  │  │  - Endpoint per AZ              │ │  │    │  │  0.0.0.0/0 → NAT GW (per AZ)│    │
│  │  └──────────────────────────────────┘ │  │    │  └──────────────┬──────────────┘    │
│  │  Route Table:                         │  │    │                 │                    │
│  │  0.0.0.0/0 → IGW                     │  │    └─────────────────┼────────────────────┘
│  └──────────────────┬────────────────────┘  │                      │
│                     │ (local route)          │                      │
│  ┌──────────────────▼────────────────────┐  │                      │
│  │  Public Subnet(s) - NLB lives here    │  │                      │
│  │  ┌──────────────────────────────────┐ │  │                      │
│  │  │  INTERNET-FACING NLB            │ │  │                      │
│  │  │  Port 22  → 10.1.1.10:22 (SSH)  │ │  │                      │
│  │  │  Port 2222 → 10.2.1.10:22 (SFTP)│ │  │                      │
│  │  │  Security Group: AllowedSourceIP │ │  │                      │
│  │  └──────────────────────────────────┘ │  │                      │
│  │  Route Table:                         │  │                      │
│  │  0.0.0.0/0 → FW Endpoint (per AZ)    │  │                      │
│  │  10.0.0.0/8 → TGW                    │  │                      │
│  └──────────────────┬────────────────────┘  │                      │
│                     │                        │                      │
│  ┌──────────────────▼────────────────────┐  │                      │
│  │  TGW Subnet(s) - local routes only    │  │                      │
│  └──────────────────┬────────────────────┘  │                      │
│                     │                        │                      │
└─────────────────────┼────────────────────────┘                      │
                      │                                                │
                      │ TGW Attachment                    TGW Attachment│
                      │ (ApplianceMode=enable)                         │
                      │                                                │
┌─────────────────────┼────────────────────────────────────────────────┼──────────────────┐
│                     │          TRANSIT GATEWAY                        │                   │
│                     │                                                │                   │
│  ┌──────────────────▼─────────────────────────────────────────────────────────────────┐ │
│  │                         TGW ROUTE TABLES                                            │ │
│  │                                                                                     │ │
│  │  ┌─────────────────────────┐  ┌─────────────────────────────────────────────────┐  │ │
│  │  │  SPOKE RT               │  │  INGRESS VPC RT                                 │  │ │
│  │  │  (Spoke A & B attached) │  │  (Ingress VPC attached)                         │  │ │
│  │  │                         │  │                                                  │  │ │
│  │  │  0.0.0.0/0 → Egress FW │  │  10.1.0.0/16 → Spoke A (propagated)             │  │ │
│  │  │  100.64.0.0/16 → Ingress│  │  10.2.0.0/16 → Spoke B (propagated)             │  │ │
│  │  │  VPC                    │  │                                                  │  │ │
│  │  └─────────────────────────┘  └─────────────────────────────────────────────────┘  │ │
│  │                                                                                     │ │
│  │  ┌─────────────────────────────────┐  ┌─────────────────────────────────────────┐  │ │
│  │  │  EGRESS INSPECTION RT           │  │  EGRESS VPC RT                          │  │ │
│  │  │  (Egress FW attached)           │  │  (Egress VPC attached)                  │  │ │
│  │  │                                 │  │                                          │  │ │
│  │  │  0.0.0.0/0 → Egress VPC        │  │  0.0.0.0/0 → Egress FW                  │  │ │
│  │  │  10.1.0.0/16 → Spoke A (prop.) │  │  (symmetric return inspection)           │  │ │
│  │  │  10.2.0.0/16 → Spoke B (prop.) │  │                                          │  │ │
│  │  └─────────────────────────────────┘  └─────────────────────────────────────────┘  │ │
│  └─────────────────────────────────────────────────────────────────────────────────────┘ │
│                     │                                                                    │
│  ┌──────────────────▼────────────────────────────────────────────┐                       │
│  │  EGRESS/EAST-WEST NETWORK FIREWALL (TGW-native attached)     │                       │
│  │  - Log-only rules (default)                                   │                       │
│  │  - Alerts on TLS (with SNI), HTTP, SSH, ICMP                  │                       │
│  │  - Geo-alerting (RU, CN)                                      │                       │
│  │  - High-risk TLD alerting (.ru, .xyz, .info, .onion)          │                       │
│  │  - Sees TRUE SOURCE IP (before NAT)                           │                       │
│  │  - Endpoint per AZ                                            │                       │
│  └───────────────────────────────────────────────────────────────┘                       │
│                     │                              │                                     │
└─────────────────────┼──────────────────────────────┼─────────────────────────────────────┘
                      │                              │
          TGW Attachment                  TGW Attachment
                      │                              │
┌─────────────────────▼──────────────┐ ┌─────────────▼────────────────────────┐
│   SPOKE VPC A (10.1.0.0/16)        │ │   SPOKE VPC B (10.2.0.0/16)          │
│   SSH Bastion - AZ1                │ │   SFTP Server - AZ2                  │
│                                    │ │                                       │
│  ┌──────────────────────────────┐  │ │  ┌─────────────────────────────────┐ │
│  │  TGW Subnet(s)              │  │ │  │  TGW Subnet(s)                  │ │
│  │  (local routes only)        │  │ │  │  (local routes only)            │ │
│  └──────────────┬───────────────┘  │ │  └──────────────┬──────────────────┘ │
│                 │                   │ │                  │                    │
│  ┌──────────────▼───────────────┐  │ │  ┌──────────────▼──────────────────┐ │
│  │  Workload Subnet             │  │ │  │  Workload Subnet                │ │
│  │  10.1.1.0/24 (AZ1)          │  │ │  │  10.2.1.0/24 (AZ2)             │ │
│  │  Route: 0.0.0.0/0 → TGW     │  │ │  │  Route: 0.0.0.0/0 → TGW        │ │
│  │                              │  │ │  │                                  │ │
│  │  ┌────────────────────────┐  │  │ │  │  ┌──────────────────────────┐   │ │
│  │  │  EC2: 10.1.1.10       │  │  │ │  │  │  EC2: 10.2.1.10          │   │ │
│  │  │  SSH Bastion           │  │  │ │  │  │  SFTP Server             │   │ │
│  │  │  User: demouser       │  │  │ │  │  │  User: sftpuser          │   │ │
│  │  │  Password: from       │  │  │ │  │  │  Password: from          │   │ │
│  │  │  Secrets Manager      │  │  │ │  │  │  Secrets Manager         │   │ │
│  │  └────────────────────────┘  │  │ │  │  └──────────────────────────┘   │ │
│  │                              │  │ │  │                                  │ │
│  │  VPC Endpoints: SSM,        │  │ │  │  VPC Endpoints: SSM,             │ │
│  │  EC2Messages, SSMMessages   │  │ │  │  EC2Messages, SSMMessages        │ │
│  └──────────────────────────────┘  │ │  └──────────────────────────────────┘ │
└────────────────────────────────────┘ └───────────────────────────────────────┘
```

---

## Traffic Flows

### Ingress (SSH - Port 22 → Spoke A)

```
Internet
  │
  ▼
IGW (Ingress VPC)
  │
  │  IGW Ingress RT: 100.64.1.0/24 → Firewall Endpoint
  ▼
INGRESS NETWORK FIREWALL (inspects inbound SSH)
  │
  │  Local route within VPC
  ▼
NLB (Public Subnet, port 22)
  │
  │  Public Subnet RT: 10.0.0.0/8 → TGW
  ▼
TGW (Ingress VPC RT → 10.1.0.0/16 propagated → Spoke A)
  │
  ▼
EC2 10.1.1.10 (SSH Bastion, Spoke A)
```

### Ingress (SFTP - Port 2222 → Spoke B)

```
Internet
  │
  ▼
IGW (Ingress VPC)
  │
  │  IGW Ingress RT: 100.64.x.0/24 → Firewall Endpoint
  ▼
INGRESS NETWORK FIREWALL (inspects inbound SFTP)
  │
  │  Local route within VPC
  ▼
NLB (Public Subnet, port 2222 → target 10.2.1.10:22)
  │
  │  Public Subnet RT: 10.0.0.0/8 → TGW
  ▼
TGW (Ingress VPC RT → 10.2.0.0/16 propagated → Spoke B)
  │
  ▼
EC2 10.2.1.10 (SFTP Server, Spoke B)
```

### Ingress Return (symmetric through firewall)

```
EC2 (Spoke A or B)
  │
  │  Workload RT: 0.0.0.0/0 → TGW
  ▼
TGW (Spoke RT: 100.64.0.0/16 → Ingress VPC)
  │
  ▼
NLB (Ingress VPC, Public Subnet)
  │
  │  Public Subnet RT: 0.0.0.0/0 → Firewall Endpoint
  ▼
INGRESS NETWORK FIREWALL (symmetric - sees return traffic)
  │
  │  Firewall Subnet RT: 0.0.0.0/0 → IGW
  ▼
IGW → Internet
```

### Egress (sees true source IP)

```
EC2 (e.g., 10.1.1.10)
  │
  │  Workload RT: 0.0.0.0/0 → TGW
  ▼
TGW (Spoke RT: 0.0.0.0/0 → Egress Firewall)
  │
  ▼
EGRESS/EAST-WEST FIREWALL (TGW-native)
  │  *** Sees src_ip=10.1.1.10 (true source, pre-NAT) ***
  │
  │  Inspection RT: 0.0.0.0/0 → Egress VPC
  ▼
TGW → Egress VPC TGW Subnet
  │
  │  TGW Subnet RT: 0.0.0.0/0 → NAT GW
  ▼
NAT Gateway (translates to public EIP)
  │
  │  Public Subnet RT: 0.0.0.0/0 → IGW
  ▼
IGW → Internet
```

### Egress Return (symmetric through firewall)

```
Internet
  │
  ▼
IGW (Egress VPC)
  │
  ▼
NAT Gateway (de-NATs back to 10.x.x.x)
  │
  │  Public Subnet RT: 10.0.0.0/8 → TGW
  ▼
TGW (Egress VPC RT: 0.0.0.0/0 → Egress Firewall)
  │
  ▼
EGRESS/EAST-WEST FIREWALL (symmetric return inspection)
  │
  │  Inspection RT: 10.1.0.0/16 → Spoke A (propagated)
  ▼
TGW → Spoke EC2
```

### East-West (Spoke A → Spoke B, inspected)

```
EC2 10.1.1.10 (Spoke A)
  │
  │  Workload RT: 0.0.0.0/0 → TGW
  ▼
TGW (Spoke RT: 0.0.0.0/0 → Egress Firewall)
  │
  ▼
EGRESS/EAST-WEST FIREWALL (inspects lateral traffic)
  │
  │  Inspection RT: 10.2.0.0/16 → Spoke B (propagated)
  ▼
TGW → EC2 10.2.1.10 (Spoke B)
```

---

## Key Design Decisions

1. **Two separate firewalls** - Ingress (VPC-attached) handles inbound inspection with IGW routing trick. Egress (TGW-native) handles outbound/east-west and sees true source IPs before NAT translation.

2. **SSH/SFTP instead of HTTP** - NFW is positioned for non-web protocol ingress filtering. For HTTP/HTTPS, AWS WAF on ALB/CloudFront/API Gateway is the recommended approach.

3. **No spoke NLBs** - EC2 instances have static private IPs (10.1.1.10, 10.2.1.10). The central ingress NLB targets them directly via TGW (IP-type targets, AvailabilityZone: "all").

4. **Port-based routing on central NLB** - Port 22 routes to Spoke A (SSH), Port 2222 routes to Spoke B (SFTP on port 22).

5. **IP-restricted access** - NLB security group only allows the deployer's specific IP (/32 CIDR, required parameter) to avoid Riddler findings for open SSH.

6. **Password auth via Secrets Manager** - Random 24-char password generated at deploy time, stored in Secrets Manager. No key pair required to lower the barrier to entry.

7. **Egress VPC is separate from Ingress VPC** - Clean separation of concerns. Ingress VPC has no NAT GW. Egress VPC has no ingress firewall.

---

## CIDR Allocations

| VPC | CIDR | Purpose |
|-----|------|---------|
| Ingress VPC | 100.64.0.0/16 | Central ingress inspection |
| Egress VPC | 10.100.0.0/16 | NAT/egress to internet |
| Spoke A | 10.1.0.0/16 | SSH Bastion |
| Spoke B | 10.2.0.0/16 | SFTP Server |

### Ingress VPC Subnets (per AZ)

| Subnet | AZ1 CIDR | AZ2 CIDR | Purpose |
|--------|----------|----------|---------|
| Firewall | 100.64.0.16/28 | 100.64.0.32/28 | NFW endpoints |
| Public | 100.64.1.0/24 | 100.64.2.0/24 | NLB |
| TGW | 100.64.0.0/28 | 100.64.0.48/28 | TGW attachment |

### Egress VPC Subnets (per AZ)

| Subnet | AZ1 CIDR | AZ2 CIDR | Purpose |
|--------|----------|----------|---------|
| TGW | 10.100.0.0/28 | 10.100.0.16/28 | TGW attachment |
| Public | 10.100.1.0/24 | 10.100.2.0/24 | NAT GW + IGW |

### Spoke Subnets

| Subnet | CIDR | AZ | Purpose |
|--------|------|----|---------|
| Spoke A Workload | 10.1.1.0/24 | AZ1 | EC2 (10.1.1.10) |
| Spoke A TGW | 10.1.0.0/28 | AZ1 | TGW attachment |
| Spoke A TGW | 10.1.0.16/28 | AZ2 | TGW attachment |
| Spoke B Workload | 10.2.1.0/24 | AZ2 | EC2 (10.2.1.10) |
| Spoke B TGW | 10.2.0.0/28 | AZ1 | TGW attachment |
| Spoke B TGW | 10.2.0.16/28 | AZ2 | TGW attachment |
