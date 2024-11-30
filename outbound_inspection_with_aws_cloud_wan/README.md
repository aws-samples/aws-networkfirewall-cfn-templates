## Prerequisites

An AWS account with an IAM user with the appropriate permissions

## Description

The repository contains AWS CloudFormation sample code for the following workshop:

* [Egress Inspection with AWS Cloud WAN and AWS Network Firewall](https://catalog.us-east-1.prod.workshops.aws/workshops/547dc923-8c8f-45b2-a772-f1c233e6864c/en-US)

![Base Architecture](../static/images/Base_Architecture.png)

* To run the workshop in your environment, visit [workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/547dc923-8c8f-45b2-a772-f1c233e6864c/en-US) guide.

## Usage
* Clone the repository.
* Change the directory to (cd) to `/path_to/aws-networkfirewall-cfn-templates/outbound_inspection_with_aws_cloud_wan`.
* (Optional) Edit the VPC and subnet CIDRs in the following region specific resource files if you want to test with other values:
  * region1_resources.yaml
  * region2_resources.yaml
  * region3_resources.yaml
* (Optional). The repository uses us-east-1, us-east-2 and us-west-2 as the deployment regions. If you want to use other regions: 
    * In the `Makefile`, change --region and --stack-name to to the desired values.
    * In the `core_network_base_policy.yaml` and `core_network_update_policy_edge_locations.yaml`, change the edge-locations to the desired values.
* You will find two files creating Core Network policy documents.
  * `core_network_base_policy.yaml` is used to create with two edge location first. This is to mimic randomly deterministic behavior.
  * `core_network_update_policy_edge_locations.yaml` contains the final format for the policy document with all three edge locaiton.
* Deploy the resources using `make deploy`.
* Remember to clean up resoures once you are done by using `make undeploy`.
  * When cleaing up resources, CloudWatch LogGroups created by AWS CloudFormation might not get deleted and you will have to manually detele it. For more details refer to [Stack deletion deletes log group but re-creates it on lambda invocation](https://repost.aws/questions/QUzZH7Nz2DT_-PagH_godYYA/stack-deletion-deletes-log-group-but-re-creates-it-on-lambda-invocation)

**Note:**
Keep the following in mind when testing this environment from a cost perspective - for production environments, we recommend the use of at least 2 AZs for high-availability:
* AWS Cloud WAN core network edge (CNE) gets created in each edge location.
* EC2 instances will be deployed in all the Workload Subnets configured for each Prod VPC. 
* EC2 Instance Connect Endpoint will be deloyed in one Endpoint Subnet configured for each Prod VPC 1, 2 and 3 and in one Firewall Subnet configured for Prod VPC 4.
* AWS Network Firewall Endpoints will be deployed in all the Firewall Subnets configured for each inspection VPC and Prod VPC 4.
* NAT Gateway will be deployed in all the Public Subnets configured for each inspection VPC and Prod VPC 4.
