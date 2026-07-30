---
title : "Public access with ALB and CloudFront"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

The three services are running but nothing outside the VPC can reach them. This section builds the path in: a load balancer in the public subnets, the CloudFront distribution designed in 5.3.4, and the DNS record that ties it to your domain.

The finished chain looks like this:

```
Browser → Route 53 → CloudFront → ALB      → ECS tasks
                                → S3 (OAC) → video files
```

#### Create the target groups

A target group is what the ALB forwards to. Create two — **EC2** console → **Target groups** → **Create target group**.

| Setting | `api-service` | `search-service` |
|---|---|---|
| Target type | **IP addresses** | **IP addresses** |
| Name | `vsp-api-tg` | `vsp-search-tg` |
| Protocol / port | HTTP / 8080 | HTTP / 8000 |
| VPC | the VPC from 5.3.1 | same |
| Protocol version | HTTP1 | HTTP1 |
| Health check path | `/health` | `/api/v1/health` |
| Healthy threshold | 2 | 2 |
| Unhealthy threshold | 3 | 3 |
| Timeout | 5 s | 5 s |
| Interval | 30 s | 30 s |

**Target type must be IP addresses**, not Instances. Fargate tasks have no EC2 instance behind them, so an instance target group cannot register them at all.

Do not register any targets manually. ECS registers and deregisters task IPs itself as tasks come and go — that is the whole point of attaching the service to the target group.

![api target](/images/5-Workshop/5.6-Public-access/api-target.png)

![search target](/images/5-Workshop/5.6-Public-access/search-target.png)

#### Create the Application Load Balancer

**EC2** console → **Load balancers** → **Create load balancer** → **Application Load Balancer**.

| Setting | Value |
|---|---|
| Name | `vsp-alb` |
| Scheme | **Internet-facing** |
| IP address type | IPv4 |
| VPC | the VPC from 5.3.1 |
| Mappings | `ap-southeast-1a` → `10.0.1.0/24`, `ap-southeast-1b` → `10.0.2.0/24` |
| Security group | `vsp-alb-sg` |
| Listener | HTTPS : 443 |
| Default action | Forward to `vsp-api-tg` |
| Certificate | the `ap-southeast-1` certificate from 5.3.4 |
| Security policy | `ELBSecurityPolicy-TLS13-1-2-2021-06` |

Select **both public subnets**. This is why 5.3.1 created two of them — the ALB will not be created with a single subnet.

The ALB is internet-facing even though only CloudFront should talk to it. An internal ALB cannot be used as a CloudFront origin, because CloudFront reaches origins over the public network. The protection comes from the security group and from never publishing the ALB's DNS name, not from making it internal.

#### Add a listener rule for the search service

The default action sends everything to the API. Add a rule so search traffic reaches the right service — open the HTTPS listener → **Rules** → **Add rule**:

| Setting | Value |
|---|---|
| Priority | 100 |
| Condition | Path is `/api/v1/search*` |
| Action | Forward to `vsp-search-tg` |

Rules are evaluated by priority, lowest first, and the default action runs only if no rule matches.

![alb](/images/5-Workshop/5.6-Public-access/alb.png)

#### Attach the ECS services to the target groups

Back in **ECS** → `vsp-ecs-cluster` → **Services** → `api-service` → **Update**:

| Setting | Value |
|---|---|
| Load balancer type | Application Load Balancer |
| Load balancer | `vsp-alb` |
| Container to load balance | `api-service-container` 8080 |
| Target group | `vsp-api-tg` |
| Health check grace period | 120 |

![service target api](/images/5-Workshop/5.6-Public-access/ser-tar-api.png)

![service target search](/images/5-Workshop/5.6-Public-access/ser-tar-search.png)

Repeat for `search-service` with `search-service-container` 8000, target group `vsp-search-tg`, and a grace period of **180** seconds.

The **health check grace period** tells ECS to ignore load balancer health checks for that long after a task starts. Without it, the ALB marks the still-booting task unhealthy, ECS kills it, starts another, and the service never stabilises. The search service needs the longer window because of model loading.

Wait for both target groups to show `healthy`:

![healthy targets](/images/5-Workshop/5.6-Public-access/health.png)


#### Create the Origin Access Control

Before the distribution, create the identity CloudFront uses to read from S3. **CloudFront** console → **Origin access** → **Create control setting**:

| Setting | Value |
|---|---|
| Name | `vsp-s3-oac` |
| Origin type | S3 |
| Signing behavior | Sign requests (recommended) |

Origin Access Control replaces the older Origin Access Identity and is what lets the bucket stay fully private while CloudFront still reads from it.

![origin access s3](/images/5-Workshop/5.6-Public-access/oas3.png)

#### Create the distribution

**CloudFront** → **Create distribution**.

**Origins** — add two:

| | ALB origin | S3 origin |
|---|---|---|
| Origin domain | `vsp-alb-…elb.amazonaws.com` | your bucket |
| Origin path | empty | empty |
| Protocol | HTTPS only | — |
| Origin access | — | Origin access control, `vsp-s3-oac` |
| Name | `alb-origin` | `s3-origin` |

**Default cache behavior** — this one serves the API:

| Setting | Value |
|---|---|
| Origin | `alb-origin` |
| Viewer protocol policy | Redirect HTTP to HTTPS |
| Allowed methods | GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE |
| Cache policy | **CachingDisabled** |
| Origin request policy | **AllViewerExceptHostHeader** |

`AllViewerExceptHostHeader` forwards every header, cookie, and query string to the ALB except `Host`. The exception matters: forwarding the viewer's `Host` header to an ALB origin breaks the routing, because the ALB expects its own hostname.

**Additional behaviors** — add two more:

| Path pattern | Origin | Cache policy | Methods |
|---|---|---|---|
| `/public/*` | `s3-origin` | CachingOptimized | GET, HEAD |
| `/private/*` | `s3-origin` | CachingOptimized | GET, HEAD |

**Settings**:

| Setting | Value |
|---|---|
| Price class | Use only North America, Europe, Asia |
| Alternate domain name (CNAME) | `app.example.com` |
| Custom SSL certificate | the **`us-east-1`** certificate from 5.3.4 |
| Security policy | TLSv1.2_2021 |
| Default root object | leave empty |

![cloudfront behaviors](/images/5-Workshop/5.6-Public-access/cloudfront-behaviors.png)

#### Update the S3 bucket policy

After the distribution is created, CloudFront shows a generated bucket policy — copy it, or write it yourself:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipal",
            "Effect": "Allow",
            "Principal": { "Service": "cloudfront.amazonaws.com" },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<your-bucket>/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::<account-id>:distribution/<distribution-id>"
                }
            }
        }
    ]
}
```

Paste it under **S3** → your bucket → **Permissions** → **Bucket policy**. Follow the step above.

#### Point the domain at CloudFront

**Route 53** → your hosted zone → **Create record**:

| Setting | Value |
|---|---|
| Record name | `app` |
| Record type | A |
| Alias | Yes |
| Route traffic to | Alias to CloudFront distribution |
| Distribution | your distribution |

![route53 alias record](/images/5-Workshop/5.6-Public-access/record.png)

#### Test the whole path

```
dig app.example.com +short
curl -I https://app.example.com/health
curl -s https://app.example.com/api/v1/health
```

The first should return CloudFront IP addresses; the second `200 OK` from the API through the ALB; the third the search service, confirming the listener rule works.

![ping](/images/5-Workshop/5.6-Public-access/ping.png)
