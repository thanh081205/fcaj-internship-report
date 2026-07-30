---
title: "Week 3 Worklog"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.3. </b> "
---
### Week 3 Objectives:

* Strengthen security practices: fine-grained IAM policies, encryption, and secrets management.
* Get started with Infrastructure as Code (IaC).
* Practice systems management without direct SSH access.

### Tasks carried out this week:
| Day | Task | Date | Reference Material |
| --- | --- | --- | --- |
| 1 (Mon) | Access Control with **IAM Policies and Conditions**; Permission Management with **IAM Permission Boundaries** | 15/06/2026 | <https://000044.awsstudygroup.com>, <https://000030.awsstudygroup.com> |
| 2 (Tue) | Encryption with **AWS Key Management Service (KMS)**; Credentials Management with **AWS Secrets Manager** | 16/06/2026 | <https://000033.awsstudygroup.com>, <https://000096.awsstudygroup.com> |
| 3 (Wed) | Remote Server Access with **Systems Manager Session Manager** (no SSH keys/bastion needed) | 17/06/2026 | <https://000058.awsstudygroup.com> |
| 4 (Thu) | Infrastructure as Code with **AWS CloudFormation**: templates, stacks, change sets | 18/06/2026 | <https://000037.awsstudygroup.com> |
| 5 (Fri) | Cloud Development Kit (**AWS CDK**) Essentials: defining infrastructure with code | 19/06/2026 | <https://000038.awsstudygroup.com> |

### Week 3 Achievements:

* Wrote IAM policies with conditions and permission boundaries to enforce least privilege more precisely.
* Encrypted data at rest using KMS and moved hard-coded credentials into Secrets Manager.
* Connected to an EC2 instance via Systems Manager Session Manager without opening inbound SSH.
* Deployed and updated a small stack using an AWS CloudFormation template.
* Wrote a first AWS CDK app (TypeScript) to provision the same resources as code instead of the console.
* Understood the trade-offs between CloudFormation (declarative YAML/JSON) and CDK (imperative code that synthesizes to CloudFormation).
