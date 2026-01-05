<div align="center">
  
![AWS Organizations](https://img.icons8.com/color/96/amazon-web-services.png)

## Prevent Resource Creation in Unauthorized AWS Regions with SCPs

**Updated: January 2, 2026**

[![Follow @nicoleepaixao](https://img.shields.io/github/followers/nicoleepaixao?label=Follow&style=social)](https://github.com/nicoleepaixao)
[![Star this repo](https://img.shields.io/github/stars/nicoleepaixao/aws-scp-region-governance?style=social)](https://github.com/nicoleepaixao/aws-scp-region-governance)
[![Medium Article](https://img.shields.io/badge/Medium-12100E?style=for-the-badge&logo=medium&logoColor=white)](https://nicoleepaixao.medium.com/enforcing-region-based-governance-in-aws-with-scps-8b37e434805b)

</div>

---

<p align="center">
  <img src="img/aws-scp-restrict-regions.png" alt="SCP Architecture" width="1200">
</p>

## **Overview**

This project implements region-based governance for AWS Organizations using Service Control Policies (SCPs). The solution enforces regional restrictions at the organizational level, preventing resource creation in unauthorized regions regardless of IAM permissions. This approach ensures compliance, cost control, and security alignment across all accounts in your AWS Organization.

---

## **The Problem**

In multi-account AWS environments, unrestricted regional access creates significant operational and compliance challenges:

| **Challenge** | **Impact** |
|--------------|------------|
| **Uncontrolled Cloud Spend** | Resources in multiple regions increase costs unpredictably |
| **Infrastructure Sprawl** | Difficult to track and manage distributed resources |
| **Compliance Violations** | LGPD, GDPR, and data residency requirements breached |
| **Security Misalignment** | Regional security policies not consistently applied |
| **Audit Complexity** | Multiple regions complicate compliance reporting |
| **Operational Overhead** | Teams must monitor and secure more regions |

### **Why IAM Policies Are Not Enough**

| **Limitation** | **Problem** |
|----------------|------------|
| **User-Level Only** | IAM policies apply to specific users/roles, not accounts |
| **Bypassable** | New admin users or roles can bypass existing policies |
| **Not Inherited** | Each new principal needs policy attachment |
| **Root Account** | IAM policies don't restrict root account actions |
| **Service-Linked Roles** | AWS-managed roles may ignore IAM restrictions |

### **Real-World Scenario**

```text
Developer creates EC2 instance in eu-west-1 for "testing"
    ↓
Data residency violated (LGPD/GDPR)
    ↓
Security team discovers resource months later
    ↓
Compliance audit failure + unexpected costs
```

---

## **Solution: Service Control Policies (SCPs)**

### **Why SCPs Work**

SCPs operate at the AWS Organizations level and provide:

| **Advantage** | **Benefit** |
|---------------|------------|
| **Account-Level Enforcement** | Applies to ALL principals in the account |
| **Unbypas sable** | Even root account must comply |
| **Inherited** | Automatically applies to all users and roles |
| **Centralized** | Manage from single Organizations account |
| **Immediate** | No per-user policy updates needed |

---

## **How It Works**

### **Approved Regions**

This SCP enforces resource creation only in the following regions:

| **Region Code** | **Region Name** | **Use Case** |
|-----------------|----------------|--------------|
| `us-east-1` | N. Virginia | Primary production region |
| `us-west-2` | Oregon | Disaster recovery / secondary |
| `sa-east-1` | São Paulo | LGPD compliance / Latin America |

### **Policy Behavior**

```text
┌─────────────────────────────────────────────────────────┐
│            User Attempts Action in Region               │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │  Is region in list?   │
         │  [us-east-1,          │
         │   us-west-2,          │
         │   sa-east-1]          │
         └────────┬──────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
        ▼                   ▼
    ┌───────┐         ┌──────────┐
    │  YES  │         │    NO    │
    │ Allow │         │   DENY   │
    └───────┘         └──────────┘
        │                   │
        ▼                   ▼
   Action            Action Blocked
   Proceeds          (AccessDenied)
```

---

## **Architecture**

### **SCP Enforcement Flow**

<div align="center">

<p align="center">
  <img src="img/aws-scp-restrict-regions.png" alt="SCP Architecture" width="800">
</p>

</div>

---

## **SCP Policy**

### **Complete Policy JSON**

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

### **Policy Breakdown**

| **Element** | **Value** | **Purpose** |
|-------------|-----------|-------------|
| `Effect` | `Deny` | Blocks actions that match conditions |
| `Action` | `*` | Applies to all AWS service actions |
| `Resource` | `*` | Applies to all AWS resources |
| `Condition` | `StringNotEquals` | Checks if region is NOT in approved list |
| `aws:RequestedRegion` | Region list | Compares against allowed regions |

### **How the Condition Works**

```text
Logic: DENY if (aws:RequestedRegion NOT IN [us-east-1, us-west-2, sa-east-1])

Examples:
- User creates EC2 in us-east-1 → ✅ NOT DENIED (region in list)
- User creates RDS in eu-west-1 → ❌ DENIED (region not in list)
- User creates S3 in ap-south-1 → ❌ DENIED (region not in list)
```

---

## **Implementation Guide**

### **Prerequisites**

| **Requirement** | **Details** |
|-----------------|-------------|
| **AWS Organization** | Must have Organizations enabled |
| **Management Account Access** | Need permissions to create and attach SCPs |
| **SCP Feature Enabled** | SCPs must be enabled in Organizations |
| **Existing Accounts** | At least one member account or OU to attach policy |

### **Step 1: Enable Service Control Policies**

```bash
# Check if SCPs are enabled
aws organizations describe-organization

# Enable SCPs if needed
aws organizations enable-policy-type \
  --root-id r-xxxx \
  --policy-type SERVICE_CONTROL_POLICY
```

### **Step 2: Create the SCP**

**Option A: Via AWS Console**

1. Open the [AWS Organizations Console](https://console.aws.amazon.com/organizations/)
2. Navigate to **Policies** → **Service Control Policies**
3. Click **Create policy**
4. Name the policy: `AllowOnlySelectedRegions`
5. Paste the JSON policy from above
6. Add description: "Restricts resource creation to us-east-1, us-west-2, and sa-east-1"
7. Click **Create policy**

**Option B: Via AWS CLI**

```bash
# Save policy to file
cat > region-restriction-scp.json << 'EOF'
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
EOF

# Create the SCP
aws organizations create-policy \
  --name "AllowOnlySelectedRegions" \
  --description "Restricts resource creation to approved regions" \
  --type SERVICE_CONTROL_POLICY \
  --content file://region-restriction-scp.json
```

### **Step 3: Attach SCP to Target**

**To Organizational Unit (OU):**

```bash
# List OUs to find target
aws organizations list-organizational-units-for-parent \
  --parent-id r-xxxx

# Attach SCP to OU
aws organizations attach-policy \
  --policy-id p-xxxxxxxx \
  --target-id ou-xxxx-xxxxxxxx
```

**To Specific Account:**

```bash
# List accounts
aws organizations list-accounts

# Attach SCP to account
aws organizations attach-policy \
  --policy-id p-xxxxxxxx \
  --target-id 123456789012
```

### **Step 4: Verify FullAWSAccess is Attached**

```bash
# List policies attached to target
aws organizations list-policies-for-target \
  --target-id ou-xxxx-xxxxxxxx \
  --filter SERVICE_CONTROL_POLICY

# Expected output should include both:
# - FullAWSAccess (p-FullAWSAccess)
# - AllowOnlySelectedRegions (p-xxxxxxxx)
```

**⚠️ CRITICAL:** Both `FullAWSAccess` and `AllowOnlySelectedRegions` must be attached, or ALL actions will be denied.

---

## **Testing and Validation**

### **Test Scenarios**

| **Test** | **Region** | **Expected Result** |
|----------|-----------|-------------------|
| Create EC2 instance | us-east-1 | ✅ Success |
| Create S3 bucket | sa-east-1 | ✅ Success |
| Launch RDS database | us-west-2 | ✅ Success |
| Create VPC | eu-west-1 | ❌ AccessDenied |
| Create Lambda | ap-southeast-1 | ❌ AccessDenied |
| Create IAM role | Global service | ✅ Success (not region-specific) |

### **Test Commands**

```bash
# Test allowed region (should succeed)
aws ec2 run-instances \
  --region us-east-1 \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --count 1

# Test blocked region (should fail)
aws ec2 run-instances \
  --region eu-west-1 \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --count 1

# Expected error:
# An error occurred (UnauthorizedOperation) when calling the RunInstances operation:
# You are not authorized to perform this operation.
```

### **Validation Checklist**

- ✅ SCP created successfully in Organizations
- ✅ SCP attached to target OU or account
- ✅ FullAWSAccess policy also attached
- ✅ Test user can create resources in allowed regions
- ✅ Test user blocked from creating resources in other regions
- ✅ Root account also respects SCP restrictions
- ✅ Existing resources in other regions still accessible (read-only)

---

## **Expected Results**

### **After SCP Attachment**

| **Actor** | **Behavior** |
|-----------|-------------|
| **Admin Users** | Blocked from creating resources outside allowed regions |
| **IAM Roles** | Respect SCP regardless of role permissions |
| **Root Account** | Also restricted by SCP (cannot bypass) |
| **Service-Linked Roles** | AWS services respect regional restrictions |
| **CloudFormation** | Stack creation denied in unauthorized regions |

### **Resource Creation Matrix**

| **Action** | **us-east-1** | **eu-west-1** | **ap-south-1** |
|------------|-------------|-------------|--------------|
| EC2 Launch | ✅ Allowed | ❌ Denied | ❌ Denied |
| S3 Bucket | ✅ Allowed | ❌ Denied | ❌ Denied |
| RDS Create | ✅ Allowed | ❌ Denied | ❌ Denied |
| Lambda Deploy | ✅ Allowed | ❌ Denied | ❌ Denied |
| VPC Create | ✅ Allowed | ❌ Denied | ❌ Denied |

### **Global Services Behavior**

| **Service** | **Affected by SCP?** |
|-------------|-------------------|
| IAM | ❌ No (global service) |
| Route 53 | ❌ No (global service) |
| CloudFront | ❌ No (global service) |
| Organizations | ❌ No (management service) |
| EC2 | ✅ Yes (regional) |
| S3 | ✅ Yes (regional) |
| RDS | ✅ Yes (regional) |

---

## **Important Considerations**

### **Policy Logic**

| **Aspect** | **Detail** |
|------------|-----------|
| **Deny-Based** | SCP uses Deny effect, requires FullAWSAccess |
| **Explicit Deny** | Overrides any Allow in IAM policies |
| **Condition Check** | Evaluated for every API call |
| **Region-Specific** | Only applies to regional services |

### **Existing Resources**

```text
Question: What happens to resources already in unauthorized regions?

Answer: They remain accessible (read-only). The SCP only prevents 
new resource CREATION. Existing resources can be:
  ✅ Viewed/Described
  ✅ Modified (within their region)
  ✅ Deleted
  ❌ New resources cannot be created
```

### **Rollback Strategy**

**If Issues Occur:**

```bash
# 1. Detach the SCP immediately
aws organizations detach-policy \
  --policy-id p-xxxxxxxx \
  --target-id ou-xxxx-xxxxxxxx

# 2. Verify detachment
aws organizations list-policies-for-target \
  --target-id ou-xxxx-xxxxxxxx \
  --filter SERVICE_CONTROL_POLICY

# 3. Test that actions work again
aws ec2 describe-regions --region eu-west-1

# 4. If needed, delete the SCP
aws organizations delete-policy \
  --policy-id p-xxxxxxxx
```

### **Monitoring and Compliance**

```bash
# CloudTrail query to find denied actions
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --max-results 50 \
  --query 'Events[?ErrorCode==`UnauthorizedOperation`]'

# CloudWatch Insights query
fields @timestamp, userIdentity.principalId, requestParameters.region, errorCode
| filter errorCode = "UnauthorizedOperation"
| filter requestParameters.region not in ["us-east-1", "us-west-2", "sa-east-1"]
| sort @timestamp desc
```

---

## **Advanced Scenarios**

### **Adding More Regions**

To allow additional regions, update the policy:

```json
{
  "aws:RequestedRegion": [
    "us-east-1",
    "us-west-2",
    "sa-east-1",
    "eu-central-1"  // Added Frankfurt
  ]
}
```

Then update the existing policy:

```bash
aws organizations update-policy \
  --policy-id p-xxxxxxxx \
  --content file://updated-region-restriction-scp.json
```

### **Region-Specific Service Exemptions**

To allow specific services in additional regions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlySelectedRegions",
      "Effect": "Deny",
      "NotAction": [
        "cloudfront:*",
        "iam:*",
        "route53:*",
        "support:*"
      ],
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

### **Phased Rollout**

**Recommended Approach:**

| **Phase** | **Action** | **Validation Period** |
|-----------|-----------|---------------------|
| **1. Test** | Attach to sandbox OU | 1 week |
| **2. Dev** | Attach to development accounts | 2 weeks |
| **3. Staging** | Attach to staging OU | 1 week |
| **4. Prod** | Attach to production OU | Ongoing monitoring |

---

## **Troubleshooting**

### **Problem: All Actions Denied**

**Cause:** Only the deny-based SCP is attached, without FullAWSAccess.

**Solution:**
```bash
# Attach FullAWSAccess policy
aws organizations attach-policy \
  --policy-id p-FullAWSAccess \
  --target-id ou-xxxx-xxxxxxxx
```

### **Problem: Global Services Blocked**

**Cause:** SCP might be too restrictive for global services.

**Solution:** Use `NotAction` to exempt global services:
```json
{
  "NotAction": [
    "iam:*",
    "route53:*",
    "cloudfront:*"
  ]
}
```

### **Problem: Can't Detach SCP**

**Cause:** Policy might be attached to multiple targets.

**Solution:**
```bash
# List all targets with this policy
aws organizations list-targets-for-policy \
  --policy-id p-xxxxxxxx

# Detach from each target
aws organizations detach-policy \
  --policy-id p-xxxxxxxx \
  --target-id <each-target-id>
```

---

## **Use Cases**

| **Use Case** | **Application** |
|--------------|-----------------|
| **Cost Control** | Prevent resource sprawl in expensive regions |
| **Compliance (LGPD)** | Ensure Brazilian data stays in sa-east-1 |
| **Compliance (GDPR)** | Restrict EU data to EU regions |
| **Security Baseline** | Enforce approved regions for security controls |
| **Disaster Recovery** | Limit regions to primary + DR locations |
| **Licensing** | Comply with software licensing region restrictions |
| **Network Architecture** | Align with existing VPN/Direct Connect setup |
| **Audit Readiness** | Simplify compliance reporting to specific regions |

---

## **Best Practices**

### **SCP Management**

| **Practice** | **Recommendation** |
|--------------|-------------------|
| **Version Control** | Store SCPs in Git repository |
| **Change Management** | Require approval before SCP updates |
| **Testing** | Always test in sandbox/dev first |
| **Documentation** | Document business justification for each region |
| **Review Cadence** | Quarterly review of approved regions |
| **Exception Process** | Define process for temporary region access |

### **Communication Plan**

```text
Before Implementing:
1. Announce changes to all teams (2 weeks notice)
2. Provide list of approved regions
3. Share migration timeline for existing resources
4. Set up office hours for questions
5. Create runbook for common scenarios

During Implementation:
1. Start with test accounts
2. Monitor CloudTrail for denied actions
3. Address issues before production rollout
4. Maintain communication channel

After Implementation:
1. Share metrics on compliance improvement
2. Document lessons learned
3. Update onboarding materials
4. Regular compliance reports
```

---

## **Full Article on Medium**

For a comprehensive explanation of the business context, implementation steps, rollback considerations, and technical background, read the complete story:

**[Enforcing Region-Based Governance in AWS with SCPs](https://nicoleepaixao.medium.com/enforcing-region-based-governance-in-aws-with-scps-8b37e434805b)**

---

## **Additional Resources**

### **Official Documentation**

- [AWS Organizations SCPs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) - Complete SCP reference
- [SCP Syntax](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_syntax.html) - Policy language details
- [AWS Regions and Endpoints](https://docs.aws.amazon.com/general/latest/gr/rande.html) - All AWS region codes
- [SCP Examples](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_examples.html) - AWS-provided examples
- [Global Condition Keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html) - Available condition keys

### **Related Resources**

- [AWS Control Tower](https://aws.amazon.com/controltower/) - Automated governance setup
- [AWS Config Rules](https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config.html) - Continuous compliance monitoring
- [CloudTrail](https://aws.amazon.com/cloudtrail/) - Track SCP-denied actions

---

## **Future Enhancements**

| **Feature** | **Description** | **Status** |
|-------------|-----------------|------------|
| **Terraform Module** | IaC implementation for SCP | Planned |
| **Automated Testing** | Lambda function to validate SCP | In Development |
| **Metrics Dashboard** | CloudWatch dashboard for denied actions | Planned |
| **Exception Workflow** | Automated approval for temporary access | Future |
| **Compliance Reports** | Automated PDF reports for audits | Planned |
| **Multi-Policy Management** | Template for layered SCPs | Future |

---

## **Connect & Follow**

Stay updated with AWS governance, security automation, and best practices:

<div align="center">

[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/nicoleepaixao)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?logo=linkedin&logoColor=white&style=for-the-badge)](https://www.linkedin.com/in/nicolepaixao/)
[![Medium](https://img.shields.io/badge/Medium-12100E?style=for-the-badge&logo=medium&logoColor=white)](https://medium.com/@nicoleepaixao)

</div>

---

## **Disclaimer**

This SCP policy is provided as a reference implementation. Service Control Policies affect all users and services in the target accounts. Always test SCPs in non-production environments before applying to production accounts. Ensure FullAWSAccess policy is attached alongside deny-based SCPs. Consult official AWS documentation and your organization's governance policies before implementation.

---

<div align="center">

**Happy governing your AWS infrastructure!**

*Document last updated: January 2, 2026*

</div>