# Distributed Architecture - Single AZ Deployment:

**Template File:** [anfw-distributed-1az-template.yaml](anfw-distributed-1az-template.yaml)

For the distributed deployment model, we deploy AWS Network Firewall into each VPC which requires protection. Each VPC is protected individually and blast radius is reduced through VPC isolation. Each VPC does not require connectivity to any other VPC or AWS Transit Gateway. Each AWS Network Firewall can have its own firewall policy or share a policy through common rule groups (reusable collections of rules) across multiple firewalls. This allows each AWS Network Firewall to be managed independently, which reduces the possibility of misconfiguration and limits the scope of impact.

![anfw-distributed-model-1az](../../images/anfw-distributed-model-1az.png)
*Figure 1: Single AZ Distrbuted Architecture*

[Distributed single AZ deployment template](anfw-distributed-1az-template.yaml), as described in Figure 1, creates:

* Spoke VPC. Spoke VPC consists of three subnets in each AZ:.
  * Public subnet for client with public ip.
  * Firewall subnet for firewall endpoint.

For return traffic, Ingress Route Table is associated with Internet Gateway to ensure the traffic is forwarded to firewall endpoint within the same AZ. This route tables has public subnet route pointing towards firewall endpoint in the same AZ.

This is a single AZ configuration. Use it for testing/POC. It is recommended to deploy resources in atleast 2 AZs. For multi AZ deployment refer to [Multi AZ Deployment](../README.md).

For more details, refer to [Blog: Deployment models for AWS Network Firewall](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/)
