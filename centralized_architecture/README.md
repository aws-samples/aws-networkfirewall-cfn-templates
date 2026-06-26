# Centralized Architecture - Inspection VPC Model

The centralized deployment model uses [AWS Transit Gateway](https://aws.amazon.com/transit-gateway/) as a network hub to simplify connectivity between VPCs and on-premises networks. All templates in this folder use Transit Gateway to route traffic through centralized inspection points.

> **Looking for TGW-native attached firewall?** See the [Transit Gateway-Attached Firewall](../transit-gateway-attached-firewall/) templates for the recommended egress/east-west deployment model using the native TGW firewall attachment (no inspection VPC required).

## Available Templates

```
centralized_architecture/
├── centralized_egress_east_west/                # Egress + east-west inspection (inspection VPC model)
│   ├── single_az/
│   └── two_az/
└── centralized_ingress_egress_east_west/        # Ingress + egress + east-west (dual firewall)
    ├── single_az/
    └── two_az/
```

---

## [Centralized Egress + East-West](centralized_egress_east_west/)

Centralized inspection for both east-west (VPC-to-VPC) and egress (internet-bound) traffic through a dedicated inspection VPC containing the firewall endpoints.

### [Single AZ Deployment](centralized_egress_east_west/single_az/)
- **Template:** `anfw-centralized-egress-east-west-1az-template.yaml`
- **Use Case:** Testing and proof-of-concept environments
- **Resources:** All components deployed in a single Availability Zone

### [Two AZ Deployment](centralized_egress_east_west/two_az/)
- **Template:** `anfw-centralized-egress-east-west-2az-template.yaml`
- **Use Case:** Production environments requiring high availability
- **Resources:** Components distributed across two Availability Zones

### Architecture Components
**Inspection VPC** - Centralized VPC for both east-west and egress traffic inspection
- Transit Gateway subnet
- Firewall subnet (contains AWS Network Firewall endpoints)
- Public subnet (contains NAT Gateway(s) for internet access)

**Spoke VPCs** - Example workload VPCs that route traffic through the inspection VPC

---

## [Centralized Ingress + Egress + East-West](centralized_ingress_egress_east_west/)

Dual-firewall architecture for full centralized network inspection: a VPC-attached ingress firewall inspects all inbound traffic to the centralized NLB (showcasing SSH/SFTP filtering as a non-web protocol use case), and a TGW-native egress/east-west firewall inspects all outbound and spoke-to-spoke traffic with visibility into true source IPs before NAT.

### [Single AZ Deployment](centralized_ingress_egress_east_west/single_az/)
- **Template:** `anfw-centralized-ingress-egress-east-west-1az-template.yaml`
- **Use Case:** Testing, development, and proof-of-concept environments
- **Resources:** All components deployed in a single Availability Zone

### [Two AZ Deployment](centralized_ingress_egress_east_west/two_az/)
- **Template:** `anfw-centralized-ingress-egress-east-west-2az-template.yaml`
- **Use Case:** Production environments requiring high availability
- **Resources:** Components distributed across two Availability Zones

### Architecture Components

**Ingress VPC** - Hosts inbound traffic inspection
- Firewall subnet (VPC-attached Network Firewall endpoint per AZ)
- Public subnet (internet-facing NLB)
- TGW subnet (Transit Gateway attachment)
- IGW with ingress route table steering traffic through firewall

**Egress VPC** - Dedicated VPC for outbound internet access
- Public subnet (NAT Gateway per AZ, IGW)
- TGW subnet (Transit Gateway attachment)

**Egress/East-West Network Firewall** - TGW-native attached
- Inspects all egress and spoke-to-spoke traffic
- Sees true source IPs before NAT translation

**Spoke VPCs** - Example workload VPCs with static EC2 IPs targeted directly by the NLB

---

## Additional Resources

For detailed information about AWS Network Firewall deployment models, refer to the [AWS Blog: Deployment models for AWS Network Firewall](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/).
