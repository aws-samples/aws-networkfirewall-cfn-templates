# Manual Deployment - Transit Gateway-Attached Firewall

**Template File:** [tgw-no-firewall-manual.yaml](tgw-no-firewall-manual.yaml)

This guide walks you through deploying AWS Network Firewall with Transit Gateway native attachment using a hands-on approach. You'll first deploy the base infrastructure, then manually configure the firewall attachment and routing to enable traffic inspection.

![Base Infrastructure](../../images/tgw-native-attach-base.png)

## Architecture Overview

### Base Infrastructure (Before Firewall Configuration)

The CloudFormation template deploys the foundational infrastructure without firewall inspection:

**Initial Traffic Flows:**
- **East-West Traffic (Spoke to Spoke):** Traffic flows directly between spoke VPCs through Transit Gateway without inspection
  - Spoke A → TGW (Spoke Route Table) → Spoke B
- **Egress Traffic (Internet-bound):** Traffic flows from spoke VPCs to egress VPC for internet access without inspection
  - Spoke → TGW (Spoke Route Table) → Egress VPC → NAT Gateway → Internet

**Resources Created:**
- Transit Gateway with two route tables (Spoke and Egress)
- Two Spoke VPCs (Spoke A and Spoke B) with EC2 instances
- Egress VPC with NAT Gateway and Internet Gateway
- CloudWatch Log Groups for firewall logging (pre-created for later use)

### Final Architecture (After Manual Configuration)

After completing the manual steps, traffic will flow through the firewall for inspection:

**Updated Traffic Flows:**
- **East-West Traffic:** Spoke A → TGW (Spoke Route Table) → Network Firewall → TGW (Spoke Route Table) → Spoke B
- **Egress Traffic:** Spoke → TGW (Spoke Route Table) → Network Firewall → TGW (Inspection Route Table) → Egress VPC → NAT Gateway → Internet

For a fully automated deployment, see the [Pre-Deployed](../pre-deployed-tgw-attached-firewall/) option.

---

## Deployment Instructions

## Phase 1: Deploy Base Infrastructure

### Step 1: Deploy CloudFormation Stack

Deploy the base infrastructure template that creates the Transit Gateway, spoke VPCs, and egress VPC.

```bash
aws cloudformation create-stack \
  --stack-name tgw-manual-firewall \
  --template-body file://tgw-no-firewall-manual.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=AvailabilityZoneSelection,ParameterValue=<your-az>
```

Wait for the stack to reach `CREATE_COMPLETE` status before proceeding.

![Deploy CloudFormation Stack](../../images/transit-gateway-attached-firewall-manual-instructions/deploy-cloudformation-stack.png)

---

## Phase 2: Configure Network Firewall

### Step 2: Create AWS Network Firewall

Navigate to the AWS Network Firewall console to create the firewall with Transit Gateway attachment.

1. Open the **VPC Console** → **Network Firewall** → **Firewalls**
2. Click **Create firewall** (e.g., `tgw-attached-firewall`)
3. Enter a **Firewall name**
4. Click **Next**

![Create Network Firewall](../../images/transit-gateway-attached-firewall-manual-instructions/create-network-firewall.png)

---

### Step 3: Configure Transit Gateway Attachment

Configure the firewall to attach directly to the Transit Gateway as a network function.

1. Under **Attachment details**, select **Transit Gateway**
2. From the dropdown, choose the Transit Gateway created by the CloudFormation stack
3. Under **Associated Availability Zone**, select the same AZ you specified during stack deployment
4. Click **Next**

![Configure Transit Gateway Attachment](../../images/transit-gateway-attached-firewall-manual-instructions/configure-tgw-attachment.png)

---

### Step 4: Configure Logging

Enable logging to capture traffic flow and alert data.

1. Enable **Alert logs**
   - **Destination type:** CloudWatch Logs
   - **Log group:** Select the pre-created log group (e.g., `/stack-name/egress-fw/alert`)
2. Enable **Flow logs**
   - **Destination type:** CloudWatch Logs
   - **Log group:** Select the pre-created log group (e.g., `/stack-name/egress-fw/flow`)
3. Click **Next**

![Configure Firewall Logging](../../images/transit-gateway-attached-firewall-manual-instructions/configure-firewall-logging.png)

---

### Step 5: Create Firewall Policy

Create a basic firewall policy to control traffic inspection behavior.

1. Select **Create and associate a new firewall policy**
2. Enter a **Policy name** (e.g., `inspection-policy`)
3. Configure policy settings:
   - **Rule evaluation order:** Strict order (default)
   - **Drop action:** None
   - **Alert action:** Alert established (default)
4. Progress to the last screen and click **Create firewall**

Wait for the firewall to reach `Ready` status (this may take 5-10 minutes).

![Create Firewall Policy](../../images/transit-gateway-attached-firewall-manual-instructions/create-firewall-policy.png)
![Create Firewall](../../images/transit-gateway-attached-firewall-manual-instructions/create-firewall-review.png)
---

## Phase 3: Configure Transit Gateway Routing

### Step 6: Create Inspection Route Table

Create a new Transit Gateway route table for the firewall attachment.

1. Navigate to **VPC Console** → **Transit Gateways** → **Transit gateway route tables**
2. Click **Create transit gateway route table**
3. Configure:
   - **Name:** Enter a name (e.g., `inspection-route-table`)
   - **Transit Gateway:** Select the existing Transit Gateway
4. Click **Create transit gateway route table**

![Create Inspection Route Table](../../images/transit-gateway-attached-firewall-manual-instructions/create-inspection-route-table.png)

---

### Step 7: Configure Route Propagations

Enable route propagations so the inspection route table learns spoke VPC routes.

1. Select the **inspection-route-table**
2. Navigate to the **Propagations** tab
3. Click **Create propagation**
4. Select **Spoke A VPC attachment** and click **Create propagation**
5. Repeat for **Spoke B VPC attachment**

This allows the firewall to route traffic back to the correct spoke VPC after inspection.

![Configure Route Propagations](../../images/transit-gateway-attached-firewall-manual-instructions/configure-route-propagations.png)

---

### Step 8: Create Default Route to Egress VPC

Configure the inspection route table to send internet-bound traffic to the egress VPC.

1. Within the **inspection-route-table**, navigate to the **Routes** tab
2. Click **Create static route**
3. Configure:
   - **CIDR:** `0.0.0.0/0`
   - **Attachment:** Select the **Egress VPC attachment**
4. Click **Create static route**

![Create Default Route to Egress](../../images/transit-gateway-attached-firewall-manual-instructions/create-default-route-egress.png)

---

### Step 9: Associate Firewall Attachment to Inspection Route Table

Associate the Network Firewall attachment with the inspection route table.

1. Within the **inspection-route-table**, navigate to the **Associations** tab
2. Click **Create association**
3. Select the **Network Firewall attachment** (shown as Network Function)
4. Click **Create association**

![Associate Firewall to Route Table](../../images/transit-gateway-attached-firewall-manual-instructions/associate-firewall-route-table.png)

---

### Step 10: Remove Spoke Propagations from Spoke Route Table

Remove spoke VPC propagations from the spoke route table to prevent direct routing between spokes.

1. Select the **spoke-route-table**
2. Navigate to the **Propagations** tab
3. Delete the propagation for **Spoke A VPC attachment**
4. Delete the propagation for **Spoke B VPC attachment**

![Remove Spoke Propagations](../../images/transit-gateway-attached-firewall-manual-instructions/remove-spoke-propagations.png)

---

### Step 11: Update Spoke Route Table Default Route

Redirect spoke VPC traffic to flow through the firewall instead of directly to the egress VPC.

1. Within the **spoke-route-table**, navigate to the **Routes** tab
2. Locate the existing default route (`0.0.0.0/0` pointing to Egress VPC)
3. Select the route, then click **Actions** → **Replace static route**
4. Configure:
   - **Attachment:** Select the **Network Firewall attachment** (Network Function)
5. Click **Replace static route**

This ensures all spoke traffic is inspected by the firewall before reaching its destination.

![Update Spoke Route Table](../../images/transit-gateway-attached-firewall-manual-instructions/update-spoke-route-table.png)
![Update Spoke Route Table - 2](../../images/transit-gateway-attached-firewall-manual-instructions/update-spoke-route-table-2.png)

---

### Step 12: Configure Egress Route Table

Update the egress route table to send return traffic through the firewall.

1. Select the **egress-route-table**
2. Navigate to the **Propagations** tab
3. Delete all existing propagations (Spoke A and Spoke B attachments)
4. Navigate to the **Routes** tab
5. Click **Create static route**
6. Configure:
   - **CIDR:** `0.0.0.0/0`
   - **Attachment:** Select the **Network Firewall attachment** (Network Function)
7. Click **Create static route**

This ensures return traffic from the egress VPC flows through the firewall for stateful inspection.

![Configure Egress Route Table](../../images/transit-gateway-attached-firewall-manual-instructions/configure-egress-route-table.png)
![Configure Egress Route Table - 2](../../images/transit-gateway-attached-firewall-manual-instructions/configure-egress-route-table-2.png)

---

## Verification

After completing all steps, verify the configuration:

1. **Test East-West Connectivity:**
   - Connect to Spoke A EC2 instance via AWS Systems Manager Session Manager
   - Ping the Spoke B EC2 instance private IP
   - Traffic should flow through the firewall

2. **Test Egress Connectivity:**
   - From either spoke EC2 instance, test internet connectivity (e.g., `ping 8.8.8.8`)
   - Traffic should flow through the firewall to the egress VPC

3. **Review Firewall Logs:**
   - Navigate to **CloudWatch Logs**
   - Check the flow and alert log groups for traffic inspection events

## Traffic Flow Summary

**Before Configuration:**
- East-West: Spoke → TGW → Spoke (no inspection)
- Egress: Spoke → TGW → Egress VPC → Internet (no inspection)

**After Configuration:**
- East-West: Spoke → TGW → Firewall → TGW → Spoke (inspected)
- Egress: Spoke → TGW → Firewall → TGW → Egress VPC → Internet (inspected)

## Additional Resources

- [AWS Network Firewall Documentation](https://docs.aws.amazon.com/network-firewall/)
- [Transit Gateway-Attached Firewalls](https://docs.aws.amazon.com/network-firewall/latest/developerguide/tgw-firewall.html)
- [Pre-Deployed Architecture](../pre-deployed-tgw-attached-firewall/) - Fully automated deployment option
