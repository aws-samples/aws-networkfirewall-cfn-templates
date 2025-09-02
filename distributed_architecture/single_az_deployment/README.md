# Distributed Architecture - Single AZ Deployment

This folder contains single Availability Zone distributed architecture templates for AWS Network Firewall. These templates are designed for testing, development, and proof-of-concept environments.

## Available Templates

### [Combined Ingress and Egress Firewall](combined-ingress-and-egress-firewall/)
- **Template:** `anfw-distributed-1az-single-firewall-template.yaml`
- **Configuration:** Single firewall handles both inbound and outbound traffic
- **Use Case:** Applications with resources that have public IP addresses

### [Separate Ingress and Egress Firewall](separate-ingress-and-egress-firewall/)
- **Template:** `anfw-distributed-1az-dual-firewall-template.yaml`
- **Configuration:** Dedicated firewalls for inbound and outbound traffic
- **Use Case:** Public load balancer applications with workloads in private subnets - maintain full source IP visibility

## Architecture Overview

The distributed model deploys AWS Network Firewall directly within each VPC that requires protection. This approach provides:

- **VPC-level Protection** - Each VPC protected independently
- **Simplified Networking** - No Transit Gateway required
- **Isolated Management** - Each firewall managed separately
- **Reduced Complexity** - Direct traffic flow within each VPC

## Single AZ Considerations

- **Testing Purpose** - Designed for development and testing environments
- **No High Availability** - Single point of failure if AZ becomes unavailable
- **Cost Optimized** - Lower cost due to single AZ deployment
- **Limited Resilience** - Not suitable for production workloads

## Production Alternative

For environments requiring high availability, consider the [Two AZ Deployment](../two_az_deployment/) which provides high availability and fault tolerance across multiple Availability Zones.

## Template Selection Guide

Choose **Combined Firewall** if:
- Resources can have public IP addresses
- Simple testing scenarios without NAT Gateway
- Cost optimization is priority
- Single policy for all traffic

Choose **Separate Firewalls** if:
- Resources need to remain private (uses NAT Gateway) fronted with public load balancer
- Full source IP visibility to firewall is required for both directions
- Seperate policies for ingress and egress traffic
- NAT Gateway needed for private subnet internet access

## Additional Resources

- [AWS Network Firewall Documentation](https://docs.aws.amazon.com/network-firewall/)
- [Deployment models for AWS Network Firewall Blog](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/)
