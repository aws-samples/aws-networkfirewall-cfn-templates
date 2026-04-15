# Centralized Ingress Inspection - Single AZ Deployment

**Template File:** [anfw-centralized-ingress-1az-template.yaml](anfw-centralized-ingress-1az-template.yaml)

This template deploys AWS Network Firewall in a centralized ingress inspection architecture within a single Availability Zone. All inbound internet traffic is routed through the Network Firewall in an Edge VPC before reaching application workloads in spoke VPCs via Transit Gateway. This configuration is designed for testing, development, and proof-of-concept environments.

## Architecture Overview

This single AZ deployment creates a centralized ingress inspection model using an Edge VPC with an internet-facing Network Load Balancer, AWS Network Firewall, and AWS Transit Gateway. Inbound traffic from the internet enters through the Edge VPC Internet Gateway, is steered through the Network Firewall for inspection via an IGW Ingress Route Table, and then reaches the Edge NLB. The Edge NLB forwards traffic to spoke VPC NLBs via Transit Gateway, which in turn distribute traffic to EC2 instances running httpd.

The Edge VPC uses an NLB (not an ALB) because ALBs require a minimum of two Availability Zones. For the high-availability variant with ALB support, see the [Two AZ Deployment](../centralized_ingress_two_az/).

## Resources Created

### Edge VPC (100.64.0.0/16)
Centralized VPC for ingress traffic inspection:
- **Firewall Subnet** (100.64.0.16/28) - Contains the AWS Network Firewall endpoint
- **Public Subnet** (100.64.1.0/24) - Contains the internet-facing NLB and NAT Gateway
- **TGW Subnet** (100.64.0.0/28) - Attachment point for Transit Gateway
- **Internet Gateway** - Entry point for inbound internet traffic
- **NAT Gateway** - Provides outbound internet access for spoke EC2 instances (package installation, SSM)

### AWS Network Firewall
- Firewall endpoint deployed in the Edge VPC Firewall Subnet
- Firewall policy with STRICT_ORDER rule ordering and REJECT stream exception
- Log-only Suricata rule group for ingress traffic (alerts on HTTP, HTTPS, SSH, ICMP)
- Comprehensive allow-list rule group (defined but not referenced in policy by default)
- CloudWatch logging for both ALERT and FLOW log types (30-day retention)

### Spoke VPCs
Two example workload VPCs demonstrating end-to-end ingress traffic flow:
- **Spoke A** (10.1.0.0/16) - Workload subnet, TGW subnet, EC2 instance with httpd (port 80)
- **Spoke B** (10.2.0.0/16) - Workload subnet, TGW subnet, EC2 instance with httpd (port 80)
- Internal NLBs with static private IPs fronting EC2 instances in each spoke (TCP port 80 only)
- SSM VPC endpoints for instance management
- Security groups allowing ICMP and HTTP from required CIDRs

### Edge NLB
- Internet-facing Network Load Balancer in the Edge VPC Public Subnet
- TLS listener on port 443 that terminates TLS using an ACM certificate and forwards to port 80
- TCP listener on port 80
- Single IP-type target group (port 80) with spoke NLB static IPs pre-registered

### Transit Gateway
- Central routing hub connecting Edge VPC and spoke VPCs
- Spoke route table with default route to Edge VPC attachment
- Inspection route table with propagated spoke routes
- Appliance Mode enabled on the Edge VPC attachment for flow symmetry

## Traffic Flow

### Ingress Path (Internet → Application)
1. Client request arrives at the Internet Gateway
2. IGW Ingress Route Table steers traffic to the Network Firewall endpoint (Public Subnet CIDR → Firewall Endpoint)
3. Network Firewall inspects the inbound traffic using Suricata stateful rules
4. Inspected traffic reaches the Edge NLB in the Public Subnet via VPC local route
5. Edge NLB forwards to spoke NLB IPs via Transit Gateway (Public Subnet routes 10.0.0.0/8 → TGW)
6. Spoke NLB distributes traffic to EC2 instances running httpd

### Ingress Return Path (Application → Internet) — Symmetric
1. EC2 response returns through the spoke NLB → Transit Gateway → Edge NLB
2. Edge NLB sends the response to the client; Public Subnet routes 0.0.0.0/0 → **Firewall Endpoint**
3. Network Firewall inspects the return traffic (matching the stateful flow from the inbound path)
4. Firewall Subnet routes 0.0.0.0/0 → **Internet Gateway**; response exits to the internet

> **Symmetric routing**: Inbound traffic goes IGW → Firewall → NLB, and return traffic goes NLB → Firewall → IGW. The firewall sees both directions of every flow.

### Spoke Egress Path (e.g., yum install, SSM)
1. Spoke EC2 initiates outbound connection → Spoke route table sends 0.0.0.0/0 → TGW
2. TGW Subnet route table sends 0.0.0.0/0 → **NAT Gateway** (source IP translated)
3. NAT Gateway outbound follows Public Subnet route: 0.0.0.0/0 → **Firewall Endpoint**
4. Firewall inspects the NAT'd egress traffic → Firewall Subnet routes 0.0.0.0/0 → **IGW**
5. Return traffic from the internet is steered back through the firewall via the IGW Ingress Route Table

## Key Routing Design

The routing differs from the existing egress/east-west templates to support symmetric ingress inspection:

| Route Table | Default Route (0.0.0.0/0) | Purpose |
|---|---|---|
| IGW Ingress RT | Public Subnet CIDR → Firewall Endpoint | Steer inbound traffic through firewall |
| Firewall Subnet RT | → Internet Gateway | Exit point for all inspected outbound traffic |
| Public Subnet RT | → Firewall Endpoint | Ensures ALB return traffic and NAT GW egress pass through firewall |
| TGW Subnet RT | → NAT Gateway | Spoke egress gets NAT'd before firewall inspection |
| Spoke Workload RT | → Transit Gateway | All spoke outbound traffic via TGW |

## Deployment Instructions

1. Ensure you have appropriate AWS permissions
2. Provision or import an ACM certificate for the Edge NLB TLS listener
3. Deploy the CloudFormation template:
   ```bash
   aws cloudformation create-stack \
     --stack-name anfw-centralized-ingress-1az \
     --template-body file://anfw-centralized-ingress-1az-template.yaml \
     --capabilities CAPABILITY_IAM \
     --parameters ParameterKey=NLBCertificateArn,ParameterValue=arn:aws:acm:REGION:ACCOUNT:certificate/CERTIFICATE-ID
   ```
4. Wait for the stack to reach `CREATE_COMPLETE` status
5. Allow approximately 8-10 minutes after stack completion for EC2 instances to finish UserData execution (installing httpd via the NAT Gateway egress path) and for the Edge NLB target group health checks to pass. You can monitor target health with:
   ```bash
   aws elbv2 describe-target-health \
     --target-group-arn $(aws elbv2 describe-target-groups --names e-http-anfw-centralized-ingress-1az --query 'TargetGroups[0].TargetGroupArn' --output text)
   ```
6. Access the Edge NLB DNS name from the stack outputs to test ingress traffic flow

## Important Notes

- **Single AZ Limitation** - This deployment lacks high availability and should not be used in production. Designed for development, testing, and learning environments.
- **NLB vs ALB** - The 1AZ template uses an internet-facing NLB because ALBs require a minimum of 2 AZs. The [Two AZ Deployment](../centralized_ingress_two_az/) uses an ALB for Layer 7 capabilities.
- **TLS Termination** - TLS is terminated at the Edge NLB using an ACM certificate. All downstream traffic to spoke NLBs and EC2 instances is plaintext HTTP on port 80.
- **Spoke NLB Static IPs** - Spoke NLBs use static private IPs via SubnetMappings, pre-registered as Edge NLB targets. No manual target registration is needed.
- **Symmetric Inspection** - The Public Subnet routes 0.0.0.0/0 to the Firewall Endpoint (not the IGW) to ensure return traffic passes through the firewall, avoiding asymmetric routing.

## Production Considerations

For environments requiring high availability, consider the [Two AZ Deployment](../centralized_ingress_two_az/) which provides:
- High availability across multiple Availability Zones
- ALB with Layer 7 capabilities and HTTPS termination
- AZ-specific routing for fault tolerance

## Additional Resources

- [AWS Network Firewall Documentation](https://docs.aws.amazon.com/network-firewall/)
- [AWS Transit Gateway Documentation](https://docs.aws.amazon.com/transit-gateway/)
- [Deployment models for AWS Network Firewall Blog](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/)
