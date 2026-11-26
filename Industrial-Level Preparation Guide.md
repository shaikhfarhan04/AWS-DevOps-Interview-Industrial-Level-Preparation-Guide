# **Cybage AWS DevOps Interview – Industrial-Level Preparation Guide**

Based on the job description, this is not a typical AWS DevOps interview focused on EC2, S3, Jenkins, Docker, and Kubernetes.

**This is primarily a Senior AWS Cloud Governance, Security Automation, Infrastructure as Code (IaC), and Multi-Account Platform Engineering role.**

The most important thing to understand is the business problem behind the requirement:

Cybage has 40+ AWS accounts. Infrastructure and configurations were deployed manually through the AWS Console. They now want to modernize the environment, introduce Terraform, automate security governance, establish centralized monitoring, and implement automated remediation without impacting existing production workloads.

You should prepare to explain how you would transform this environment from manually managed AWS infrastructure into a centrally governed, automated, secure, and observable AWS platform.

I'll structure your preparation around a real enterprise implementation rather than isolated AWS definitions.

# **1. Understand the actual project scenario**

## **Existing architecture – before modernization**

## **Current AWS environment**

**40+ AWS Accounts**

Development • QA • UAT • Production • Shared Services

**Manual EC2**

Console provisioning

**Manual IAM**

Users and policies

**Manual VPC**

Networking and security groups

**Manual Security**

Guardrails and logging

**Problems**

- Configuration drift
- Inconsistent security policies
- No centralized enforcement
- Manual remediation
- Difficult auditing
- No standardized infrastructure deployment

## **Target architecture – after modernization**

## **Centralized AWS governance platform**

**GitLab – Central Infrastructure Repository**

Terraform • CloudFormation • Python • Cloud Custodian • Policy-as-code

**GitLab CI/CD Pipeline**

Validate → Security Scan → Plan → Approval → Apply → Verify

**AWS Organizations / Control Tower**

Security OU

Infrastructure OU

Production OU

Non-Production OU

**40+ AWS Accounts**

Centralized identity • Security • Networking • Logging • Governance

**Detection and remediation**

CloudTrail • EventBridge • Config • Lambda • Cloud Custodian

**Monitoring**

CloudWatch • Security Hub • Grafana • Alerts

**The key architectural principle:** Terraform manages the desired infrastructure state, AWS Organizations provides organization-wide governance, and event-driven automation detects and remediates deviations.

These are complementary systems, not replacements for each other.

# **2. How to approach the modernization of 40+ AWS accounts**

This is likely to be one of the most important interview questions.

**Question: How would you migrate 40+ manually configured AWS accounts to Terraform without impacting production?**

Your answer should demonstrate a controlled migration strategy.

## **Phase 1 – AWS account discovery and assessment**

First, I would not start writing Terraform immediately.

I would inventory the existing environment.

| Area       | What I would investigate                    |
| ---------- | ------------------------------------------- |
| IAM        | Users, roles, policies, trust relationships |
| EC2        | Instances, AMIs, EBS, security groups       |
| Networking | VPCs, subnets, route tables, NAT, VPN       |
| S3         | Buckets, encryption, versioning, policies   |
| Security   | GuardDuty, Security Hub, Config             |
| Logging    | CloudTrail, CloudWatch, retention           |
| Identity   | IAM Identity Center, SAML providers         |
| Governance | SCPs, OUs, delegated administrators         |

Tools:

- AWS Config – resource inventory and configuration history.
- AWS Resource Explorer – discover resources.
- AWS CLI – export and inspect configurations.
- AWS CloudTrail – understand historical API changes.
- Terraform import – bring existing resources under Terraform management.

Deliverable: An account-by-account inventory and dependency map.

## **Phase 2 – Establish Terraform architecture**

I would create a centralized GitLab repository.

Example enterprise repository:

```
aws-cloud-platform/
│
├── modules/
│   ├── iam/
│   ├── vpc/
│   ├── security-group/
│   ├── s3/
│   ├── cloudtrail/
│   ├── guardduty/
│   ├── aws-config/
│   └── eventbridge/
│
├── accounts/
│   ├── security/
│   ├── shared-services/
│   ├── development/
│   ├── testing/
│   └── production/
│
├── governance/
│   ├── scp/
│   ├── config-rules/
│   ├── cloud-custodian/
│   └── security-policies/
│
├── environments/
│   ├── dev/
│   ├── qa/
│   ├── uat/
│   └── prod/
│
├── pipelines/
│   └── gitlab-ci.yml
│
└── README.md
```

### **Important Terraform concepts**

You must understand these in depth:

1. Terraform modules and reusable architecture.
2. Remote state in S3.
3. State locking using S3 lockfiles or an appropriate supported locking mechanism.
4. State isolation for different accounts and environments.
5. Provider aliases for cross-account deployment.
6. IAM AssumeRole for secure cross-account access.
7. Importing existing resources.
8. Terraform plan and apply.
9. State drift detection.
10. Lifecycle rules and resource protection.
11. Terraform version and provider version pinning.

**Critical interview point:** Do not simply run `terraform apply` against manually created production infrastructure. First import resources into Terraform state, reconcile the configuration with actual AWS settings, review the plan, and ensure the plan does not unintentionally replace or destroy resources.

## **Phase 3 – Import existing AWS resources**

For example, suppose a production S3 bucket was created manually.

You create its Terraform resource definition:

```
resource "aws_s3_bucket" "production_logs" {
  bucket = "company-production-logs"
}
```

Then import the existing bucket:

```
terraform import \
  aws_s3_bucket.production_logs \
  company-production-logs
```

Next:

```
terraform plan
```

You compare the actual configuration with your Terraform configuration.

The objective is to achieve a clean plan without unintended changes.

For newer Terraform versions, you should also understand declarative import blocks:

```
import {
  to = aws_s3_bucket.production_logs
  id = "company-production-logs"
}
```

**Important:** Terraform import associates an existing resource with Terraform state. It does not automatically generate a complete, correct configuration or make the infrastructure safe to change.

## **Phase 4 – Multi-account Terraform deployment**

A central pipeline should assume a dedicated deployment role in each target account.

For example:

```
provider "aws" {
  region = "ap-south-1"

  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformExecutionRole"
  }
}
```

For multiple accounts, you can use provider aliases:

```
provider "aws" {
  alias  = "dev"
  region = "ap-south-1"

  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformExecutionRole"
  }
}

provider "aws" {
  alias  = "prod"
  region = "ap-south-1"

  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformExecutionRole"
  }
}
```

For an enterprise with 40+ accounts, I would generally use a reusable account deployment pattern rather than maintaining 40 separate copies of the same Terraform configuration.

The pipeline should:

1. Identify the target account.
2. Authenticate using GitLab OIDC.
3. Assume the account-specific deployment role.
4. Execute Terraform validation.
5. Generate a plan.
6. Store the plan as an artifact.
7. Obtain production approval.
8. Apply the approved plan.
9. Verify deployment and record the result.

### **Terraform remote state architecture**

**Central Terraform State Account**

Dedicated S3 backend bucket

Account 001 state

Account 002 state

Account 003 state

Account 040 state

Separate state keys and access permissions for independent account/environment stacks.

Example:

```
terraform {
  backend "s3" {
    bucket       = "company-terraform-state"
    key          = "production/network/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

Enable S3 versioning, restrict state access, and protect encryption keys. Terraform state can contain sensitive information.

Note that current Terraform S3 backend documentation supports native S3 lockfiles through `use_lockfile`; older DynamoDB-based locking configurations are deprecated.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.hashicorp.com\&sz=32)

HashiCorp Developer



# **3. AWS Organizations – Your highest-priority interview topic**

Very high priority

The job description explicitly says IAM and Organizations are the primary requirements.

You should be able to explain how you would govern 40+ AWS accounts centrally.

## **3.1 What is AWS Organizations?**

AWS Organizations is a service for centrally managing multiple AWS accounts.

It provides:

- Organizational Units (OUs).
- Service Control Policies (SCPs).
- Centralized account management.
- Consolidated billing.
- Delegated administration.
- Integration with AWS security services.

### **Example OU architecture**

**AWS Organization**

Management Account

**Security OU**

Security Tooling

Log Archive

**Infrastructure OU**

Network

Shared Services

**Production OU**

Production Account 1

Production Account 2

**Non-Production OU**

Dev / QA

UAT / Testing

In production, OU design should be based on security and workload requirements rather than simply the organizational structure of the company.

AWS recommends separating foundational accounts such as security, logging, and infrastructure accounts from workload accounts.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

AWS Organizations+1



## **3.2 What is an SCP?**

An SCP is a Service Control Policy.

It defines the maximum permissions available to IAM principals in member accounts of an AWS Organization.

**Most important interview statement:**

An SCP does not grant permissions. It establishes permission boundaries at the organization level.

Example: Prevent users from disabling CloudTrail.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyCloudTrailChanges",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*"
    }
  ]
}
```

You can attach this SCP to the appropriate OU.

Other common SCP use cases:

| Guardrail                 | Purpose                                      |
| ------------------------- | -------------------------------------------- |
| Restrict regions          | Prevent deployment in unauthorized regions   |
| Protect CloudTrail        | Prevent disabling audit logging              |
| Protect Config            | Prevent disabling configuration recording    |
| Restrict public S3        | Prevent unsafe bucket configurations         |
| Restrict IAM              | Prevent unauthorized privilege escalation    |
| Protect security services | Prevent disabling GuardDuty and Security Hub |

Important exceptions and limitations:

- SCPs do not apply to the management account.
- SCPs do not grant permissions.
- SCPs do not restrict service-linked roles.
- An explicit deny in an applicable SCP overrides an allow from an IAM policy.

## **3.3 IAM policy evaluation – expert-level preparation**

This is a likely technical interview discussion.

Understand these policy types:

| Policy type         | Purpose                                           |
| ------------------- | ------------------------------------------------- |
| Identity-based      | Permissions assigned to users, groups, roles      |
| Resource-based      | Permissions attached to S3, SQS, KMS, etc.        |
| Permission boundary | Maximum permissions an identity policy can grant  |
| Session policy      | Restricts permissions for an assumed-role session |
| SCP                 | Organization-level maximum permissions            |
| Trust policy        | Determines who can assume a role                  |

### **Example**

A developer has this IAM policy:

```
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

But their permission boundary only allows:

```
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:ListBucket"
  ],
  "Resource": "*"
}
```

The developer cannot perform `s3:DeleteObject` just because the identity policy allows it.

Why?

Because the permission boundary restricts the maximum permissions available.

### **Remember the evaluation sequence**

1. Start with implicit deny.
2. Check applicable policy allows and explicit denies.
3. Evaluate permission boundaries and session policies.
4. Evaluate SCP restrictions.
5. Consider resource-based policies and service-specific authorization rules.
6. Explicit deny wins over an allow.

Do not describe this as one simple universal intersection formula: resource policies, role sessions, KMS key policies, and certain principal types have additional evaluation nuances.

## **3.4 Cross-account IAM role assumption**

This is extremely important for a 40+ account environment.

Example:

- Central Terraform account: `111111111111`
- Production AWS account: `222222222222`

The central account needs to deploy infrastructure in production.

Architecture:

**Central GitLab Pipeline**

AssumeRole / STS

**Production Account**

TerraformExecutionRole

**Terraform deployment**

VPC • IAM • S3 • Security • Monitoring

Trust policy in the target account:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

The source principal must also have permission to call `sts:AssumeRole` on that role.

For GitLab, I would prefer OIDC federation to AWS STS rather than long-lived access keys. Restrict the OIDC trust conditions to the correct GitLab project, branch or protected ref, and deployment environment.

# **4. Event-driven compliance and automated remediation**

Very high priority

This is another major responsibility in the job description.

You must understand the difference between:

- Detecting a security issue.
- Reporting a security issue.
- Automatically remediating a security issue.
- Enforcing a security policy.

## **4.1 Understand the event-driven architecture**

Consider this real-world example:

A developer accidentally creates an S3 bucket with public access.

What should happen?

1. AWS Config detects that the bucket is non-compliant.
2. An event is generated.
3. EventBridge receives the relevant event.
4. A Lambda function or remediation workflow is triggered.
5. The workflow blocks public access.
6. The result is logged.
7. An alert is sent to the security team.
8. Compliance status is updated.

## **Event-driven remediation workflow**

**Developer creates public S3 bucket**

**AWS Config**

Detects non-compliance

**Amazon EventBridge**

Matches the compliance event

**AWS Lambda / SSM Automation**

Executes remediation

**S3 public access blocked**

Remediated

CloudWatch logs, SNS/Jira notifications and audit records capture the outcome.

AWS Security Hub findings can also be delivered to EventBridge and used to trigger Lambda, Step Functions, notifications, and other response actions.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

AWS Security Hub+1



## **4.2 Detect-only mode versus enforcement mode**

The requirement explicitly mentions:

"Rules start in detect-only mode and move to enforcement."

This is a very important operational concept.

Suppose the organization wants to prevent public S3 buckets.

**Stage 1 – Detect-only**

- Deploy the AWS Config rule.
- Run Cloud Custodian in audit mode.
- Identify existing non-compliant resources.
- Generate reports.
- Notify owners.
- Do not modify production resources.

**Stage 2 – Analyze**

- Identify false positives.
- Identify legitimate business exceptions.
- Check whether applications depend on the current configuration.
- Obtain application-owner approval.

**Stage 3 – Remediate**

- Enable remediation for one guardrail.
- Test on a non-production account.
- Monitor the remediation.
- Verify the resulting configuration.
- Document any exceptions.

**Stage 4 – Enforce**

- Enable automatic remediation for approved cases.
- Introduce preventive controls where appropriate.
- Apply organization-wide only after successful testing.

## **Guardrail maturity model**

1. **Detect**

   Audit mode

   Identify policy violations without changing resources.
2. **Analyze**

   Validate findings, false positives, exceptions and business impact.
3. **Remediate**

   Enable controlled remediation and verify outcomes.
4. **Enforce**

   Enable approved preventive and automatic controls.

This is how you avoid breaking existing workloads.

## **4.3 AWS Config vs Cloud Custodian**

This distinction is essential.

| Feature      | AWS Config                                 | Cloud Custodian                        |
| ------------ | ------------------------------------------ | -------------------------------------- |
| Purpose      | Configuration compliance                   | Cloud governance and policy automation |
| Rules        | Managed and custom rules                   | YAML policies                          |
| Detection    | Resource configuration                     | Resource filters and events            |
| Remediation  | SSM Automation or other configured actions | Lambda, event modes and actions        |
| Event-driven | Supported                                  | Supported                              |
| Reporting    | Compliance dashboard                       | Policy execution and resource reports  |
| Main use     | Configuration auditing                     | Flexible governance automation         |

### **Example AWS Config rule**

Detect S3 buckets without encryption.

```
resource "aws_config_config_rule" "s3_encryption" {
  name = "s3-bucket-server-side-encryption"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
  }
}
```

### **Example Cloud Custodian policy**

Suppose you want to identify EC2 instances that do not have an approved tag.

```
policies:
  - name: ec2-missing-owner-tag
    resource: aws.ec2

    filters:
      - "tag:Owner": absent

    actions:
      - type: notify
        template: default
        subject: "EC2 missing owner tag"
```

This is a detection/notification example. It does not terminate or modify instances.

A production Cloud Custodian policy can also use CloudTrail event mode to react to resource creation or modification events.

For example, you can detect an EC2 instance launched without the required tags and trigger a notification or controlled remediation workflow.

Cloud Custodian supports CloudTrail-based event modes that execute policies through Lambda.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.cloudcustodian.io\&sz=32)

Cloud Custodian documentation



### **Interview question: Why do we need both AWS Config and Cloud Custodian?**

My answer:

AWS Config provides a native AWS configuration compliance engine with managed and custom rules, compliance history, and remediation integration.

Cloud Custodian provides flexible policy-as-code capabilities to filter, evaluate, notify, and remediate AWS resources using YAML policies.

I would use AWS Config for native compliance guardrails and Cloud Custodian for more complex resource governance and event-driven automation. They can complement each other, but I would avoid implementing the same remediation independently in both systems.

# **5. AWS security services – know their roles**

The following services are directly relevant to this interview.

| Service             | Industrial purpose                                        |
| ------------------- | --------------------------------------------------------- |
| IAM                 | Identity and permission management                        |
| AWS Organizations   | Multi-account governance                                  |
| SCP                 | Organization-level permission guardrails                  |
| IAM Access Analyzer | Identify unintended external access and validate policies |
| GuardDuty           | Threat detection                                          |
| Security Hub        | Centralized security findings and security posture        |
| AWS Config          | Configuration compliance                                  |
| CloudTrail          | API activity and audit logging                            |
| KMS                 | Encryption key management                                 |
| AWS WAF             | Web application protection                                |
| EventBridge         | Event routing and automation                              |
| Lambda              | Serverless remediation                                    |
| SSM Automation      | Operational remediation workflows                         |

### **Important interview scenario: GuardDuty detects suspicious activity**

Question: What happens if GuardDuty detects suspicious API activity in one of 40 accounts?

A good answer:

1. GuardDuty detects the suspicious activity.
2. The finding is centralized in the delegated Security Hub/security administration account, where configured.
3. EventBridge identifies the relevant finding.
4. A Lambda or Step Functions workflow evaluates the finding.
5. Depending on severity and confidence, it can create an incident, notify the SOC, or isolate a resource.
6. All actions are recorded in CloudTrail and CloudWatch.
7. The security team verifies the incident and closes the finding.

Important: Do not automatically isolate every resource based only on a low-confidence finding. Remediation should depend on finding type, severity, confidence, and business impact.

# **6. GitLab CI/CD – Terraform automation**

Very high priority

You need to understand how Terraform is automatically deployed across multiple AWS accounts.

## **Enterprise pipeline**

## **GitLab Terraform pipeline**

**1**

**Developer creates feature branch**

Terraform code changes

**2**

**Merge request**

Code review and approval

**3**

**Terraform validation**

fmt, init, validate

**4**

**Security scanning**

Checkov / tfsec / policy checks

**5**

**Terraform plan**

Review infrastructure changes

**6**

**Approval**

Protected production deployment

**7**

**Terraform apply**

Deploy through AWS AssumeRole

**8**

**Post-deployment verification**

Drift and compliance checks

## **Sample .gitlab-ci.yml**

This is a simplified foundation you can explain in an interview.

```

stages:
  - validate
  - security
  - plan
  - deploy

variables:
  TF_ROOT: "terraform"

terraform_validate:
  stage: validate
  image: hashicorp/terraform:latest
  script:
    - cd $TF_ROOT
    - terraform fmt -check -recursive
    - terraform init -backend=false
    - terraform validate

terraform_security:
  stage: security
  image: bridgecrew/checkov
  script:
    - checkov -d $TF_ROOT

terraform_plan:
  stage: plan
  image: hashicorp/terraform:latest
  script:
    - cd $TF_ROOT
    - terraform init
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - terraform/tfplan

terraform_apply:
  stage: deploy
  image: hashicorp/terraform:latest
  when: manual
  script:
    - cd $TF_ROOT
    - terraform init
    - terraform apply -auto-approve tfplan
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

```

For a real enterprise implementation, improve this example:

- Pin Terraform and scanner image versions instead of using `latest`.
- Authenticate using GitLab OIDC.
- Restrict deployment roles using IAM trust conditions.
- Use separate state for each account/environment.
- Use the exact saved plan artifact that was approved.
- Protect production environments.
- Prevent unapproved branches from deploying.
- Ensure apply jobs use the same Terraform version and provider lockfile as plan jobs.
- Add plan expiry and stale-plan handling.
- Include rollback or recovery procedures.

GitLab supports OIDC-based AWS authentication without storing long-lived AWS credentials, and protected-environment deployment approvals can gate production deployments.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.gitlab.com\&sz=32)

GitLab Docs+1



# **7. AWS monitoring, Lambda runtime EOL and Grafana**

The job description specifically mentions:

"Build alerting for platform-health risks, such as Lambda functions on runtimes approaching end of life, and publish it to dashboards, typically by integrating AWS to Grafana."

This is a practical automation use case. Prepare it properly.

## **7.1 Problem statement**

Suppose 40 AWS accounts have 500 Lambda functions.

Some functions might run on:

- Python 3.9
- Python 3.10
- Node.js 18
- Other older runtimes.

AWS periodically deprecates Lambda runtimes.

You need to identify functions running on outdated or soon-to-be-deprecated runtimes and notify the respective application teams.

## **7.2 Architecture**

**40+ AWS Accounts**

Lambda functions across accounts and regions

**Scheduled EventBridge Rule**

Daily or weekly inventory scan

**Python Lambda**

AssumeRole into accounts and list Lambda configurations

**Runtime compliance evaluation**

Compare runtimes against AWS lifecycle information

**CloudWatch**

Metrics and logs

**Grafana**

Dashboards

**SNS / Jira / Email alerts**

Notify application owners to upgrade runtimes

## **7.3 Python implementation example**

This example checks Lambda runtimes in one account. In production, extend it to multiple accounts and regions using AssumeRole.

```
import boto3def lambda_handler(event, context):    client = boto3.client("lambda")    paginator = client.get_paginator("list_functions")    deprecated_candidates = []    # Illustrative list only.    # Maintain this from AWS runtime lifecycle data.    outdated_runtimes = [        "python3.9",        "python3.10",        "nodejs18.x"    ]    for page in paginator.paginate():        for function in page["Functions"]:            runtime = function.get("Runtime")            if runtime in outdated_runtimes:                deprecated_candidates.append({                    "FunctionName": function["FunctionName"],                    "Runtime": runtime,                    "LastModified": function["LastModified"]                })    print("Runtime review required:")    for function in deprecated_candidates:
```

Important: These runtimes are illustrative review candidates, not a verified current EOL list. In a real implementation, maintain an authoritative runtime lifecycle dataset, including deprecation and function-creation-block dates. Do not automatically update production runtimes without testing application dependencies.

### **Grafana integration**

Two common approaches:

1. Amazon Managed Grafana integrated with CloudWatch and other supported data sources.
2. Self-managed Grafana integrated with CloudWatch or a metrics backend.

Useful dashboard metrics:

| Metric                       | Example |
| ---------------------------- | ------- |
| Total Lambda functions       | 500     |
| Runtime review required      | 35      |
| Accounts affected            | 12      |
| Critical runtime issues      | 8       |
| Remediation completed        | 20      |
| Pending application upgrades | 15      |

This demonstrates that you understand not just the AWS service but also the platform engineering requirement.

# **8. Cloud hygiene – practical interview scenarios**

Cloud hygiene means regularly identifying and removing unnecessary, risky, or outdated cloud configurations.

The job description gives three explicit examples.

## **8.1 Retiring unused SAML identity providers**

First understand SAML.

SAML is an authentication protocol that enables Single Sign-On between an identity provider and AWS.

For example:

**Corporate Identity Provider**

Microsoft Entra ID / Okta

SAML authentication assertion

**AWS IAM SAML Identity Provider**

**IAM Role**

AssumeRoleWithSAML

**AWS Console Access**

Suppose an organization has migrated from an old identity provider to Microsoft Entra ID.

But the old SAML identity provider still exists in 40 accounts.

How would you remove it?

My approach:

1. Inventory SAML providers across all accounts.
2. Identify the provider's ARN and certificate expiry.
3. Review IAM roles referencing that provider in their trust policies.
4. Verify with the identity team whether the provider is still being used.
5. Identify dependencies and exceptions.
6. Remove obsolete trust relationships and providers through a controlled change.
7. Verify that existing SSO access continues working.

**Important:** Never delete a SAML provider simply because it appears old or unused. Validate its trust relationships and actual usage first.

## **8.2 Security group review**

This is a very common AWS security scenario.

Example of a risky rule:

```
Inbound:
Port: 22
Protocol: TCP
Source: 0.0.0.0/0
```

This exposes SSH access to the internet.

Another example:

```
Port: 3389
Protocol: TCP
Source: 0.0.0.0/0
```

This exposes RDP.

### **Enterprise automation approach**

1. Inventory security groups across all accounts.
2. Identify unrestricted inbound access.
3. Categorize rules by severity.
4. Find attached EC2 instances, ENIs, load balancers and other dependencies.
5. Contact application owners.
6. Create remediation tickets.
7. Remove or restrict approved risky rules.
8. Re-scan and verify compliance.

Do not blindly delete security group rules from production. First understand dependencies and business requirements.

## **8.3 Log-retention policies**

Suppose different AWS accounts have different CloudWatch log retention periods.

Some have indefinite retention; others have 7 days or 30 days.

You want to standardize retention.

Example Terraform:

```
resource "aws_cloudwatch_log_group" "application" {
  name              = "/company/application"
  retention_in_days = 90

  kms_key_id = var.kms_key_arn

  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}
```

For existing log groups, inventory the current retention first. Then apply the appropriate retention standard based on regulatory, audit, operational, and cost requirements.

Remember that CloudTrail retention and CloudWatch log-group retention are separate concerns. Also consider centralized S3 archival and lifecycle policies.

# **9. VPC and AWS Cloud WAN**

The requirement specifically lists VPC and Cloud WAN.

You should be able to explain the difference.

## **VPC**

Amazon VPC provides isolated virtual networking within an AWS Region.

Know:

- Public and private subnets.
- Route tables.
- Internet Gateway.
- NAT Gateway.
- VPC endpoints.
- Security groups.
- Network ACLs.
- VPC peering.
- Transit Gateway.
- VPC Flow Logs.

### **Example multi-account networking**

**Central Networking Account**

Transit Gateway / Network Hub

Development VPC

Production VPC

Shared VPC

Connectivity is controlled using route tables, attachments and security controls.

## **AWS Cloud WAN**

AWS Cloud WAN is a managed service for building and centrally managing global networks across AWS Regions and connected environments.

Think of it as a centralized network management solution for larger distributed environments.

| VPC / Transit Gateway                      | Cloud WAN                                                 |
| ------------------------------------------ | --------------------------------------------------------- |
| VPC provides isolated regional networking  | Cloud WAN manages global network connectivity             |
| Transit Gateway connects VPCs and networks | Cloud WAN uses a global network and core network policies |
| Regional networking focus                  | Global networking focus                                   |
| Routing and attachments managed regionally | Centralized policy-driven network management              |

For the interview, understand Cloud WAN concepts:

- Global network.
- Core network.
- Core network policy.
- Core network edges.
- Segments.
- Attachments.
- Network policies.

A possible interview question:

**How would you connect 40 AWS accounts securely?**

Answer:

I would evaluate the network topology and use a centralized network account with AWS Transit Gateway for regional connectivity, or AWS Cloud WAN where global multi-region policy-based networking is required. I would separate production and non-production traffic using appropriate routing and segmentation, centralize inspection where required, and deploy the network configurations using reusable Terraform modules.

# **10. Packer and Puppet – image and configuration management**

These technologies are included because the organization wants standardized infrastructure images and configuration.

## **Packer**

Packer is an image-building tool.

It automates the creation of machine images, such as AWS AMIs.

Example workflow:

**Base Amazon Linux AMI**

**Packer**

Install packages, agents and security patches

**Test and scan image**

**Approved Golden AMI**

Versioned and shared with approved accounts

Benefits:

- Standardized AMIs.
- Reduced configuration drift.
- Faster instance provisioning.
- Pre-installed security agents.
- Repeatable image creation.

## **Puppet**

Puppet is a configuration management tool.

It ensures that systems maintain a desired configuration.

For example:

```
package { 'httpd':
  ensure => installed,
}

service { 'httpd':
  ensure  => running,
  enable  => true,
  require => Package['httpd'],
}
```

Packer creates the standardized image; Puppet can manage and enforce the configuration of running systems.

**Interview distinction:**

Terraform provisions infrastructure, Packer builds machine images, and Puppet manages operating-system configuration.

# **11. Python and Node.js – Lambda automation**

The job requires production Lambda code with tests.

You should prepare the following concepts:

- Boto3.
- AWS SDK for JavaScript.
- IAM execution roles.
- Environment variables.
- Lambda layers.
- Timeout and memory configuration.
- Reserved concurrency.
- Retry behavior.
- Dead-letter queues and failure destinations.
- CloudWatch logging.
- Idempotency.
- Unit testing using pytest and mocks.
- Structured logging.
- Error handling.

## **Example production-style Lambda logic**

Suppose a Lambda function receives an event when an EC2 instance is launched without an approved tag.

Your remediation logic should:

1. Parse the event.
2. Validate the account and region.
3. Retrieve the EC2 instance ID.
4. Check whether the instance still exists.
5. Check whether an exception is registered.
6. Apply an approved remediation or notify the owner.
7. Log the outcome.
8. Handle retries safely.

### **Example Python unit test**

```
from unittest.mock import patchfrom app import lambda_handler@patch("app.boto3.client")def test_lambda_handler(mock_client):    mock_ec2 = mock_client.return_value    event = {        "detail": {            "instance-id": "i-123456789"        }    }    response = lambda_handler(event, None)    assert response["statusCode"] == 200    mock_client.assert_called_once_with("ec2")
```

This is an illustrative test contract; the implementation must match the actual Lambda handler and remediation design.

For a production solution, also test:

- Missing or malformed event fields.
- Unauthorized accounts.
- API failures.
- Already-remediated resources.
- Duplicate events.
- Retry behavior.
- Exceptions and exclusions.

**Key concept: Idempotency**

If EventBridge delivers the same event twice, your remediation should not cause additional damage or unintended changes.

# **12. How all these technologies work together**

This is the architecture you should be able to draw during the interview.

## **Enterprise AWS DevSecOps architecture**

**GitLab**

Source control • MR • CI/CD • approvals

**Terraform / CloudFormation**

Reusable modules • Account provisioning • Governance

**AWS Organizations**

SCP • OU • IAM Identity Center • Delegated Administration

**40+ AWS Accounts**

**Security**

GuardDuty, Security Hub, Config, CloudTrail, KMS, WAF

**Networking**

VPC, Transit Gateway, Cloud WAN

**Event-driven automation**

CloudTrail → EventBridge → Lambda / Cloud Custodian / SSM

**Remediation**

Fix non-compliant resources

**Observability**

CloudWatch / Grafana / Alerts

This is the end-to-end system that connects almost every requirement in the job description.

# **13. Most important interview questions and answers**

Prepare these questions verbally. Don't just memorize definitions.

## **AWS Organizations and IAM**

**Q1. How would you manage security across 40 AWS accounts?**

Answer:

I would use AWS Organizations and group accounts into OUs based on security and workload requirements. I would implement SCPs for preventive guardrails, use delegated administration for services such as GuardDuty, Security Hub and AWS Config, centralize audit logging, and use IAM Identity Center for workforce access. Terraform would manage reusable governance configurations across accounts.

**Q2. What is the difference between IAM policies and SCPs?**

IAM policies grant permissions to identities, subject to other applicable authorization controls. SCPs define the maximum permissions available to principals in member accounts. SCPs do not grant permissions.

**Q3. What is the difference between trust policy and permission policy?**

A trust policy specifies who can assume a role. A permission policy specifies what an assumed role can do.

**Q4. What is IAM Access Analyzer?**

It helps identify resources shared with external entities, analyze unintended access, validate policies, and support least-privilege policy design.

**Q5. How would you implement cross-account access?**

Use IAM roles with appropriate trust policies, STS AssumeRole, and least-privilege permissions. For automated GitLab deployment, use OIDC federation to avoid static credentials.

## **Terraform**

**Q6. How would you migrate manually created infrastructure to Terraform?**

Inventory resources, understand dependencies, write Terraform configuration, import resources into state, compare actual and desired configuration, review plans, and migrate gradually starting with non-production.

**Q7. What is Terraform drift?**

Drift occurs when actual infrastructure differs from the configuration and state Terraform expects, often because someone changed resources manually.

**Q8. How do you prevent Terraform state corruption?**

Use a secure remote backend, S3 versioning, locking, restricted IAM access, encryption, and controlled CI/CD execution.

**Q9. How do you deploy Terraform to multiple accounts?**

Use reusable modules, account-specific configuration, provider AssumeRole, isolated state, and a GitLab pipeline with environment-specific approval.

**Q10. What happens if Terraform wants to destroy a production resource?**

I would stop the deployment, investigate the plan, check state and dependencies, correct the configuration or state mapping, and obtain appropriate approval before proceeding. For critical resources, use lifecycle protection where suitable.

## **Event-driven automation**

**Q11. Explain CloudTrail, EventBridge and Lambda integration.**

CloudTrail records AWS API activity. EventBridge matches relevant events. Lambda executes the corresponding automation, such as notifying the owner or remediating a security issue.

**Q12. What is the difference between AWS Config and CloudTrail?**

AWS Config records resource configurations and evaluates compliance. CloudTrail records API activity and who performed actions.

**Q13. How do you prevent remediation from breaking production?**

Start with audit-only detection, analyze existing violations, test in non-production, introduce exceptions, enable remediation gradually, monitor results, and maintain rollback procedures.

**Q14. What happens if Lambda remediation fails?**

Use structured logging, retry controls, dead-letter queues or failure destinations where appropriate, CloudWatch alarms, and operational alerts. Make the remediation idempotent.

## **Cloud security and operations**

**Q15. How would you detect publicly accessible security groups?**

Use AWS Config managed rules, Cloud Custodian policies, or custom evaluation logic. Generate findings, identify resource dependencies, notify owners, and remediate approved rules.

**Q16. How do you centrally manage GuardDuty?**

Enable organization integration, designate a delegated administrator, configure member accounts, aggregate findings, and route findings through Security Hub and EventBridge where appropriate.

**Q17. How would you retire unused SAML providers?**

Inventory providers, inspect trust policies and role dependencies, validate usage with the identity team, coordinate migration, remove obsolete references, and then delete the provider.

**Q18. How do you integrate AWS with Grafana?**

Use CloudWatch as a data source in Amazon Managed Grafana or a self-managed Grafana environment. Publish metrics and dashboards, configure alerts, and provide visibility into compliance, runtime health and platform risks.

# **14. Hands-on project you should build before the interview**

I strongly recommend building one mini-project that combines the most important responsibilities.

Call it:

**Multi-Account AWS Governance and Automated Remediation Platform**

You do not need 40 AWS accounts to demonstrate the architecture. You can use a sandbox AWS Organization or a small multi-account setup, with a few simulated accounts if necessary.

## **Project implementation plan**

| Step | Implementation                     | Technology             |
| ---- | ---------------------------------- | ---------------------- |
| 1    | Create reusable AWS infrastructure | Terraform              |
| 2    | Create IAM cross-account roles     | IAM, STS               |
| 3    | Set up governance policies         | AWS Organizations, SCP |
| 4    | Enable configuration compliance    | AWS Config             |
| 5    | Create security detection rules    | Cloud Custodian        |
| 6    | Build event-driven automation      | EventBridge, Lambda    |
| 7    | Implement remediation              | Python, Boto3          |
| 8    | Build deployment automation        | GitLab CI/CD           |
| 9    | Implement runtime monitoring       | CloudWatch, Grafana    |
| 10   | Generate compliance reports        | Python, AWS APIs       |

### **Suggested practical exercises**

## **Your interview lab checklist**

Complete these in order. Prioritize the first six.

**Progress**

# **0%**

Terraform S3 backend with versioning and state locking

Critical

Create reusable Terraform IAM and S3 modules

Critical

Import an existing AWS resource into Terraform

Critical

Create an SCP to prevent disabling CloudTrail

Critical

Configure AWS Config managed rules

Critical

Create an EventBridge-to-Lambda remediation workflow

Critical

Write a Cloud Custodian YAML policy

High

Configure GitLab CI/CD for Terraform

High

Create cross-account AssumeRole access

High

Build a Lambda runtime monitoring script

High

Create a Grafana compliance dashboard

Medium

Build a Packer AMI pipeline

Medium

Write a Puppet configuration manifest

Medium

Reset checklist

# **15. Study material – official documentation and learning resources**

These are the resources I recommend studying. Start with the AWS and Terraform documentation, rather than relying exclusively on random YouTube tutorials.

## **A. AWS Organizations and Security**

[AWS Organizations – A Complete Guide - DEV Community](https://images.openai.com/static-rsc-4/-tlBFtz9Qqs9IWAvU8f3u-Ic6rm4CSnVVTmcTClL9vzGaXdhURwz0DWpl--lHIm2CdD-FtYi72Jv22vt3PqzbVh96m8HE7zj9vNxfdCWGXEkKSKiFdiHDWVnhra1vOulFO90tsC1ATiuDhwPRnF2L6Sp4e7_IJPlkQxsNrn35bQ?purpose=inline)

**AWS Organizations**

OUs, SCPs, multi-account governance

[Official documentation](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html?utm_source=chatgpt.com)

[Cost & Usage Governance with IAM :: INTRODUCTION TO AMAZON EC2.](https://images.openai.com/static-rsc-4/ESRXSaugCnboWRxnZjtY-kYyCqP-3cdZHqZelQIm-xDDt35F1einX84YCLOOEOuo0eqpi__LrhVZHtBcSogo5VG1Oyo5LIqKqgZVMS-b24fEztce_KxWHQyPjoIQwABDmLf8C9-sMFleZa0PbDbfo-GNyurTHTvVFOuI26pGLxQ?purpose=inline)

**AWS IAM**

Policy evaluation, trust policies, permissions boundaries

[IAM documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html?utm_source=chatgpt.com)

[AWS Security Hub | AWS Security Blog](https://images.openai.com/static-rsc-4/d7vT_vFnP5s2e_5n-A6mqKDtVhq5GXsc0AZP-3m4jQwce05iJsE6jRDUq5SaKbrydLcuj6-H4zqaqslwvFF85mAz-7QZ84Ef_kCHbbbT1mOoMBDXNzpv7nGqTvIXJ_RuKeEatODfkBz_epHpGnJQmyOEEIi0BVIXafygN7WET9U?purpose=inline)

**AWS Security Hub**

Security findings and automated response

[Security Hub documentation](https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html?utm_source=chatgpt.com)

[Cloud Icons | AWS Config](https://images.openai.com/static-rsc-4/VSlp542WVxUzgU477_0c_HIsFPCkz7z7d4OJt5Q-dv1EFWD_XqbCiQePRDAHkgm2l65CICE4YebNZgYj17Ijl6r9e-Y-fg1BFsa4QfDg8Uq4SxBE1eDW6j8lj5-wH0cm-blPBrWvY0_QJ2OuW-JJJ2IEmI_wVCaBA4NtiSqZ9NE?purpose=inline)

**AWS Config**

Configuration compliance and remediation

[AWS Config documentation](https://docs.aws.amazon.com/config/latest/developerguide/WhatIsConfig.html?utm_source=chatgpt.com)

## **B. Terraform**

- [HashiCorp Terraform Tutorials](https://developer.hashicorp.com/terraform/tutorials?utm_source=chatgpt.com)
- [Terraform Modules](https://developer.hashicorp.com/terraform/language/modules?utm_source=chatgpt.com)
- [Terraform S3 Backend](https://developer.hashicorp.com/terraform/language/backend/s3?utm_source=chatgpt.com)
- [Import Existing Resources](https://developer.hashicorp.com/terraform/language/import?utm_source=chatgpt.com)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs?utm_source=chatgpt.com)

## **C. Cloud Custodian**

- [Cloud Custodian official documentation](https://cloudcustodian.io/docs/?utm_source=chatgpt.com)
- [Cloud Custodian event modes](https://cloudcustodian.io/docs/aws/aws-modes.html?utm_source=chatgpt.com)
- [Cloud Custodian GitHub repository](https://github.com/cloud-custodian/cloud-custodian?utm_source=chatgpt.com)

## **D. GitLab CI/CD**

- [GitLab CI/CD documentation](https://docs.gitlab.com/ci/?utm_source=chatgpt.com)
- [GitLab AWS OIDC integration](https://docs.gitlab.com/ci/cloud_services/aws/?utm_source=chatgpt.com)
- [GitLab deployment approvals](https://docs.gitlab.com/ci/environments/deployment_approvals/?utm_source=chatgpt.com)

## **E. Other technologies**

- [Packer documentation](https://developer.hashicorp.com/packer/docs?utm_source=chatgpt.com)
- [Puppet documentation](https://help.puppet.com/core/current/Content/PuppetCore/puppet_index.htm?utm_source=chatgpt.com)
- [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html?utm_source=chatgpt.com)
- [Grafana documentation](https://grafana.com/docs/grafana/latest/?utm_source=chatgpt.com)

# **16. Seven-day interview preparation schedule**

If your interview is approaching, I would follow this priority order.

## **7-day preparation roadmap**

1. **Day 1 – AWS IAM and Organizations**
   - IAM policy evaluation
   - SCPs and permission boundaries
   - Cross-account AssumeRole
   - SAML federation
   - Delegated administration

   Highest priority
2. **Day 2 – Terraform enterprise architecture**
   - Terraform modules
   - State management
   - Resource import
   - Drift detection
   - Multi-account deployment
3. **Day 3 – AWS Security and compliance**
   - AWS Config
   - GuardDuty
   - Security Hub
   - CloudTrail
   - KMS and WAF
4. **Day 4 – Event-driven remediation**
   - EventBridge
   - Lambda
   - Cloud Custodian
   - Python Boto3
   - Detect-only vs enforcement
5. **Day 5 – GitLab and automation**
   - GitLab CI/CD
   - OIDC
   - Terraform plan/apply
   - Production approvals
   - Automated testing
6. **Day 6 – Monitoring and cloud hygiene**
   - Grafana
   - Lambda runtime lifecycle
   - Security group reviews
   - SAML provider retirement
   - Log retention
7. **Day 7 – Mock interview and architecture**
   - Draw the complete architecture.
   - Explain the 40-account migration.
   - Explain production-safe remediation.
   - Practice Terraform and IAM questions.
   - Present one end-to-end automation example.

# **17. How to introduce yourself in the Cybage interview**

Based on your backend development experience and your AWS/DevOps project work, I would position your introduction around automation, programming, infrastructure as code and security.

Do not claim production experience with Cloud Custodian, AWS Organizations or Terraform at 40-account scale unless you have actually done that work.

Here is a draft you can adapt:

I have over 8 years of experience in software development, primarily working with Java, Python, backend services, REST APIs, distributed systems and enterprise applications.

Alongside backend development, I have been building my expertise in AWS and DevOps, focusing on infrastructure automation, Terraform, Docker, CI/CD, AWS networking, IAM and cloud security.

I am particularly interested in cloud platform engineering and automation, where infrastructure is managed through reusable Terraform modules and operational processes are automated using Python, AWS Lambda and event-driven services.

I understand the importance of centralized governance in multi-account AWS environments, including AWS Organizations, SCPs, cross-account IAM roles, AWS Config, CloudTrail, EventBridge and automated remediation.

For this opportunity, I am particularly interested in the challenge of modernizing manually provisioned AWS environments into a secure, automated and centrally governed platform without disrupting existing production workloads.

# **18. The most important scenario-based question**

I want you to prepare this answer particularly well because it connects nearly every responsibility in the job description.

**Interviewer:**

"We have 40 AWS accounts. Everything was deployed manually using the AWS Console. We want to introduce Terraform, implement compliance automation and enable automatic remediation. How would you approach this?"

### **Suggested interview answer**

"I would divide this initiative into five major workstreams.

**First – Discovery and assessment.**

I would inventory all 40 accounts and identify existing resources, IAM roles, policies, networking, security configurations, logging and dependencies. I would also identify configuration drift and existing compliance gaps.

**Second – Infrastructure as Code modernization.**

I would establish a centralized GitLab repository containing reusable Terraform modules. I would configure secure remote state in S3, state locking, account-specific deployment roles and CI/CD pipelines.

For existing resources, I would use Terraform import and reconcile the configuration with the actual infrastructure. I would migrate gradually, beginning with non-production resources and avoiding unnecessary recreation of production infrastructure.

**Third – Centralized governance.**

I would organize the AWS accounts using AWS Organizations and appropriate OUs. I would implement SCPs for preventive guardrails and configure centralized security services such as GuardDuty, Security Hub, AWS Config and CloudTrail.

**Fourth – Event-driven compliance and remediation.**

I would implement AWS Config rules and Cloud Custodian policies. Initially, all rules would operate in detect-only mode. I would analyze findings, identify false positives, validate business exceptions and test remediation in non-production.

After validating each guardrail, I would enable remediation individually and monitor the results before expanding enforcement.

**Fifth – Monitoring and continuous improvement.**

I would integrate CloudWatch with Grafana to monitor platform health, Lambda runtime lifecycle, compliance findings and remediation status. I would also automate cloud-hygiene activities, document guardrails, maintain runbooks and follow GitLab code-review and change-management practices.

My overall objective would be to establish a secure-by-default, automated and observable AWS platform while maintaining the availability of existing business workloads."

# **19. My recommendation: What you should master first**

The job description is broad, but not every technology has equal importance.

## **Preparation priority**

AWS IAM and Organizations

100%

Terraform and resource import

95%

AWS Config and Cloud Custodian

95%

EventBridge and Lambda

90%

GitLab CI/CD and OIDC

85%

AWS Security services

85%

Python and Boto3

75%

Grafana and monitoring

60%

Packer and Puppet

50%

VPC and Cloud WAN

75%

These percentages represent my suggested study emphasis, not actual interview scoring.

**One important observation:** This role is much closer to an AWS Cloud Security / Governance Automation Engineer than a conventional DevOps Engineer.

The interviewer will likely be interested in whether you can reason about production risks, IAM permissions, policy evaluation, Terraform state, cross-account automation and safe remediation.

Don't just learn what each service does. Learn how they interact, how failures are handled, and how you prevent automation from damaging production.

**My suggested next step:** We should conduct a Cybage-specific technical interview preparation in a practical, question-and-answer format, starting with AWS IAM and Organizations, then Terraform migration, and finally Cloud Custodian and automated remediation. I can teach each topic from fundamentals through senior-level scenario questions, with working Terraform, Python and YAML examples.
