# AWS Network Firewall - Getting Started Policy

A CloudFormation template that deploys a Network Firewall policy in **monitor mode**: all 15 recommended AWS managed rule groups plus a custom Suricata rule group, configured to log detections without blocking. The goal is to let the firewall observe your traffic for a period of time so you have a clear picture of what is flowing through it before enabling enforcement.

Once you understand your traffic patterns, you transition the policy to enforcement mode by removing the `DROP_TO_ALERT` overrides from managed rule groups and adding the drop default action.

## What the template deploys

| Resource | Configuration |
|---|---|
| Firewall policy | `STRICT_ORDER`, stream exception policy `REJECT`, TCP idle timeout 350s, `HOME_NET` set to all RFC 1918 ranges |
| Default action | `aws:alert_established_app_layer_to_server` (logs without blocking) |
| Active Threat Defense | AttackInfrastructureStrictOrder (priority 1, `DROP_TO_ALERT`) |
| Domain/IP reputation | 4 rule groups (priorities 2-5, `DROP_TO_ALERT`) |
| Threat signatures | 10 rule groups (priorities 6-15, `DROP_TO_ALERT`) |
| Custom rule group | HOME_NET validation, plaintext HTTP detection, east-west monitoring, inbound monitoring (priority 100, 200 capacity) |

Total: 16 rule group references, 29,200 of 30,000 default capacity used.

## Prerequisites

- An existing AWS Network Firewall (the template creates the policy only; associate it with your firewall after deployment)
- For the recommended deployment architecture, see the other templates in this repository

## Deployment

1. Open the [CloudFormation console](https://console.aws.amazon.com/cloudformation/) in the region where your firewall is deployed.
2. Choose **Create stack > With new resources**.
3. Upload `getting-started-policy.yaml` and proceed through the wizard.
4. Associate the created policy with your existing Network Firewall.

**After deployment:** Submit a [Service Quotas increase request](https://console.aws.amazon.com/servicequotas/home/services/network-firewall/quotas) to raise your stateful rule capacity to 50,000. This lets you add additional managed rule groups and larger custom rule groups.

## Transitioning to enforcement mode

After a monitoring period (typically 1-2 weeks), transition to enforcement:

1. Remove the `Override` block from each managed rule group reference to restore native drop/reject actions.
2. Add `aws:drop_established_app_layer_to_server` alongside the existing alert default action.
3. Add domain allowlist pass rules for the destinations your workloads legitimately need.

## Additional resources

- [AWS Network Firewall Best Practices Guide](https://github.com/aws/aws-security-services-best-practices) — detailed guidance on every policy setting in this template
- [AWS Network Firewall documentation](https://docs.aws.amazon.com/network-firewall/latest/developerguide/)
