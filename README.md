# AWS Network Firewall CloudFormation Templates

Sample CloudFormation templates demonstrating [AWS Network Firewall](https://aws.amazon.com/network-firewall/) routing architectures and deployment models.

> **Looking for Terraform?** These same architectures are available as Terraform templates: [aws-network-firewall-terraform](https://github.com/aws-samples/aws-network-firewall-terraform)

## Available Architectures

### Centralized Architecture

Uses AWS Transit Gateway for centralized inspection of East-West (VPC-to-VPC) and egress (internet-bound) traffic.

#### [Transit Gateway-Attached Firewall](transit-gateway-attached-firewall/)

Attaches AWS Network Firewall directly to [Transit Gateway as a native attachment](https://docs.aws.amazon.com/network-firewall/latest/developerguide/tgw-firewall.html). AWS creates and manages the inspection VPC transparently, removing the need to create and manage your own.

**Note:** Transit Gateway-Attached Firewall is required to use [Transit Gateway Flexible Cost Allocation](https://docs.aws.amazon.com/vpc/latest/tgw/metering-policy.html) for chargebacks. The other centralized deployment models in this repository do not support this feature.

| Template | Use Case |
|----------|----------|
| [Manual Deployment](transit-gateway-attached-firewall/manual-deployment-tgw-attached-firewall/) | Learning and hands-on configuration |
| [Pre-Deployed](transit-gateway-attached-firewall/pre-deployed-tgw-attached-firewall/) | Automated provisioning |

![TGW-Attached Firewall](images/tgw-native-attach-full.png)

#### [Inspection VPC Model](centralized_architecture/)

Routes traffic through a dedicated inspection VPC containing the firewall endpoints.

| Template | Use Case |
|----------|----------|
| [Single AZ](centralized_architecture/single_az_deployment/) | Single availability zone |
| [Two AZ](centralized_architecture/two_az_deployment/) | High availability across two AZs |

![Centralized Architecture](images/anfw-centralized-model-1az.png)

### Distributed Architecture

Deploys AWS Network Firewall into each VPC individually. No Transit Gateway required—each VPC is protected independently.

#### [Multiple VPC Endpoint Associations](multi_endpoint/)

Leverages the [VPC Endpoint Association feature](https://docs.aws.amazon.com/network-firewall/latest/developerguide/creating-vpc-endpoint-association.html) to deploy multiple firewall endpoints per availability zone to maintain source IP visibility when inspecting both ingress and egress traffic with the same firewall.

![Multi-Endpoint Architecture](images/DistributedSeparate1AZArch.png)

#### [Single Endpoint](distributed_architecture/)

Single firewall endpoint per availability zone with options for combined or separate ingress/egress inspection.

| Configuration | Single AZ | Two AZ |
|--------------|-----------|--------|
| Combined Ingress/Egress Firewall | [Template](distributed_architecture/single_az_deployment/combined-ingress-and-egress-firewall/) | [Template](distributed_architecture/two_az_deployment/combined-ingress-and-egress-firewall/) |
| Separate Ingress/Egress Firewalls | [Template](distributed_architecture/single_az_deployment/separate-ingress-and-egress-firewall/) | [Template](distributed_architecture/two_az_deployment/separate-ingress-and-egress-firewall/) |

![Distributed Architecture - Combined](images/anfw-distributed-model-combined-endpoint-1az.png)

![Distributed Architecture - Separate](images/anfw-distributed-model-seperate-endpoint-1az.png)

### [Egress Inspection with AWS Cloud WAN](outbound_inspection_with_aws_cloud_wan/)

Workshop-based templates for deploying egress inspection using AWS Cloud WAN and AWS Network Firewall across multiple regions.

![Cloud WAN Architecture](images/egress-inspection-aws-cloud-wan-base-architecture.png)

### [CloudWatch Dashboard](cloudwatch_dashboard/)

CloudFormation templates for creating a comprehensive monitoring dashboard for AWS Network Firewall metrics and logs.

| Template | Region |
|----------|--------|
| [Standard](cloudwatch_dashboard/nfw-cloudwatch-dashboard.yaml) | Commercial regions |
| [GovCloud](cloudwatch_dashboard/nfw-cloudwatch-dashboard-govcloud.yaml) | AWS GovCloud |
| [China](cloudwatch_dashboard/nfw-cloudwatch-dashboard-china.yaml) | AWS China regions |

## License

This sample code is made available under the MIT-0 license. See the [LICENSE](LICENSE) file.

## Additional Resources

- [AWS Network Firewall Best Practices Guide](https://aws.github.io/aws-security-services-best-practices/guides/network-firewall/)
- [Deployment Models for AWS Network Firewall](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/)
- [Deployment Models for AWS Network Firewall - Part 2](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall-with-vpc-routing-enhancements/)

## Authors

| Name | Email |
|------|-------|
| Lawton Pittenger, WW Security Specialist Solutions Architect | lawtontp@amazon.com |
| Anvesh Koganti, Networking Specialist Solutions Architect | anvkog@amazon.com |
| Pratik R. Mankad, Sr. Solutions Architect | pmankad@amazon.com |
| Shakeel Ahmad, Sr. Solutions Architect | shkahma@amazon.com |
| Daniel Yu, Sr. Technical Account Manager | dyuamzn@amazon.com |


