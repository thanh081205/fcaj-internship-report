---
title : "Clean up"
date : 2024-01-01
weight : 8
chapter : false
pre : " <b> 5.8. </b> "
---

Most of this architecture bills by the hour whether anyone uses it or not. Amazon MQ, RDS, ElastiCache, the ALB, and the eight interface endpoints keep charging while completely idle — together roughly 200 USD per month if left running.

Delete everything in the order below. The order matters: AWS refuses to delete a resource while something else still depends on it, and working backwards through the dependency chain avoids a long sequence of failed delete attempts.

{{% notice warning %}}
This deletes all data permanently — the database, the vector index, and every uploaded video. If you want to keep anything, take a final RDS snapshot and download the S3 objects before starting.
{{% /notice %}}

#### 1. Stop the pipeline

Disable the workflow first, or a later push will recreate resources midway through the teardown.

In GitHub: **Actions** → **Deploy to ECS** → **⋯** → **Disable workflow**.

#### 2. Delete the CloudFront distribution

CloudFront is slow, so start it early and let it run while you delete other things.

**CloudFront** → your distribution → **Disable**, wait for the state to become `Deployed` (up to 15 minutes), then **Delete**.

A distribution cannot be deleted while enabled. There is no way to skip the disable step or speed it up.

#### 3. Delete the Route 53 record

**Route 53** → your hosted zone → select the `app` A record → **Delete record**.

Keep the hosted zone itself if you intend to reuse the domain — it costs 0.50 USD per month. Delete it only if you are done with the domain entirely, and note that you cannot delete a zone that still contains records other than the default `NS` and `SOA`.

#### 4. Delete the load balancer and target groups

**EC2** → **Load balancers** → `vsp-alb` → **Delete**.

Then **Target groups** → delete `vsp-api-tg` and `vsp-search-tg`.

The target groups must go after the ALB; a target group attached to a listener cannot be deleted.

#### 5. Delete the ECS services and cluster

Scale each service to zero, then delete:

```
for SVC in api-service search-service qdrant; do
  aws ecs update-service --cluster vsp-ecs-cluster --service $SVC --desired-count 0
done

for SVC in api-service search-service qdrant; do
  aws ecs delete-service --cluster vsp-ecs-cluster --service $SVC --force
done

aws ecs delete-cluster --cluster vsp-ecs-cluster
```

`--force` deletes a service without scaling it down first, but scaling to zero beforehand makes the deletion faster and cleaner.

Task definitions do not cost anything and can be left. To remove them anyway, deregister each revision:

```
aws ecs list-task-definitions --family-prefix vsp-api-service --query "taskDefinitionArns" --output text
aws ecs deregister-task-definition --task-definition vsp-api-service:1
```

#### 6. Delete the Cloud Map namespace

**AWS Cloud Map** → `vsp.internal` → delete the three registered services first (`api-service`, `search-service`, `qdrant`), then the namespace.

A namespace with registered services cannot be deleted. If the services will not delete, an ECS service is probably still holding a registration — confirm step 5 finished.

#### 7. Delete the data layer

This is where the real money is.

**Amazon MQ** → `vsp-mq` → **Delete**. Deleting the broker takes several minutes.

**RDS** → `vsp-rds-postgresql` → **Delete**. Uncheck **Create final snapshot** unless you want to keep the data — snapshots are billed for storage. You must type `delete me` to confirm.

**ElastiCache** → delete `vsp-api-redis` and `vsp-search-redis`, choosing no final backup.

**EFS** → `vsp-qdrant-efs` → delete the mount target first, then the file system.

**S3** → the bucket must be emptied before it can be deleted:

```
aws s3 rm s3://<your-bucket> --recursive
aws s3 rb s3://<your-bucket>
```

#### 8. Delete the VPC endpoints

Eight interface endpoints at roughly 9 USD each per month makes this the second-largest line item after the data layer.

```
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=<your-vpc-id>" \
  --query "VpcEndpoints[].VpcEndpointId" --output text
```

```
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <id1> <id2> <id3> ...
```

The S3 gateway endpoint is free but delete it too — a gateway endpoint blocks deletion of the route table it is attached to.

#### 9. Delete the ECR repositories

```
aws ecr delete-repository --repository-name video-platform/api-service --force
aws ecr delete-repository --repository-name video-platform/search-service --force
```

`--force` is required because the repositories still contain images.

#### 10. Delete the network

Now the VPC can go. Deleting the VPC from the console removes subnets, route tables, and the Internet Gateway with it, but **not** security groups that still reference each other.

Delete the security groups first, in this order:

1. `vsp-vpce-sg`, `vsp-efs-sg`
2. `vsp-rds-sg`, `vsp-redis-api-sg`, `vsp-redis-search-sg`, `vsp-rabbitmq-sg`, `vsp-qdrant-sg`
3. `vsp-grpc-api-sg` and `vsp-grpc-search-sg` — **remove the inbound rules from both before deleting either**, because each references the other
4. `vsp-ecs-tasks-sg`
5. `vsp-alb-sg`

Then **VPC** → your VPC → **Delete VPC**.

{{% notice note %}}
`DependencyViolation` on a security group almost always means a network interface still exists. Deleted ECS tasks and RDS instances leave their ENIs behind for a few minutes. Wait, then retry — or find the holder with `aws ec2 describe-network-interfaces --filters "Name=group-id,Values=<sg-id>"`.
{{% /notice %}}

#### 11. Delete the certificates, parameters, and roles

**ACM** → delete both certificates, in `us-east-1` and `ap-southeast-1`. A certificate in use by a distribution cannot be deleted, so this has to come after step 2.

**Systems Manager** → Parameter Store → delete everything under `/vsp/`:

```
aws ssm get-parameters-by-path --path /vsp --recursive \
  --query "Parameters[].Name" --output text \
  | xargs -n 10 aws ssm delete-parameters --names
```

**CloudWatch** → delete the three `/ecs/vsp-*` log groups.

**IAM** → delete `ecsTaskExecutionRole`, `vspTaskRole`, and `GitHubActionsDeployRole`, plus the OIDC provider if you have no other repository using it.

#### Verify nothing is left

```
aws ecs list-clusters
aws rds describe-db-instances --query "DBInstances[].DBInstanceIdentifier"
aws elasticache describe-cache-clusters --query "CacheClusters[].CacheClusterId"
aws mq list-brokers --query "BrokerSummaries[].BrokerName"
aws elbv2 describe-load-balancers --query "LoadBalancers[].LoadBalancerName"
aws efs describe-file-systems --query "FileSystems[].FileSystemId"
aws s3 ls
```

Every one of these should come back empty, apart from resources unrelated to this workshop.

#### Check the bill

Deleting resources does not clear charges already accrued. Open **AWS Billing** → **Bills** and confirm the current month looks like what you expect, then check again a day or two later — some services report usage with a delay.

If the budget alert from 5.2 is still active, leave it. It costs nothing and will tell you if something was missed.

#### Deletion order at a glance

| Order | Resource | Why here |
|---|---|---|
| 1 | GitHub workflow | Prevents redeploying mid-teardown |
| 2 | CloudFront | Slow; blocks certificate deletion |
| 3 | Route 53 record | Points at CloudFront |
| 4 | ALB, target groups | Target groups are held by the listener |
| 5 | ECS services, cluster | Hold ENIs and service registrations |
| 6 | Cloud Map namespace | Blocked by registered services |
| 7 | RDS, ElastiCache, MQ, EFS, S3 | The bulk of the cost |
| 8 | VPC endpoints | Gateway endpoint blocks the route table |
| 9 | ECR repositories | Independent |
| 10 | Security groups, VPC | Blocked by leftover ENIs |
| 11 | ACM, SSM, logs, IAM | Certificates blocked by CloudFront |

#### That is the whole workshop

You built a private VPC with no NAT Gateway, a managed data layer, three containerised services discovering each other through Cloud Map, a CDN-fronted public entry point, and a keyless CI/CD pipeline — and then took it all down again without leaving anything billing quietly in the background.
