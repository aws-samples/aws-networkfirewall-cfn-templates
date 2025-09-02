# Centralized Architecture

The centralized deployment model uses [AWS Transit Gateway](https://aws.amazon.com/transit-gateway/) as a network hub to simplify connectivity between VPCs and on-premises networks. This architecture provides centralized inspection for both East-West (VPC to VPC) and egress (internet-bound) traffic through a dedicated inspection VPC.

## Available Templates

This folder contains two CloudFormation templates that deploy the same centralized architecture pattern:

### [Single AZ Deployment](single_az_deployment/)
- **Template:** `anfw-centralized-1az-template.yaml`
- **Use Case:** Testing and proof-of-concept environments
- **Resources:** All components deployed in a single Availability Zone

### [Two AZ Deployment](two_az_deployment/)
- **Template:** `anfw-centralized-2az-template.yaml`
- **Use Case:** Production environments requiring high availability
- **Resources:** Components distributed across two Availability Zones

## Architecture Components

Both templates create:

**Inspection VPC** - Centralized VPC for both East-West and Egress traffic inspection
- Transit Gateway subnet
- Firewall subnet (contains AWS Network Firewall endpoints)
- Public subnet (contains NAT Gateway(s) for Internet access)

**Spoke VPCs** - Example workload VPCs that route traffic through the inspection VPC


## Additional Resources

For detailed information about AWS Network Firewall deployment models, refer to the [AWS Blog: Deployment models for AWS Network Firewall](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/).
