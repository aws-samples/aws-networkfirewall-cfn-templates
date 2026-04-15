# Centralized Ingress Inspection - Two AZ Deployment

**Template File:** [anfw-centralized-ingress-2az-template.yaml](anfw-centralized-ingress-2az-template.yaml)

This template deploys AWS Network Firewall in a centralized ingress inspection architecture across two Availability Zones. All inbound internet traffic is routed through the Network Firewall in an Edge VPC before reaching application workloads in spoke VPCs via Transit Gateway. This configuration provides high availability and is recommended for production environments.

## Architecture Overview

This multi-AZ deployment creates a centralized ingress inspection model using an Edge VPC with an internet-facing Application Load Balancer, AWS Network Firewall endpoints in each AZ, and AWS Transit Gateway. Inbound traffic from the internet enters through the Edge VPC Internet Gateway, is steered through the Network Firewall for inspection via an IGW Ingress Route Table, and then reaches the Edge ALB. The ALB terminates TLS and forwards plaintext HTTP to spoke VPC NLBs via Transit Gateway, which distribute traffic to EC2 instances running httpd.

## Resources Created

### Edge VPC (100.64.0.0/16)
Centralized VPC for ingress traffic inspection across two AZs:
- **Firewall Subnets** (100.64.0.16/28, 100.64.0.32/28) - Contains AWS Network Firewall endpoints in each AZ
- **Public Subnets** (100.64.1.0/24, 100.64.2.0/24) - Contains the internet-facing ALB and NAT Gateways
- **TGW Subnets** (100.64.0.0/28, 100.64.0.48/28) - Attachment points for Transit Gateway
- **Internet Gateway** - Entry point for inbound internet traffic
- **NAT Gateways** (one per AZ) - Provide outbound internet access for spoke EC2 instances (package installation, SSM)

### AWS Network Firewall
- Firewall endpoints deployed in both AZs of the Edge VPC
- Firewall policy with STRICT_ORDER rule ordering and REJECT stream exception
- Log-only Suricata rule group for ingress traffic (alerts on HTTP, HTTPS, SSH, ICMP)
- Comprehensive allow-list rule group (defined but not referenced in policy by default)
- CloudWatch logging for both ALERT and FLOW log types (30-day retention)

### Spoke VPCs
Two example workload VPCs demonstrating end-to-end ingress traffic flow:
- **Spoke A** (10.1.0.0/16) - Workload subnets, TGW subnets, EC2 instances with httpd (port 80) in each AZ
- **Spoke B** (10.2.0.0/16) - Workload subnets, TGW subnets, EC2 instances with httpd (port 80) in each AZ
- Internal NLBs with static private IPs fronting EC2 instances in each spoke (TCP port 80 only)
- SSM VPC endpoints for instance management
- Security groups allowing ICMP and HTTP from required CIDRs

### Edge ALB
- Internet-facing Application Load Balancer in the Edge VPC Public Subnets
- (Optional) HTTPS listener on port 443 that terminates TLS using an ACM certificate and forwards to port 80 (conditional on `ALBCertificateArn` parameter)
- HTTP listener on port 80
- Single IP-type target group (port 80, HTTP) with spoke NLB static IPs pre-registered
- Security group allowing inbound HTTP and HTTPS from 0.0.0.0/0

### Transit Gateway
- Central routing hub connecting Edge VPC and spoke VPCs
- Spoke route table with default route to Edge VPC attachment
- Inspection route table with propagated spoke routes
- Appliance Mode enabled on the Edge VPC attachment for flow symmetry

## Traffic Flow

### Ingress Path (Internet → Application)
1. Client request arrives at the Internet Gateway
2. IGW Ingress Route Table steers traffic to the Network Firewall endpoint in the same AZ (Public Subnet CIDR → Firewall Endpoint)
3. Network Firewall inspects the inbound traffic using Suricata stateful rules
4. Inspected traffic reaches the Edge ALB in the Public Subnet via VPC local route
5. ALB terminates TLS (if HTTPS) and forwards HTTP to spoke NLB IPs via Transit Gateway (Public Subnet routes 10.0.0.0/8 → TGW)
6. Spoke NLB distributes traffic to EC2 instances running httpd on port 80

### Ingress Return Path (Application → Internet) — Symmetric
1. EC2 response returns through the spoke NLB → Transit Gateway → Edge ALB
2. Edge ALB sends the response to the client; Public Subnet routes 0.0.0.0/0 → **Firewall Endpoint** (same AZ)
3. Network Firewall inspects the return traffic (matching the stateful flow from the inbound path)
4. Firewall Subnet routes 0.0.0.0/0 → **Internet Gateway**; response exits to the internet

> **Symmetric routing**: Inbound traffic goes IGW → Firewall → ALB, and return traffic goes ALB → Firewall → IGW. The firewall sees both directions of every flow.

### Spoke Egress Path (e.g., yum install, SSM)
1. Spoke EC2 initiates outbound connection → Spoke route table sends 0.0.0.0/0 → TGW
2. TGW Subnet route table sends 0.0.0.0/0 → **NAT Gateway** (same AZ, source IP translated)
3. NAT Gateway outbound follows Public Subnet route: 0.0.0.0/0 → **Firewall Endpoint** (same AZ)
4. Firewall inspects the NAT'd egress traffic → Firewall Subnet routes 0.0.0.0/0 → **IGW**
5. Return traffic from the internet is steered back through the firewall via the IGW Ingress Route Table

## High Availability Features

- **Multi-AZ Deployment** - Resources distributed across two Availability Zones
- **AZ-Specific Routing** - Each Public Subnet routes to its same-AZ firewall endpoint; each TGW Subnet routes to its same-AZ NAT Gateway
- **Per-AZ Firewall Endpoints** - Network Firewall endpoints in each AZ ensure traffic stays within the same AZ

## Deployment Instructions

1. Ensure you have appropriate AWS permissions
2. (Optional) Provision or import an ACM certificate for the ALB HTTPS listener - if not provided, the template will skip the HTTPS listener
3. Deploy the CloudFormation template:
   ```bash
   aws cloudformation create-stack \
     --stack-name anfw-centralized-ingress-2az \
     --template-body file://anfw-centralized-ingress-2az-template.yaml \
     --capabilities CAPABILITY_IAM \
     --parameters ParameterKey=ALBCertificateArn,ParameterValue=arn:aws:acm:REGION:ACCOUNT:certificate/CERTIFICATE-ID
   ```
   Omit the `ALBCertificateArn` parameter to deploy without the HTTPS listener (HTTP only).
4. Wait for the stack to reach `CREATE_COMPLETE` status
5. Allow approximately 8-10 minutes after stack completion for EC2 instances to finish UserData execution (installing httpd via the NAT Gateway egress path) and for the Edge ALB target group health checks to pass. You can monitor target health with:
   ```bash
   aws elbv2 describe-target-health \
     --target-group-arn $(aws elbv2 describe-target-groups --names e-http-anfw-centralized-ingress-2az --query 'TargetGroups[0].TargetGroupArn' --output text)
   ```
6. Access the Edge ALB DNS name from the stack outputs to test ingress traffic flow

## Important Notes

- **(Optional) TLS Termination** - TLS is terminated at the Edge ALB using an ACM certificate. All downstream traffic to spoke NLBs and EC2 instances is plaintext HTTP on port 80.
- **Spoke NLB Static IPs** - Spoke NLBs use static private IPs via SubnetMappings, pre-registered as ALB targets. No manual target registration is needed.
- **Symmetric Inspection** - The Public Subnet routes 0.0.0.0/0 to the Firewall Endpoint (not the IGW) to ensure return traffic passes through the firewall, avoiding asymmetric routing.

## Enabling the Allow-List Rule Group

By default, only the log-only rule group is active — it alerts on inbound traffic patterns without blocking anything. The template also creates a comprehensive allow-list rule group (`IngressAllowListRuleGroup`) that is defined but not referenced in the firewall policy.

When you're ready to move beyond log-only mode and enforce ingress filtering:

1. Open the [Network Firewall console](https://console.aws.amazon.com/vpc/home#NetworkFirewallPolicies) and select the `ingress-firewall-policy-<stack-name>` policy
2. Under **Stateful rule group references**, click **Add rule group** and select `ingress-allow-list-<stack-name>`
3. Set its priority to a value higher than the log-only group (e.g., 100 if log-only is at 50) so alerts fire before enforcement
4. Save the policy — the firewall endpoints will sync within a few minutes

The allow-list rules will:
- **Pass** inbound HTTP (port 80) and HTTPS (port 443) to HOME_NET
- **Pass** established return traffic from HOME_NET
- **Alert** on traffic from RU/CN geolocations
- **Drop** all other inbound TCP, UDP, ICMP, and IP traffic

> **Note:** The log-only group fires first (lower priority number), so you retain full alert visibility even after enabling enforcement. To revert to log-only mode, remove the allow-list reference from the policy. If you added the allow-list via the console (outside CloudFormation), remember to remove it before deleting the stack to avoid delete failures.

## Cost Considerations

This deployment incurs higher costs compared to single AZ due to:
- Additional NAT Gateway in second AZ
- Additional firewall endpoint in second AZ
- ALB cross-AZ load balancing

## Testing Alternative

For development and testing environments, consider the [Single AZ Deployment](../centralized_ingress_single_az/) which provides the same functionality at lower cost using an NLB instead of an ALB.

## Additional Resources

- [AWS Network Firewall Documentation](https://docs.aws.amazon.com/network-firewall/)
- [AWS Transit Gateway Documentation](https://docs.aws.amazon.com/transit-gateway/)
- [Deployment models for AWS Network Firewall Blog](https://aws.amazon.com/blogs/networking-and-content-delivery/deployment-models-for-aws-network-firewall/)
