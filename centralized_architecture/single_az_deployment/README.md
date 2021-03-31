# Centralized Architecture - Single AZ Deployment:

**Template File:** [anfw-centralized-single-1az-template.yaml](anfw-centralized-1az-template.yaml)

For centralized deployment model, [AWS Transit Gateway](https://aws.amazon.com/transit-gateway/) is a prerequisite. AWS Transit Gateway acts as a network hub and simplifies the connectivity between VPCs as well as on-premises networks. AWS Transit Gateway also provides inter-region peering capabilities to other Transit Gateways to establish a global network using AWS backbone.

![anfw-centralized-model-1az](images/anfw-centralized-model-1az.jpg)
*Figure 1: Single AZ Centralized Architecture*

[anfw-centralized-single-1az-template.yaml](anfw-centralized-1az-template.yaml) template, as described in Figure 1, creates dedicated

* Inspection VPC for East-West (inter-vpc) traffic inspection. Inspection VPC consists of two subnets in single AZ:
  1. Transit Gateway subnet for Transit Gateway attchment.
  2. Firewall subnet for firewall endpoint.

* Central Egress VPC for North-South (spoke VPCs to Internet) traffic inspection. Central Egress VPC consists of 3 subnets in single AZ: 
  1. Transit Gateway subnet for Transit Gateway attchment.
  2. Firewall subnet for firewall endpoint.
  3. Public subnet for NAT Gateway.

* Two Spoke VPCs.

Each Transit Gateway subnet in each dedicated VPC requires a dedicated VPC route table to ensure the traffic is forwarded to firewall endpoint within the same AZ. These route tables have a default route (0.0.0.0/0) pointing towards firewall endpoint in the same AZ.

This is a single AZ configuration. Use it for testing/POC. It is recommended to deploy resources in atleast 2 AZs. For multi AZ deployment refer to [Multi AZ Deployment](../README.md).

For more details, refer to [Blog: Deployment models for AWS Network Firewall](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/)