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

### Centralized Ingress Inspection

Templates that add ingress traffic inspection to the centralized model. All inbound internet traffic is routed through AWS Network Firewall in an Edge VPC before reaching spoke VPC workloads via Transit Gateway. TLS is terminated at the Edge load balancer; downstream traffic to spoke NLBs and EC2 instances uses plaintext HTTP.

#### [Single AZ Ingress Deployment](centralized_ingress_single_az/)
- **Template:** `anfw-centralized-ingress-1az-template.yaml`
- **Use Case:** Testing and proof-of-concept environments
- **Edge LB:** Internet-facing NLB with TLS termination (ALBs require 2 AZs)

#### [Two AZ Ingress Deployment](centralized_ingress_two_az/)
- **Template:** `anfw-centralized-ingress-2az-template.yaml`
- **Use Case:** Production environments requiring high availability
- **Edge LB:** Internet-facing ALB with HTTPS termination across two AZs


## Additional Resources

For detailed information about AWS Network Firewall deployment models, refer to the [AWS Blog: Deployment models for AWS Network Firewall](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/).
