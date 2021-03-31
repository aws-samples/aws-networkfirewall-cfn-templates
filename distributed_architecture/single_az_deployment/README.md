# Distributed Architecture - Single AZ Deployment:

**Template File:** [anfw-distributed-1az-template.yaml](anfw-distributed-1az-template.yaml)

For centralized deployment model, [AWS Transit Gateway](https://aws.amazon.com/transit-gateway/) is a prerequisite. AWS Transit Gateway acts as a network hub and simplifies the connectivity between VPCs as well as on-premises networks. AWS Transit Gateway also provides inter-region peering capabilities to other Transit Gateways to establish a global network using AWS backbone.

![anfw-distributed-model-1az](../../images/anfw-distributed-model-1az.png)
*Figure 1: Single AZ Distrbuted Architecture*

[Distributed single AZ deployment template](anfw-distributed-1az-template.yaml), as described in Figure 1, creates:

* Spoke VPC. Spoke VPC consists of three subnets in each AZ:.
  * Public subnet for client with public ip.
  * Firewall subnet for firewall endpoint.

For return traffic, Ingress Route Table is associated with Internet Gateway to ensure the traffic is forwarded to firewall endpoint within the same AZ. This route tables has public subnet route pointing towards firewall endpoint in the same AZ.

This is a single AZ configuration. Use it for testing/POC. It is recommended to deploy resources in atleast 2 AZs. For multi AZ deployment refer to [Multi AZ Deployment](../README.md).

For more details, refer to [Blog: Deployment models for AWS Network Firewall](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/)
