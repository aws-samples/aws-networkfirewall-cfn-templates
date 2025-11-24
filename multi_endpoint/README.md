# AWS Network Firewall Multi-Endpoint Architecture

CloudFormation template for deploying AWS Network Firewall with multiple endpoints for combined ingress/egress inspection architectures.

## Overview

This template addresses a critical challenge in AWS Network Firewall deployments: maintaining source IP visibility when inspecting both ingress and egress traffic within the same VPC via multiple AWS Network Firewall endpoints. 

**The Problem**: Traditional on-premises firewalls handle source NAT (SNAT) and destination NAT (DNAT) on the same appliance, preserving source IP visibility. On AWS, inspection is decoupled from NAT functions - NAT Gateways handle SNAT for egress, and Elastic Load Balancers handle DNAT for ingress. This separation can mask source IPs during inspection.

**The Solution**: Deploy **two firewall endpoints per availability zone** using AWS Network Firewall's multiple VPC endpoints feature. This architecture ensures:

- **Ingress inspection** occurs BEFORE ELB destination NAT (preserving client source IPs)
- **Egress inspection** occurs BEFORE NAT Gateway source NAT (preserving internal source IPs)  
- **Single firewall** with multiple endpoints vs. deploying separate firewalls (cost optimization)
- **Proper traffic separation** through dedicated subnets and route tables

This represents a significant improvement over previous approaches that either sacrificed source IP visibility or required complex dual-firewall architectures. The multiple endpoints feature enables both cost efficiency and security visibility in a single deployment.

### Architecture

![Multi-Endpoint Network Firewall Architecture](../images/DistributedSeparate1AZArch.png)

**Traffic Separation:**
- **Primary Endpoint**: Handles traffic going OUT from your private instances to the internet (egress inspection)
- **Secondary Endpoint**: Handles traffic coming IN from the internet to your services (ingress inspection)

## Quick Start

### Deploy
```bash
aws cloudformation create-stack \
  --stack-name nfw-multi-endpoint-demo \
  --template-body file://nfw-multi-endpoint.yaml \
  --parameters ParameterKey=AvailabilityZoneSelection,ParameterValue=us-east-1a \
  --capabilities CAPABILITY_IAM
```


## Source IP Visibility Architecture

This template follows AWS best practices for maintaining source IP visibility in combined ingress/egress inspection:

- **Ingress inspection** inbound from Internet via secondary firewall endpoint
- **Egress inspection** outbound from private instances via primary firewall endpoint
- **Separate endpoints** ensures all traffic flows through the firewall in the correct direction and makes the traffic paths more predictable
- **Route separation** ensures proper traffic flow through designated SSM VPC endpoints so private instances can communicate with AWS Systems Manager without going through the internet


## References

- [Creating a VPC endpoint association in AWS Network Firewall](https://docs.aws.amazon.com/network-firewall/latest/developerguide/creating-vpc-endpoint-association.html)
- [Source IP Visibility for Combined Ingress/Egress Inspection](https://repost.aws/articles/ARYy1Pfr4BQOGvxntapZBgSQ/source-ip-visibility-for-combined-ingress-and-egress-inspection-architectures)