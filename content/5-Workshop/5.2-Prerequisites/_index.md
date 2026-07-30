---
title : "Prerequisites"
date : 2024-01-01
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

Before building anything, prepare the AWS account, the local tools, and the source code. Working through this section first avoids interruptions later, since every subsequent step assumes these are in place.

#### Region

This workshop uses the **Asia Pacific (Singapore) — `ap-southeast-1`** region throughout. Create every resource in this region: ECS tasks, RDS, ElastiCache, and Amazon MQ must all sit in the same VPC, and a mismatched region is the most common cause of resources that cannot see each other.

{{% notice warning %}}
**AWS Certificate Manager is the one exception.** A certificate used by CloudFront must be requested in **US East (N. Virginia) — `us-east-1`**, regardless of where the rest of your stack lives. Section 5.6 covers this in detail.
{{% /notice %}}

#### IAM permissions

To create these: open the **IAM** console, go to **Policies** → **Create policy**, switch to the **JSON** tab, and paste in the document below after adjusting it for your account.

Create the permissions for deployment:

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "ECRAuthToken",
			"Effect": "Allow",
			"Action": [
				"ecr:GetAuthorizationToken"
			],
			"Resource": "*"
		},
		{
			"Sid": "ECRRepositoryOperations",
			"Effect": "Allow",
			"Action": [
				"ecr:BatchCheckLayerAvailability",
				"ecr:GetDownloadUrlForLayer",
				"ecr:BatchGetImage",
				"ecr:InitiateLayerUpload",
				"ecr:UploadLayerPart",
				"ecr:CompleteLayerUpload",
				"ecr:PutImage"
			],
			"Resource": "arn:aws:ecr:ap-southeast-1:{id}:{repo_name}/*"
		},
		{
			"Sid": "ECSServiceManagement",
			"Effect": "Allow",
			"Action": [
				"ecs:DescribeTaskDefinition",
				"ecs:RegisterTaskDefinition",
				"ecs:UpdateService",
				"ecs:DescribeServices",
				"ecs:RunTask",
				"ecs:DescribeTasks"
			],
			"Resource": "*"
		},
		{
			"Sid": "RestrictedPassRoleToECSOnly",
			"Effect": "Allow",
			"Action": "iam:PassRole",
			"Resource": [
				"arn:aws:iam::{id}:role/ecsTaskExecutionRole"
			],
			"Condition": {
				"StringEquals": {
					"iam:PassedToService": "ecs-tasks.amazonaws.com"
				}
			}
		}
	]
}
```

Add permissions to the **ecsTaskExecutionRole** role:

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"ssmmessages:CreateControlChannel",
				"ssmmessages:CreateDataChannel",
				"ssmmessages:OpenControlChannel",
				"ssmmessages:OpenDataChannel"
			],
			"Resource": "*"
		}
	]
}
```

and:

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "SSMParameterReadAccess",
			"Effect": "Allow",
			"Action": [
				"ssm:GetParameters",
				"ssm:GetParameter",
				"ssm:GetParametersByPath"
			],
			"Resource": "arn:aws:ssm:ap-southeast-1:{id}:parameter/vsp/*"
		},
		{
			"Sid": "KMSKMSDecryptAccess",
			"Effect": "Allow",
			"Action": [
				"kms:Decrypt"
			],
			"Resource": "arn:aws:kms:ap-southeast-1:{id}:key/{kms-key-id}"
		},
		{
			"Effect": "Allow",
			"Action": [
				"logs:CreateLogGroup"
			],
			"Resource": "arn:aws:logs:ap-southeast-1:{id}:log-group:/ecs/*"
		}
	]
}
```

#### A domain name

CloudFront sits in front of the platform with a custom domain and a TLS certificate. You need a domain you control — either registered through **Route 53** or registered elsewhere with its nameservers pointed at a Route 53 hosted zone.

This is the only prerequisite that costs money up front (roughly 10–15 USD per year for a common TLD). If you would rather skip it, you can still complete every section using the default CloudFront domain (`dxxxxxxxxxx.cloudfront.net`), but the TLS and DNS steps in the following sections will not apply.

#### A GitHub repository

The next section builds a CI/CD pipeline with GitHub Actions authenticating to AWS through OIDC. You need a GitHub account and a repository holding the source code, with permission to add repository secrets and to run Actions workflows.
