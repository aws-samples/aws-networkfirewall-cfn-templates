# Distributed Architecture - Single AZ Separate Firewalls

**Template File:** [anfw-distributed-1az-dual-firewall-template.yaml](anfw-distributed-1az-dual-firewall-template.yaml)

This template deploys AWS Network Firewall in a distributed architecture with separate firewalls for ingress and egress traffic within one Availability Zone. This configuration provides enhanced security through traffic separation and is designed for testing environments.

![Base Architecture](../../../images/anfw-distributed-model-seperate-endpoint-1az.png)

## Architecture Overview

This template creates a VPC with two AWS Network Firewall deployments - one dedicated to inbound traffic and another for outbound traffic. This separation provides enhanced security, granular policy control, and ensures source IP visibility to the firewalls for both ingress and egress traffic.

## Resources Created

### VPC Subnets
- **Private Subnet** - Contains application resources without public IP addresses
- **NAT Subnet (Public)** - Contains NAT Gateway for outbound Internet access
- **NLB Subnet (Public)** - Contains Network Load Balancer
- **Ingress Firewall Subnet** - Contains firewall endpoint for inbound traffic
- **Egress Firewall Subnet** - Contains firewall endpoint for outbound traffic

### AWS Network Firewall
- **Ingress Firewall** - Dedicated endpoint for inbound Internet traffic
- **Egress Firewall** - Dedicated endpoint for outbound Internet traffic
- Separate firewall policies for ingress and egress traffic
- Independent logging configuration for each firewall

### Networking Components
- **Internet Gateway** - Provides Internet connectivity
- **NAT Gateway** - Enables outbound Internet access for private resources
- **Network Load Balancer** - Provides public IP for ingress connections to private resources
- **Route Tables** - Direct traffic through appropriate firewall endpoints
- **Ingress Route Table** - Associated with Internet Gateway for return traffic

## Traffic Flow

1. **Inbound Traffic** - Internet traffic routes through ingress firewall endpoint to Network Load Balancer (NLB), then to private resources
2. **Outbound Traffic** - Private resources route through egress firewall endpoint to NAT Gateway
3. **Traffic Separation** - Ingress and egress traffic processed by dedicated firewalls
4. **Private Resources** - Resources remain private, using NLB for ingress and NAT Gateway for egress

## Security Benefits

- **Source IP Visibility** - Separate endpoints ensure source IP is visible to firewalls in both directions
- **Traffic Separation** - Ingress and egress traffic isolated
- **Granular Policies** - Different rules for inbound vs outbound traffic
- **Enhanced Monitoring** - Separate logging for each traffic type
- **Reduced Attack Surface** - Compromised policy affects only one traffic direction

## Deployment Instructions

1. Ensure you have appropriate AWS permissions
2. Deploy the CloudFormation template:
   ```bash
   aws cloudformation create-stack \
     --stack-name anfw-distributed-1az-dual \
     --template-body file://anfw-distributed-1az-dual-firewall-template.yaml \
     --capabilities CAPABILITY_IAM
   ```

## Use Cases

- **Security Testing** - Test traffic separation concepts
- **Policy Development** - Develop different policies for ingress/egress
- **Compliance Testing** - Test requirements for traffic isolation

## Limitations

- **Single AZ** - No high availability or fault tolerance
- **Higher Cost** - Two firewall endpoints increase costs
- **Testing Only** - Not suitable for production workloads
- **Complexity** - More complex routing than combined firewall

## High Availability Considerations

For environments requiring high availability, consider:
- [Two AZ Separate Firewalls](../../two_az_deployment/separate-ingress-and-egress-firewall/) for high availability
- [Combined Firewall](../combined-ingress-and-egress-firewall/) for simplified management

## Cost Optimization

For cost-optimized testing, consider the [Combined Firewall](../combined-ingress-and-egress-firewall/) template which uses a single firewall endpoint.

## Additional Resources

- [AWS Network Firewall Documentation](https://docs.aws.amazon.com/network-firewall/)
- [VPC Route Tables Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [Deployment models for AWS Network Firewall Blog](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/)
