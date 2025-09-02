# Distributed Architecture - Two AZ Deployment

This folder contains multi-Availability Zone distributed architecture templates for AWS Network Firewall. These templates are designed for environments requiring high availability and fault tolerance.

*Insert architecture diagram link here*

## Available Templates

### [Combined Ingress and Egress Firewall](combined-ingress-and-egress-firewall/)
- **Template:** `anfw-distributed-2az-single-firewall-template.yaml`
- **Configuration:** Single firewall endpoint per AZ handles both inbound and outbound traffic
- **Use Case:** Applications with resources that have public IP addresses

### [Separate Ingress and Egress Firewall](separate-ingress-and-egress-firewall/)
- **Template:** `anfw-distibuted-2az-dual-firewall-template.yaml`
- **Configuration:** Dedicated firewall endpoints for inbound and outbound traffic (2 endpoints per AZ)
- **Use Case:** Public load balancer applications with workloads in private subnets - maintain full source IP visibility


## Architecture Overview

The distributed model deploys AWS Network Firewall directly within each VPC across two Availability Zones. This approach provides:

- **High Availability** - Resources distributed across multiple AZs
- **VPC-level Protection** - Each VPC protected independently
- **Fault Tolerance** - Continues operating if one AZ becomes unavailable
- **Simplified Networking** - No Transit Gateway required
- **AZ Affinity** - Traffic processed within the same AZ when possible

## Multi-AZ Benefits

- **High Availability** - Suitable for environments requiring high uptime
- **Redundant Resources** - NAT Gateways and firewall endpoints in each AZ
- **Performance** - AZ-aware routing minimizes latency
- **Compliance** - Meets availability requirements for critical workloads

## Template Selection Guide

Choose **Combined Firewall** if:
- Resources can have public IP addresses
- Simplified firewall management preferred
- Single policy for all traffic types
- Cost optimization while maintaining availability (no NAT Gateway)

Choose **Separate Firewalls** if:
- Resources need to remain private (uses NAT Gateway) fronted with public load balancer
- Full source IP visibility to firewall is required for both directions
- Seperate policies for ingress and egress traffic
- NAT Gateway needed for private subnet internet access

## Cost Considerations

Multi-AZ deployment incurs higher costs due to:
- Additional NAT Gateways in second AZ
- Additional firewall endpoints

## Testing Alternative

For development and testing environments, consider the [Single AZ Deployment](../single_az_deployment/) which provides the same functionality at lower cost.

## Additional Resources

- [AWS Network Firewall Documentation](https://docs.aws.amazon.com/network-firewall/)
- [Deployment models for AWS Network Firewall Blog](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/)
