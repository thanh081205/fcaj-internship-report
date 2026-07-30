---
title : "CI/CD with GitHub Actions"
date : 2024-01-01
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
---

Everything so far has been done by hand. This section replaces that with a pipeline: push to `main`, and GitHub Actions builds both images, runs the database migration, and rolls the ECS services forward — with no AWS access keys stored anywhere.

#### Why OIDC instead of access keys

The obvious way to let GitHub Actions deploy is to create an IAM user, generate an access key, and paste it into repository secrets. That key is long-lived, works from anywhere on the Internet, and stays valid until someone remembers to rotate it. If it leaks — in a log, a fork, a screen share — the account is exposed.

**OIDC federation** removes the key entirely. GitHub signs a short-lived token describing the workflow run, AWS verifies that signature against GitHub's public keys, and issues temporary credentials valid for the length of the job. Nothing persistent is stored on either side.

#### Register GitHub as an identity provider

**IAM** → **Identity providers** → **Add provider**:

| Setting | Value |
|---|---|
| Provider type | OpenID Connect |
| Provider URL | `https://token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |

![oidc provider](/images/5-Workshop/5.7-CICD/provider.png)

This is created once per AWS account. If it already exists, reuse it.

#### Create the deployment role

The identity provider only tells AWS that it trusts GitHub's signatures. It does not yet grant anything. That is the job of a role whose trust policy names both *which* repository may assume it and *what* it is then allowed to do.

**IAM** → **Roles** → **Create role** → **Custom trust policy**, and paste:

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				"Federated": "arn:aws:iam::{id}:oidc-provider/token.actions.githubusercontent.com"
			},
			"Action": "sts:AssumeRoleWithWebIdentity",
			"Condition": {
				"StringEquals": {
					"token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
					"token.actions.githubusercontent.com:sub": "repo:{github-owner}/{repo_name}:ref:refs/heads/main"
				}
			}
		}
	]
}
```

On the permissions page, attach the **deployment permissions policy you created in section 5.2** — the one with `ECRAuthToken`, `ECRRepositoryOperations`, `ECSServiceManagement`, and `RestrictedPassRoleToECSOnly`. That policy is already scoped to exactly what the pipeline does: push images to ECR, register and update task definitions, run the migration task, and pass `ecsTaskExecutionRole` to ECS and nothing else.

Name the role **`GitHubActionsDeployRole`** — this must match the `AWS_ROLE_ARN` variable in the next step.

{{% notice warning %}}
The `sub` condition is the entire security boundary of this setup. Omit it, or replace it with `"token.actions.githubusercontent.com:sub": "*"`, and **any** GitHub Actions workflow in **any** repository on GitHub can assume this role and deploy into your account. The `aud` check alone does not restrict anything — every GitHub OIDC token carries `sts.amazonaws.com` as its audience.
{{% /notice %}}

The `sub` value above restricts assumption to the `main` branch of one repository. Two common variations:

| Goal | `sub` value | Condition operator |
|---|---|---|
| Only `main` (recommended) | `repo:{github-owner}/{repo_name}:ref:refs/heads/main` | `StringEquals` |
| Any branch or tag in that repo | `repo:{github-owner}/{repo_name}:*` | `StringLike` |

Use `StringLike` only when the value contains a wildcard. A wildcard inside `StringEquals` is matched literally, which produces a role that silently never assumes — the failure looks like a permissions error rather than a typo.

#### Add the repository variables

In GitHub: **Settings** → **Secrets and variables** → **Actions** → **Variables**:

| Name | Value |
|---|---|
| `AWS_ROLE_ARN` | `arn:aws:iam::<account-id>:role/GitHubActionsDeployRole` |
| `AWS_REGION` | `ap-southeast-1` |
| `ECS_CLUSTER` | `vsp-ecs-cluster` |

These are **variables**, not secrets — none of them is confidential, and variables show up in log output, which makes debugging easier. There are no secrets to add: that is the point of OIDC.

#### What the workflow does, and why in that order

**Tag by commit SHA.** `${GITHUB_SHA::7}` gives every build a unique, traceable tag. You can look at a running task and know exactly which commit produced it, and rolling back means redeploying a known tag rather than rebuilding.

**Migrate before deploying.** The migration job blocks the deploy jobs through `needs`. If the migration fails, no new code reaches the cluster — the old version keeps running against the old schema, which is a working system. Deploying first and migrating second would leave new code running against a schema it does not understand.

**Fail on a non-zero exit code.** `aws ecs run-task` returns success once the task is *scheduled*, not once it succeeds. Without the explicit `describe-tasks` check on `exitCode`, a failed migration would be reported as a green step.

**Render rather than rewrite the task definition.** The render action downloads the live task definition and swaps only the image field, so secrets, port mappings, and resource limits are preserved exactly. Hand-writing the JSON in the pipeline means it drifts from reality the moment someone changes something in the console.

**Wait for stability.** `wait-for-service-stability: true` combined with the deployment circuit breaker from 5.5 means a broken image fails the workflow *and* rolls the service back automatically.

![workflow run](/images/5-Workshop/5.7-CICD/run.png)

#### Test it

```
git commit --allow-empty -m "test: trigger deployment pipeline"
git push origin main
```

Watch the run in the **Actions** tab. When it finishes, confirm the running task uses the new tag:

```
aws ecs describe-services --cluster vsp-ecs-cluster --services api-service \
  --query "services[0].taskDefinition" --output text

aws ecs describe-task-definition --task-definition vsp-api-service \
  --query "taskDefinition.containerDefinitions[0].image" --output text
```

The image tag should match the short commit SHA you just pushed.
