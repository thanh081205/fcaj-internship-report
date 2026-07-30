---
title : "Network foundation"
date : 2024-01-01
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---

Every resource in the rest of this workshop lives inside the network you build here. The database, the cache, the message broker, and all three ECS tasks are placed on private subnets with no route to the Internet, and the only thing exposed publicly is the load balancer. Getting this layer right first means the later sections are mostly configuration rather than troubleshooting.

#### Design principles

**Private by default.** Compute and data sit on a subnet with no route to the Internet Gateway. Nothing in the platform needs to be reachable from outside except the load balancer, so nothing else gets a public path.

**No NAT Gateway.** ECS tasks still need to pull images from ECR, ship logs to CloudWatch, and read parameters from Systems Manager. Instead of routing that traffic out through a NAT Gateway and back into AWS, we place **VPC Endpoints** inside the VPC so the traffic never leaves the AWS network.

**Security groups reference security groups.** Rules are written as "allow port 5432 from the ECS task security group", never "allow port 5432 from 10.0.10.0/24". When a task's IP changes — which happens on every Fargate deployment — the rule keeps working.

#### Address plan

| Subnet | CIDR | Availability Zone | Type | What it holds |
|---|---|---|---|---|
| Public A | `10.0.1.0/24` | `ap-southeast-1a` | Public | Application Load Balancer |
| Public B | `10.0.2.0/24` | `ap-southeast-1b` | Public | Application Load Balancer |
| Private A | `10.0.10.0/24` | `ap-southeast-1a` | Private | ECS tasks, RDS, ElastiCache, Amazon MQ, VPC Endpoints |

The VPC uses `10.0.0.0/16`, which leaves plenty of room to add subnets later.

Two public subnets exist because an Application Load Balancer requires subnets in at least two Availability Zones — it will refuse to be created with only one. The ALB itself is the only resource that lives there.

{{% notice note %}}
Everything else is placed in a single private subnet in `ap-southeast-1a`. That is a deliberate simplification to keep the workshop affordable: a second AZ would mean a second set of VPC Endpoints and a Multi-AZ database, roughly doubling the hourly cost. It also means an AZ failure takes the platform down. A production deployment would span both AZs.
{{% /notice %}}

![network diagram](/images/5-Workshop/5.3-Network/network-diagram.png)

#### Traffic paths

Once this section is complete, there are exactly three ways traffic moves:

+ **Inbound** — the Internet reaches the ALB in the public subnets, and the ALB forwards to ECS tasks in the private subnet. Nothing else accepts connections from outside.
+ **East-west** — ECS tasks talk to each other and to the data layer entirely within the private subnet, over private IP addresses.
+ **Outbound to AWS services** — ECS tasks reach ECR, CloudWatch Logs, Systems Manager, KMS, Amazon MQ, and S3 through VPC Endpoints, which resolve to private IP addresses inside the VPC.

There is no fourth path. A task cannot reach an arbitrary address on the Internet, which is a meaningful part of the security posture but also something to keep in mind if you later add a dependency on an external API.

#### Content

1. [Create the VPC, subnets, and routing](5.3.1-create-vpc/)
2. [Create the security groups](5.3.2-security-groups/)
3. [Create the VPC Endpoints](5.3.3-vpc-endpoints/)
4. [DNS and the CloudFront edge](5.3.4-dns-cdn/)
