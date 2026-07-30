---
title : "Create the VPC Endpoints"
date : 2024-01-01
weight : 3
chapter : false
pre : " <b> 5.3.3 </b> "
---

The private subnet has no route to the Internet, but the ECS tasks placed there still need to reach several AWS services. VPC Endpoints solve that by putting an entry point for each service inside the VPC.

#### What a task actually needs

Before a container even starts, Fargate has to authenticate to ECR, download the image layers, and create a log stream. If any of those calls cannot complete, the task fails during provisioning with an error that points at the network rather than at your application. Once running, the container reads its configuration from Systems Manager and connects to the message broker.

That produces a specific list:

| Endpoint | Type | Needed for |
|---|---|---|
| `com.amazonaws.ap-southeast-1.ecr.api` | Interface | Authenticating to ECR and resolving image manifests |
| `com.amazonaws.ap-southeast-1.ecr.dkr` | Interface | The Docker Registry API used to pull images |
| `com.amazonaws.ap-southeast-1.s3` | **Gateway** | Downloading the image layers themselves |
| `com.amazonaws.ap-southeast-1.logs` | Interface | Shipping container logs to CloudWatch |
| `com.amazonaws.ap-southeast-1.ssm` | Interface | Reading configuration from Parameter Store |
| `com.amazonaws.ap-southeast-1.kms` | Interface | Decrypting `SecureString` parameters |
| `com.amazonaws.ap-southeast-1.ssmmessages` | Interface | ECS Exec — opening a shell into a running task |
| `com.amazonaws.ap-southeast-1.ec2messages` | Interface | ECS Exec control channel |
| `com.amazonaws.ap-southeast-1.mq` | Interface | Amazon MQ management API |

{{% notice note %}}
Pulling an image requires **three** endpoints, not one. `ecr.api` handles authentication and metadata, `ecr.dkr` serves the registry protocol, and the layers themselves are stored in S3 — so the S3 endpoint is mandatory for image pulls even if your application never touches a bucket. Missing the S3 endpoint is the classic cause of a task that authenticates to ECR successfully and then stalls while downloading.
{{% /notice %}}

The S3 endpoint is a **Gateway** endpoint rather than an Interface endpoint. Gateway endpoints work by adding a route to the route table instead of creating a network interface, and they are **free** — which is worth knowing given the cost discussion below.

#### Create a security group for the endpoints

Interface endpoints are network interfaces inside your subnet, so they need a security group. Create one more group:

| Name | Port | Source |
|---|---|---|
| `vsp-vpce-sg` | 443 | `vsp-ecs-tasks-sg` |
| `vsp-vpce-sg` | 443 | `vsp-qdrant-sg` |

All AWS service APIs are HTTPS, so port 443 is the only one needed. Both task security groups are listed as sources because the Qdrant task also pulls its image from ECR and writes logs.

#### Create the interface endpoints

For each of the eight interface endpoints: **VPC** console → **Endpoints** → **Create endpoint**.

![create interface endpoint](/images/5-Workshop/5.3-Network/interface-endpoint.png)

| Setting | Value |
|---|---|
| Type | AWS services |
| VPC | the VPC from 5.3.1 |
| Subnets | Private subnet A (`10.0.10.0/24`) only |
| Enable DNS name | **Checked** |
| Security group | `vsp-vpce-sg` |
| Policy | Full access |

**Enable DNS name** is the setting that makes this transparent to your application. With Private DNS on, a request to `ecr.ap-southeast-1.amazonaws.com` from inside the VPC resolves to the endpoint's private IP instead of the public one. Your containers and the AWS CLI need no configuration change at all — they call the normal service names and the traffic quietly stays inside the VPC. With it off, the endpoint exists but nothing routes through it.

Select **only the private subnet**. Endpoints are billed per subnet they are placed in, so selecting all three subnets triples the cost for no benefit — nothing in the public subnets makes AWS API calls.

#### Create the S3 gateway endpoint

The S3 endpoint is configured differently:

| Setting | Value |
|---|---|
| Service name | `com.amazonaws.ap-southeast-1.s3` |
| Type | **Gateway** |
| VPC | the VPC from 5.3.1 |
| Route tables | the **private** route table |
| Policy | Full access |

There is no subnet or security group to choose. Selecting the private route table adds a route for the S3 prefix list, which is what sends S3 traffic through the endpoint.

#### Verify

All nine endpoints should show `available`:

![endpoint list](/images/5-Workshop/5.3-Network/ep-list.png)

Check that every interface endpoint reports `PrivateDnsEnabled: true`. An endpoint in `available` state with Private DNS disabled will not carry any traffic, and the resulting failure looks like a generic timeout.

#### About the cost

This design is usually presented as the cheap alternative to a NAT Gateway. At this endpoint count, that is not quite true, and it is worth being straight about it.

An interface endpoint bills roughly 0.01–0.013 USD per hour for each Availability Zone it is placed in, plus a small per-gigabyte data processing charge. Eight interface endpoints in one AZ therefore land somewhere around 60–80 USD per month. A single NAT Gateway is roughly 0.06 USD per hour, about 43 USD per month, plus its own per-gigabyte charge.

So on price alone, one NAT Gateway is cheaper than eight interface endpoints. What the endpoints buy is that the private subnet has genuinely no path to the Internet — traffic to AWS services never traverses the public network, and a compromised container cannot call out to an arbitrary host. The gateway endpoint for S3 is free, and S3 is where the bulk of the data volume goes, so the data processing charges stay low.

{{% notice note %}}
If cost matters more than egress isolation in your own build, drop the endpoints you do not need. `ssmmessages` and `ec2messages` exist solely for ECS Exec; if you never plan to shell into a running task, removing them saves two endpoints. Check current pricing on the [VPC pricing page](https://aws.amazon.com/vpc/pricing/) — the figures above are approximate and specific to `ap-southeast-1`.
{{% /notice %}}

#### What is next

The network is complete: a VPC with public and private subnets, correct routing, nine security groups expressing who may talk to whom, and nine endpoints giving the private subnet access to AWS services without an Internet path. Section 5.4 places the data layer — RDS, ElastiCache, Amazon MQ, and EFS — into it.
