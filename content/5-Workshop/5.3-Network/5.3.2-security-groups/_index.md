---
title : "Create the security groups"
date : 2024-01-01
weight : 2
chapter : false
pre : " <b> 5.3.2 </b> "
---

Security groups are the real access control layer in this architecture. Because everything sits on one private subnet, subnet boundaries do not separate the database from the cache from the containers — the security groups do.

#### The rule that makes this work

Every rule in this section allows traffic **from another security group**, not from a CIDR range. The only exception is the load balancer, which necessarily accepts traffic from the Internet.

This matters on Fargate specifically. Tasks get a new private IP on every deployment, and there may be several tasks at once behind an auto-scaling service. A rule written against `10.0.10.0/24` would work, but it would also let any future resource on that subnet reach the database. A rule written against the ECS task security group grants access to exactly the thing that needs it, and keeps working as IPs churn.

#### The groups

Create these nine security groups in the VPC from the previous step. Give each one a description — the console requires it, and it is the only place the intent gets recorded.

| Name | Attached to |
|---|---|
| `vsp-alb-sg` | Application Load Balancer |
| `vsp-ecs-tasks-sg` | `api-service` and `search-service` tasks |
| `vsp-qdrant-sg` | Qdrant task |
| `vsp-grpc-api-sg` | `api-service` task (gRPC listener) |
| `vsp-grpc-search-sg` | `search-service` task (gRPC listener) |
| `vsp-rds-sg` | RDS PostgreSQL instance |
| `vsp-redis-api-sg` | ElastiCache cluster for `api-service` |
| `vsp-redis-search-sg` | ElastiCache cluster for `search-service` |
| `vsp-rabbitmq-sg` | Amazon MQ broker |

To create one: **VPC** console → **Security groups** → **Create security group**. Set the name, a description, and the VPC. Leave the inbound rules empty for now.

![create security group](/images/5-Workshop/5.3-Network/create-sg.png)

{{% notice note %}}
Create all nine groups first with no rules, then go back and add the rules. Several rules reference other groups, and two of them reference each other, so a group has to exist before it can be named as a source.
{{% /notice %}}

#### The inbound rules

Once all nine exist, add the inbound rules below. Everything not listed here stays closed; outbound rules can keep the default allow-all.

| Security group | Port | Protocol | Source | Why |
|---|---|---|---|---|
| `vsp-alb-sg` | 443 | TCP | `0.0.0.0/0` | HTTPS from CloudFront and clients |
| `vsp-ecs-tasks-sg` | 8080 | TCP | `vsp-alb-sg` | ALB → `api-service` GraphQL and REST |
| `vsp-ecs-tasks-sg` | 8000 | TCP | `vsp-alb-sg` | ALB → `search-service` FastAPI |
| `vsp-grpc-api-sg` | 50051 | TCP | `vsp-grpc-search-sg` | `search-service` → `api-service` gRPC |
| `vsp-grpc-search-sg` | 50052 | TCP | `vsp-grpc-api-sg` | `api-service` → `search-service` gRPC |
| `vsp-qdrant-sg` | 6333 | TCP | `vsp-ecs-tasks-sg` | Qdrant HTTP API |
| `vsp-qdrant-sg` | 6334 | TCP | `vsp-ecs-tasks-sg` | Qdrant gRPC API |
| `vsp-rds-sg` | 5432 | TCP | `vsp-ecs-tasks-sg` | PostgreSQL |
| `vsp-redis-api-sg` | 6379 | TCP | `vsp-ecs-tasks-sg` | Valkey for sessions, cache, BullMQ |
| `vsp-redis-search-sg` | 6379 | TCP | `vsp-ecs-tasks-sg` | Valkey for search cache |
| `vsp-rabbitmq-sg` | 5671 | TCP | `vsp-ecs-tasks-sg` | AMQPS to Amazon MQ |
| `vsp-rabbitmq-sg` | 443 | TCP | `vsp-ecs-tasks-sg` | RabbitMQ management console over HTTPS |

![security group list](/images/5-Workshop/5.3-Network/sg-list.png)

#### The circular reference between the gRPC groups

`vsp-grpc-api-sg` allows port 50051 from `vsp-grpc-search-sg`, and `vsp-grpc-search-sg` allows port 50052 from `vsp-grpc-api-sg`. Each names the other as its source.

This is legal in AWS, but it cannot be created in a single pass — the first group cannot reference a group that does not exist yet. Create both groups empty, then add the rule to each. If you are scripting this with the CLI or CloudFormation, the same ordering applies: create the groups, then attach the rules as a separate operation.

The two services genuinely call each other. `api_service` serves a metadata RPC that `search_service` consumes, and `search_service` serves a delete RPC that `api_service` consumes, so traffic really does flow in both directions on different ports.

#### Why separate gRPC groups at all

The `api-service` and `search-service` tasks already share `vsp-ecs-tasks-sg` for their HTTP ports, so it would be simpler to add the gRPC ports there too. The reason not to is that a single shared group with 50051 and 50052 open would also let each service reach its own gRPC port, and would grant both ports to any future task attached to that group. Splitting them keeps each RPC path explicit and one-directional per port, which makes the intent readable when you come back to it later.

A task can belong to more than one security group, so `api-service` is attached to both `vsp-ecs-tasks-sg` and `vsp-grpc-api-sg`. We do that when creating the ECS services in section 5.5.

#### Verify

Confirm the rules landed correctly:

```
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=<your-vpc-id>" \
  --query "SecurityGroups[].{Name:GroupName,ID:GroupId}" \
  --output table
```

Then check one group in detail, for example the database:

```
aws ec2 describe-security-groups \
  --group-ids <vsp-rds-sg-id> \
  --query "SecurityGroups[].IpPermissions"
```

The source should appear under `UserIdGroupPairs` with the ECS task group ID — not under `IpRanges`. If it shows up as a CIDR block, the rule was entered as an address range and will break the next time a task IP changes.
