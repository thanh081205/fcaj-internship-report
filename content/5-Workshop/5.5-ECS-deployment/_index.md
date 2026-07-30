---
title : "Deploy services to ECS Fargate"
date : 2024-01-01
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
---

This is the section where the platform actually starts running. Three services go onto Fargate, they find each other through service discovery, and they connect to the data layer built in 5.4.

| Service | Container ports | CPU / memory |
|---|---|---|
| `qdrant` | 6333 HTTP, 6334 gRPC | 0.25 vCPU / 2 GB |
| `search-service` | 8000 HTTP, 50052 gRPC | 4 vCPU / 9 GB |
| `api-service` | 8080 HTTP, 50051 gRPC | 2 vCPU / 5 GB |

`search-service` gets the largest allocation because it runs the embedding model in-process — `fastembed` loads the model into memory and is CPU-bound while generating vectors. `api-service` needs headroom for FFmpeg transcoding. Qdrant is small at this data volume.

**Build order matters here.** Qdrant must be running before `search-service` starts, because `search-service` connects to it during startup. Create the services in the order given on this page.

#### Create the ECR repositories

**ECR** console → **Create repository**, three times:

| Repository name | Settings |
|---|---|
| `video-platform/api-service` | Private, scan on push enabled |
| `video-platform/search-service` | Private, scan on push enabled |
| `video-platform/qdrant` | Private, scan on push enabled |

**Scan on push** runs a vulnerability scan against every image you upload, at no extra cost. There is no reason to leave it off.

![ecr repo](/images/5-Workshop/5.5-ECS-deployment/ecr-repo.png)

Images built by GitHub Actions are stored here.

![ecr images](/images/5-Workshop/5.5-ECS-deployment/ecr-images.png)

The `:latest` tag is fine for this first manual deployment. Section 5.7 switches to commit-SHA tags, which is what you want for anything repeatable — `:latest` makes it impossible to tell which build a running task came from, and rollback becomes guesswork.

#### Create the IAM roles

ECS uses two distinct roles, and mixing them up is a common source of confusion.

**The task execution role** is used by the ECS agent *before* your container starts — to pull the image, read SSM parameters, and create log streams. **The task role** is used by your application code *while it runs* — to call S3, for example.

Create `ecsTaskExecutionRole`: **IAM** → **Roles** → **Create role** → **AWS service** → **Elastic Container Service Task**. Attach the managed policy `AmazonECSTaskExecutionRolePolicy`, then add an inline policy for Parameter Store:

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
			"Resource": "arn:aws:ssm:xxxxxxxxxx:parameter/vsp/*"
		},
		{
			"Sid": "KMSKMSDecryptAccess",
			"Effect": "Allow",
			"Action": [
				"kms:Decrypt"
			],
			"Resource": "arn:aws:kms:xxxxxxxxxxxxxxxxxxxxx"
		},
		{
			"Effect": "Allow",
			"Action": [
				"logs:CreateLogGroup"
			],
			"Resource": "arn:aws:xxxxxxxxxxxxxxxxxx:log-group:/ecs/*"
		}
	]
}
```

The KMS key is the account's default `aws/ssm` key unless you created your own; find its ID under **KMS** → **AWS managed keys**.

Now create `vspTaskRole` with the same trust policy, and give it the permissions the application itself needs:

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

The `ssmmessages` block enables ECS Exec, which lets you open a shell inside a running task. On a private subnet with no bastion host, that is the only way to debug a container interactively — and it is worth having before something goes wrong rather than after.

#### Create the service discovery namespace

The three services need to find each other by name. Fargate task IPs change on every deployment, so hardcoding them is not an option.

**AWS Cloud Map** → **Create namespace**:

| Setting | Value |
|---|---|
| Namespace name | `vsp.internal` |
| Instance discovery | API calls and DNS queries in VPCs |
| VPC | the VPC from 5.3.1 |

![name space](/images/5-Workshop/5.5-ECS-deployment/namespace.png)

This creates a Route 53 private hosted zone attached to the VPC. When an ECS service registers with it, each task gets an `A` record, so `api.vsp.internal` resolves to the current task IP from anywhere inside the VPC. This is what makes the `GRPC_SEARCH_URL` and `QDRANT_URL` parameters from 5.4 work.

#### Create the cluster and log groups

**ECS** console → **Clusters** → **Create cluster**:

| Setting | Value |
|---|---|
| Cluster name | `vsp-ecs-cluster` |
| Infrastructure | AWS Fargate (serverless) |
| Container Insights | Enabled |

Then create three CloudWatch log groups — **CloudWatch** → **Log groups** → **Create**:

```
/ecs/vsp-api-service
/ecs/vsp-search-service
/ecs/vsp-qdrant
```

![cluster](/images/5-Workshop/5.5-ECS-deployment/cluster.png)

#### Task definition 1 — Qdrant

**Task definitions** → **Create new task definition** → **JSON**:

```
{
  "taskDefinitionArn": "arn:aws:ecs:ap-southeast-1:{account-id}:task-definition/vsp-qdrant:2",
  "containerDefinitions": [
    {
      "name": "qdrant-container",
      "image": {image_uri},
      "cpu": 0,
      "portMappings": [
        {
          "containerPort": 6333,
          "hostPort": 6333,
          "protocol": "tcp",
          "name": "qdrant-container-6333-tcp",
          "appProtocol": "http"
        },
        {
          "containerPort": 6334,
          "hostPort": 6334,
          "protocol": "tcp",
          "name": "qdrant-container-6334-tcp",
          "appProtocol": "grpc"
        }
      ],
      "essential": true,
      "environment": [],
      "environmentFiles": [],
      "mountPoints": [],
      "volumesFrom": [],
      "ulimits": [],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/vsp-qdrant",
          "awslogs-create-group": "true",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs"
        },
        "secretOptions": []
      },
      "systemControls": []
    }
  ],
  "family": "vsp-qdrant",
  "taskRoleArn": "arn:aws:iam::{account-id}:role/ecsTaskExecutionRole",
  "executionRoleArn": "arn:aws:iam::{account-id}:role/ecsTaskExecutionRole",
  "networkMode": "awsvpc",
  "revision": 2,
  "volumes": [
    {
      "name": "qdrant-volume",
      "efsVolumeConfiguration": {
        "fileSystemId": "{efs-id}",
        "rootDirectory": "/"
      }
    }
  ],
  "status": "ACTIVE",
  "requiresAttributes": [
    {
      "name": "ecs.capability.execution-role-awslogs"
    },
    {
      "name": "com.amazonaws.ecs.capability.ecr-auth"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.21"
    },
    {
      "name": "com.amazonaws.ecs.capability.task-iam-role"
    },
    {
      "name": "ecs.capability.execution-role-ecr-pull"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.18"
    },
    {
      "name": "ecs.capability.task-eni"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.29"
    },
    {
      "name": "com.amazonaws.ecs.capability.logging-driver.awslogs"
    },
    {
      "name": "ecs.capability.efsAuth"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.19"
    },
    {
      "name": "ecs.capability.efs"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.25"
    }
  ],
  "placementConstraints": [],
  "compatibilities": [
    "EC2",
    "FARGATE",
    "MANAGED_INSTANCES"
  ],
  "runtimePlatform": {
    "cpuArchitecture": "X86_64",
    "operatingSystemFamily": "LINUX"
  },
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "cpu": "256",
  "memory": "2048",
  "registeredAt": "2026-07-28T10:54:31.814Z",
  "registeredBy": "arn:aws:iam::{account-id}:root",
  "enableFaultInjection": false,
  "tags": [
    {
      "key": "Project",
      "value": "Video-Streaming-Platform"
    }
  ]
}
```

Pin the Qdrant image to a specific version rather than `:latest`. A vector database that silently upgrades itself during a redeploy can change index formats underneath you.

#### Task definition 2 — search-service

```
{
  "taskDefinitionArn": "arn:aws:ecs:ap-southeast-1:{account-id}:task-definition/vsp-ecs-search-service-task:11",
  "containerDefinitions": [
    {
      "name": "search-service-container",
      "image": {image_uri},
      "cpu": 0,
      "portMappings": [
        {
          "containerPort": 8000,
          "hostPort": 8000,
          "protocol": "tcp",
          "name": "search-service-container-8000-tcp",
          "appProtocol": "http"
        },
        {
          "containerPort": 50052,
          "hostPort": 50052,
          "protocol": "tcp",
          "name": "search-service-container-50052-tcp",
          "appProtocol": "grpc"
        }
      ],
      "essential": true,
      "environment": [
        {
          "name": "REDIS_TLS",
          "value": "true"
        }
      ],
      "mountPoints": [],
      "volumesFrom": [],
      "secrets": [
        {
          "name": "API_SERVICE_GRPC_URL",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/search/API_SERVICE_GRPC_URL"
        },
        {
          "name": "CLOUDFRONT_DOMAIN_NAME",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/search/CLOUDFRONT_DOMAIN_NAME"
        },
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/search/DATABASE_URL"
        },
        {
          "name": "GRPC_URL",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/search/GRPC_URL"
        },
        {
          "name": "JWT_SECRET",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/search/JWT_SECRET"
        },
        {
          "name": "MQ_URL",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/search/MQ_URL"
        },
        {
          "name": "QDRANT_URL",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/search/QDRANT_URL"
        },
        {
          "name": "REDIS_URL",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/search/REDIS_URL"
        },
        {
          "name": "S3_ACCESS_KEY",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/search/S3_ACCESS_KEY"
        },
        {
          "name": "S3_BUCKET",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/search/S3_BUCKET"
        },
        {
          "name": "S3_REGION",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/search/S3_REGION"
        },
        {
          "name": "S3_SECRET_KEY",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/search/S3_SECRET_KEY"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/vsp-ecs-search-service-task",
          "awslogs-create-group": "true",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs"
        },
        "secretOptions": []
      },
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -f http://localhost:8000/api/v1/health || exit 1"
        ],
        "interval": 30,
        "timeout": 5,
        "retries": 5,
        "startPeriod": 200
      },
      "systemControls": []
    }
  ],
  "family": "vsp-ecs-search-service-task",
  "taskRoleArn": "arn:aws:iam::{account-id}:role/ecsTaskExecutionRole",
  "executionRoleArn": "arn:aws:iam::{account-id}:role/ecsTaskExecutionRole",
  "networkMode": "awsvpc",
  "revision": 11,
  "volumes": [],
  "status": "ACTIVE",
  "requiresAttributes": [
    {
      "name": "com.amazonaws.ecs.capability.logging-driver.awslogs"
    },
    {
      "name": "ecs.capability.execution-role-awslogs"
    },
    {
      "name": "com.amazonaws.ecs.capability.ecr-auth"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.19"
    },
    {
      "name": "com.amazonaws.ecs.capability.task-iam-role"
    },
    {
      "name": "ecs.capability.container-health-check"
    },
    {
      "name": "ecs.capability.execution-role-ecr-pull"
    },
    {
      "name": "ecs.capability.secrets.ssm.environment-variables"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.18"
    },
    {
      "name": "ecs.capability.task-eni"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.29"
    }
  ],
  "placementConstraints": [],
  "compatibilities": [
    "EC2",
    "MANAGED_INSTANCES",
    "FARGATE"
  ],
  "runtimePlatform": {
    "cpuArchitecture": "X86_64",
    "operatingSystemFamily": "LINUX"
  },
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "cpu": "4096",
  "memory": "9216",
  "registeredAt": "2026-07-29T15:59:09.260Z",
  "registeredBy": "arn:aws:iam::{account-id}:root",
  "enableFaultInjection": false,
  "tags": []
}
```

Note the split between `secrets` and `environment`. Values under `secrets` are pulled from Parameter Store at task startup and never appear in the console; values under `environment` are stored in the task definition as plain text. Anything with a password goes in `secrets`.

The `startPeriod` of 120 seconds matters for this container. `fastembed` downloads and loads the embedding model on first start, which takes well over a minute. Without a generous start period, the health check fails during model loading and ECS kills the task before it ever becomes ready — producing an endless restart loop that looks like a crash.

#### Task definition 3 — api-service

```
{
  "taskDefinitionArn": "arn:aws:ecs:ap-southeast-1:{account-id}:task-definition/vsp-ecs-api-service-task:18",
  "containerDefinitions": [
    {
      "name": "api-service-container",
      "image": {image_uri},
      "cpu": 0,
      "portMappings": [
        {
          "containerPort": 8080,
          "hostPort": 8080,
          "protocol": "tcp",
          "name": "api-service-container-8080-tcp",
          "appProtocol": "http"
        },
        {
          "containerPort": 50051,
          "hostPort": 50051,
          "protocol": "tcp",
          "name": "api-service-container-50051-tcp",
          "appProtocol": "grpc"
        }
      ],
      "essential": true,
      "environment": [
        {
          "name": "REDIS_TLS",
          "value": "true"
        }
      ],
      "environmentFiles": [],
      "mountPoints": [],
      "volumesFrom": [],
      "secrets": [
        {
          "name": "BUCKET_NAME",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/BUCKET_NAME"
        },
        {
          "name": "CDN_DOMAIN",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/CDN_DOMAIN"
        },
        {
          "name": "CLOUDFRONT_DOMAIN_NAME",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/CLOUDFRONT_DOMAIN_NAME"
        },
        {
          "name": "CLOUDFRONT_KEY_PAIR_ID",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/CLOUDFRONT_KEY_PAIR_ID"
        },
        {
          "name": "CLOUDFRONT_PRIVATE_KEY",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/CLOUDFRONT_PRIVATE_KEY"
        },
        {
          "name": "COOKIE_SECRET",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/COOKIE_SECRET"
        },
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/DATABASE_URL"
        },
        {
          "name": "FIREBASE_CLIENT_EMAIL",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/FIREBASE_CLIENT_EMAIL"
        },
        {
          "name": "FIREBASE_PRIVATE_KEY",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/FIREBASE_PRIVATE_KEY"
        },
        {
          "name": "FIREBASE_PROJECT_ID",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/FIREBASE_PROJECT_ID"
        },
        {
          "name": "GRPC_URL",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/GRPC_URL"
        },
        {
          "name": "JWT_SECRET",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/JWT_SECRET"
        },
        {
          "name": "QDRANT_URL",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/QDRANT_URL"
        },
        {
          "name": "QUEUE_HOST",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/QUEUE_HOST"
        },
        {
          "name": "QUEUE_NAME",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/QUEUE_NAME"
        },
        {
          "name": "QUEUE_PORT",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/QUEUE_PORT"
        },
        {
          "name": "RABBITMQ_URI",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/RABBITMQ_URI"
        },
        {
          "name": "REDIS_HOST",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/REDIS_HOST"
        },
        {
          "name": "REDIS_PORT",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/REDIS_PORT"
        },
        {
          "name": "S3_ACCESS_KEY_ID",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/S3_ACCESS_KEY_ID"
        },
        {
          "name": "S3_REGION",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/S3_REGION"
        },
        {
          "name": "S3_SECRET_ACCESS_KEY",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/S3_SECRET_ACCESS_KEY"
        },
        {
          "name": "SEARCH_SERVICE_GRPC_URL",
          "valueFrom": "arn:aws:ssm:ap-southeast-1:{account-id}:parameter/vsp/api/SEARCH_SERVICE_GRPC_URL"
        }
      ],
      "ulimits": [],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/vsp-ecs-api-service-task",
          "awslogs-create-group": "true",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs"
        },
        "secretOptions": []
      },
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -f http://localhost:8080/health || exit 1"
        ],
        "interval": 30,
        "timeout": 5,
        "retries": 5,
        "startPeriod": 200
      },
      "systemControls": []
    }
  ],
  "family": "vsp-ecs-api-service-task",
  "taskRoleArn": "arn:aws:iam::{account-id}:role/ecsTaskExecutionRole",
  "executionRoleArn": "arn:aws:iam::{account-id}:role/ecsTaskExecutionRole",
  "networkMode": "awsvpc",
  "revision": 18,
  "volumes": [],
  "status": "ACTIVE",
  "requiresAttributes": [
    {
      "name": "com.amazonaws.ecs.capability.logging-driver.awslogs"
    },
    {
      "name": "ecs.capability.execution-role-awslogs"
    },
    {
      "name": "com.amazonaws.ecs.capability.ecr-auth"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.19"
    },
    {
      "name": "com.amazonaws.ecs.capability.task-iam-role"
    },
    {
      "name": "ecs.capability.container-health-check"
    },
    {
      "name": "ecs.capability.execution-role-ecr-pull"
    },
    {
      "name": "ecs.capability.secrets.ssm.environment-variables"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.18"
    },
    {
      "name": "ecs.capability.task-eni"
    },
    {
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.29"
    }
  ],
  "placementConstraints": [],
  "compatibilities": [
    "EC2",
    "MANAGED_INSTANCES",
    "FARGATE"
  ],
  "runtimePlatform": {
    "cpuArchitecture": "X86_64",
    "operatingSystemFamily": "LINUX"
  },
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "cpu": "2048",
  "memory": "5120",
  "registeredAt": "2026-07-29T13:54:46.984Z",
  "registeredBy": "arn:aws:iam::{account-id}:root",
  "enableFaultInjection": false,
  "tags": []
}
```

#### Run the database migration

Before the API starts, the schema has to exist. Run the migration as a one-off task rather than baking it into the container startup — a migration that runs on every task boot will race with itself the moment you have more than one task.

**Clusters** → `vsp-ecs-cluster` → **Tasks** → **Run new task**:

| Setting | Value |
|---|---|
| Launch type | FARGATE |
| Task definition | `vsp-api-service` |
| Subnet | `10.0.10.0/24` |
| Security groups | `vsp-ecs-tasks-sg` |
| Public IP | Disabled |

Under **Container overrides**, set the command to:

```
npx,prisma,migrate,deploy
```

The console expects a comma-separated list, not a shell string. Run the task and watch the `/ecs/vsp-api-service` log group in CloudWatch until it exits with code 0.

#### Create the three services

Now create the ECS services, **in this order**. For each: **Clusters** → `vsp-ecs-cluster` → **Services** → **Create**.

**1. Qdrant**

| Setting | Value |
|---|---|
| Task definition | `vsp-qdrant` |
| Service name | `qdrant` |
| Desired tasks | 1 |
| Subnet | `10.0.10.0/24` |
| Security groups | `vsp-qdrant-sg` |
| Public IP | Disabled |
| Service discovery | Enable, namespace `vsp.internal`, service name `qdrant` |
| Load balancing | None |

Wait until the task is `RUNNING` before continuing.

![qdrant](/images/5-Workshop/5.5-ECS-deployment/qdrant-service.png)

**2. search-service**

| Setting | Value |
|---|---|
| Task definition | `vsp-search-service` |
| Service name | `search-service` |
| Desired tasks | 1 |
| Subnet | `10.0.10.0/24` |
| Security groups | `vsp-ecs-tasks-sg`, `vsp-grpc-search-sg` |
| Public IP | Disabled |
| Service discovery | Enable, service name `search-service` |
| Load balancing | None for now — added in 5.6 |
| Deployment failure detection | Enable rollback on failure |

![search](/images/5-Workshop/5.5-ECS-deployment/search.png)

**3. api-service**

| Setting | Value |
|---|---|
| Task definition | `vsp-api-service` |
| Service name | `api-service` |
| Desired tasks | 1 |
| Subnet | `10.0.10.0/24` |
| Security groups | `vsp-ecs-tasks-sg`, `vsp-grpc-api-sg` |
| Public IP | Disabled |
| Service discovery | Enable, service name `api-service` |
| Load balancing | None for now — added in 5.6 |
| Deployment failure detection | Enable rollback on failure |

![api](/images/5-Workshop/5.5-ECS-deployment/api.png)

Attach **two** security groups to each application service: the shared `vsp-ecs-tasks-sg` for HTTP, plus its own gRPC group. This is where the split from 5.3.2 pays off.

**Deployment failure detection** turns on the ECS circuit breaker. If a new deployment cannot reach a steady state, ECS stops it and rolls back to the previous task definition automatically instead of leaving the service in a broken half-deployed state.
