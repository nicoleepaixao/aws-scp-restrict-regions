# Enforcing Region-Based Governance in AWS with SCPs

## The Problem

In multi-account AWS environments, it's easy for teams to spin up resources in unintended regions. This creates challenges like:

- Increased and unpredictable cloud spend
- Difficulty tracking distributed infrastructure
- Compliance risks (e.g. LGPD, GDPR)
- Security misalignment with regional policies

IAM policies are not enough, since they apply only to specific users or roles. If someone creates a new admin user, they can bypass those policies completely.

This project solves the problem by enforcing region-based restrictions using AWS Service Control Policies (SCPs), ensuring that resources can only be created in selected regions, regardless of IAM privileges.

## How It Works

This SCP prevents the creation of any AWS resource outside the following approved regions:

- `us-east-1` (N. Virginia)
- `us-west-2` (Oregon)
- `sa-east-1` (São Paulo)

All actions attempted in any other region will be blocked at the organizational level.

### SCP Policy JSON

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlySelectedRegions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2",
            "sa-east-1"
          ]
        }
      }
    }
  ]
}
```
### How to Use
1. Open the AWS Organizations console
2. Go to Policies > Service Control Policies
3. Click Create policy
4. Name it AllowOnlySelectedRegions
5. Paste the policy JSON from this repo
6. Attach the SCP to the desired account or Organizational Unit (OU)
7. Ensure the policy FullAWSAccess is also attached to allow valid actions within permitted regions

❗ Important: Always attach an Allow SCP (like FullAWSAccess) together with this deny-based policy, or no actions will be permitted at all.

### Expected Results
Once attached:
- Users (including Admins and root) will be blocked from creating or modifying resources in disallowed regions
- All IAM roles will respect the SCP regardless of their own permissions
- Only resources in the allowed regions (us-east-1, us-west-2, sa-east-1) will be created or modified

### Full Article on Medium
For a complete explanation of the business context, implementation steps, rollback considerations, and technical background, check out the full article:

[Enforcing Region-Based Governance in AWS with SCPs](https://nicoleepaixao.medium.com/enforcing-region-based-governance-in-aws-with-scps-8b37e434805b).
