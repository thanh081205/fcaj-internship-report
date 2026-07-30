---
title : "Data layer"
date : 2024-01-01
weight : 4
chapter : false
pre : " <b> 5.4. </b> "
---

The network is ready, so now we place the stateful services inside it. Everything on this page is a managed service — none of it runs as a container — and none of it is reachable from the Internet.

| Service | What it stores |
|---|---|
| Amazon RDS for PostgreSQL | Users, videos, metadata — the relational source of truth |
| Amazon ElastiCache for Valkey (×2) | Sessions, BullMQ job queue, search cache |
| Amazon MQ for RabbitMQ | Metadata sync messages between the two services |
| Amazon S3 | Raw uploads and transcoded DASH output |
| Amazon EFS | Qdrant's vector index, so it survives task restarts |

#### Amazon RDS for PostgreSQL

Create the subnet group first: **RDS** console → **Subnet groups** → **Create DB subnet group**.

| Setting | Value |
|---|---|
| Name | `vsp-db-subnet-group` |
| VPC | the VPC from 5.3.1 |
| Availability Zones | `ap-southeast-1a`, `ap-southeast-1b` |
| Subnets | `10.0.10.0/24`, `10.0.11.0/24` |

Now **Databases** → **Create database**:

| Setting | Value |
|---|---|
| Creation method | Standard create |
| Engine | PostgreSQL 16 |
| Template | Free tier (or Dev/Test) |
| DB instance identifier | `vsp-rds-postgresql` |
| Master username | `postgres` |
| Credentials management | Self managed — set a strong password |
| Instance class | `db.t3.micro` |
| Storage | 20 GiB gp3, autoscaling disabled |
| Multi-AZ | Do not create a standby |
| VPC | the VPC from 5.3.1 |
| Public access | **No** |
| VPC security group | `vsp-rds-sg` |
| Availability Zone | `ap-southeast-1a` |
| Backup retention | 7 days |
| Encryption | Enabled |

![create rds](/images/5-Workshop/5.4-Data-layer/create-rds.png)

**Public access: No** is the setting that matters most. Set to Yes, RDS assigns a public IP and the database becomes reachable from the Internet, protected by nothing but the security group and your password.

Creation takes about 10 minutes.

When it finishes, copy the endpoint from the **Connectivity & security** tab:

```
vsp-rds-postgresql.xxxxxxxxxxxx.ap-southeast-1.rds.amazonaws.com:5432
```

#### Amazon ElastiCache for Valkey

Two separate clusters, one per service. They could share one, but the two workloads behave very differently: `api_service` uses Redis as a BullMQ job broker where a flush would lose queued transcode jobs, while `search_service` uses it as a disposable result cache. Keeping them apart means one service cannot evict the other's data, and either can be resized independently.

Then create two clusters with **Valkey** as the engine:

| Setting | `api-service` cluster | `search-service` cluster |
|---|---|---|
| Name | `vsp-api-redis` | `vsp-search-redis` |
| Deployment | Design your own cache — Cluster mode disabled | same |
| Node type | `cache.t4g.micro` | `cache.t4g.micro` |
| Replicas | 0 | 0 |
| Security group | `vsp-redis-api-sg` | `vsp-redis-search-sg` |
| Encryption in transit | Enabled | Enabled |
| Encryption at rest | Enabled | Enabled |

![create elasticache](/images/5-Workshop/5.4-Data-layer/create-valkey.png)

{{% notice warning %}}
If you enable **encryption in transit**, the application must connect with TLS. In practice that means the connection string uses `rediss://` rather than `redis://`, and for `ioredis` or BullMQ the client needs `tls: {}` in its options. A client configured for plaintext will hang on connect rather than returning a clear error — the handshake simply never completes.
{{% /notice %}}

Copy the **Primary endpoint** of each cluster once they reach `available`.

#### Amazon MQ for RabbitMQ

RabbitMQ carries the metadata sync between the two services: when a video finishes transcoding, `api_service` publishes to `video.metadata.trans`, `search_service` consumes it, generates embeddings, and replies on `video.metadata.res`.

**Amazon MQ** console → **Create brokers** → **RabbitMQ**:

| Setting | Value |
|---|---|
| Deployment mode | **Single-instance broker** |
| Broker name | `vsp-mq` |
| Instance type | `mq.m7g.medium` |
| Username | `vspadmin` |
| Password | 12+ characters, no commas, colons, or equals signs |
| Access type | **Private access** |
| VPC | the VPC from 5.3.1 |
| Subnet | private subnet |
| Security group | `vsp-rabbitmq-sg` |

![create amazon mq](/images/5-Workshop/5.4-Data-layer/create-mq.png)

**Private access** puts the broker inside the VPC with no public endpoint. Combined with the `mq` VPC endpoint from 5.3.3, the ECS tasks reach it entirely over private addressing.

The password has character restrictions that the console only reveals after a failed submit — avoid `,` `:` `=` and whitespace.

The broker takes about 15 minutes. When ready, copy the AMQP endpoint:

```
amqps://b-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.mq.ap-southeast-1.amazonaws.com:5671
```

{{% notice note %}}
Amazon MQ is the most expensive component in this workshop that has no free tier — roughly 40–50 USD per month for `mq.m7g.medium` running continuously. It bills whether or not any message flows through it. Section 5.8 covers deleting it.
{{% /notice %}}

#### Amazon S3

One bucket holds both raw uploads and transcoded output. **S3** console → **Create bucket**:

| Setting | Value |
|---|---|
| Bucket name | `video-platform-<your-account-id>-ap-southeast-1` |
| Region | `ap-southeast-1` |
| Block all public access | **Enabled** |
| Bucket versioning | Disabled |
| Encryption | SSE-S3 |

Bucket names are globally unique across all AWS accounts, so include your account ID to avoid collisions.

**Block all public access stays enabled.** CloudFront will read from this bucket through an Origin Access Control in section 5.6, which grants access through a bucket policy rather than by making objects public. There is never a reason to make the bucket itself public.

Open the Permissions tab and add the bucket policy:

```
{
    "Version": "2008-10-17",
    "Id": "PolicyForCloudFrontPrivateContent",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipal",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::xxxxxxxxxxx/*",
            "Condition": {
                "ArnLike": {
                    "AWS:SourceArn": "arn:aws:cloudfront::xxxxxxxxxxx"
                }
            }
        }
    ]
}
```

![create S3](/images/5-Workshop/5.4-Data-layer/create-s3.png)

#### Amazon EFS

Qdrant stores its vector index on disk. Fargate task storage is ephemeral, so without a persistent volume every deployment would wipe the index and force a full re-embedding of the video catalogue.

**EFS** console → **Create file system** → **Customize**:

| Setting | Value |
|---|---|
| Name | `vsp-qdrant-efs` |
| VPC | the VPC from 5.3.1 |
| Storage class | Regional |
| Automatic backups | Enabled |
| Lifecycle management | Transition to IA after 30 days |
| Encryption | Enabled |

On the network step, remove the default mount targets and keep **only** the one in `ap-southeast-1a` on subnet `10.0.10.0/24`. Create a security group `vsp-efs-sg` for it:

| Security group | Port | Source |
|---|---|---|
| `vsp-efs-sg` | 2049 | `vsp-qdrant-sg` |

Port 2049 is NFS. Only the Qdrant task needs it.

![create efs security group](/images/5-Workshop/5.4-Data-layer/efs-sg.png)

![create efs](/images/5-Workshop/5.4-Data-layer/create-efs.png)

#### Store the configuration in Parameter Store

Every endpoint and credential created above now needs to reach the containers. Rather than baking them into task definitions where they would appear in plain text in the console, put them in **Systems Manager Parameter Store** and have ECS inject them at runtime.

**Systems Manager** → **Parameter Store** → **Create parameter**, once per row:

![create parameter store](/images/5-Workshop/5.4-Data-layer/para-store.png)

Use **SecureString** for anything containing a password or key. SecureString parameters are encrypted with a KMS key, which is why the ECS task execution role needs `kms:Decrypt` and why the `kms` VPC endpoint exists.

The `.vsp.internal` hostnames do not resolve yet — the Cloud Map namespace is created in section 5.5. Enter them now anyway; they will resolve by the time a container reads them.
