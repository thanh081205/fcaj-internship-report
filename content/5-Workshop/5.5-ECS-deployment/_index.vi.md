---
title : "Triển khai service lên ECS Fargate"
date : 2024-01-01
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
---

Đây là phần mà hệ thống thực sự bắt đầu chạy. Ba service được đưa lên Fargate, chúng tìm thấy nhau qua service discovery, và kết nối tới tầng dữ liệu đã dựng ở mục 5.4.

| Service | Port container | CPU / bộ nhớ |
|---|---|---|
| `qdrant` | 6333 HTTP, 6334 gRPC | 0.25 vCPU / 2 GB |
| `search-service` | 8000 HTTP, 50052 gRPC | 4 vCPU / 9 GB |
| `api-service` | 8080 HTTP, 50051 gRPC | 2 vCPU / 5 GB |

`search-service` được cấp nhiều tài nguyên nhất vì nó chạy mô hình embedding ngay trong tiến trình — `fastembed` nạp mô hình vào bộ nhớ và ngốn CPU trong lúc sinh vector. `api-service` cần dư địa cho việc transcode bằng FFmpeg. Qdrant khá nhẹ ở quy mô dữ liệu này.

**Thứ tự dựng ở đây có ý nghĩa.** Qdrant phải chạy trước khi `search-service` khởi động, vì `search-service` kết nối tới nó ngay trong quá trình start. Hãy tạo các service theo đúng thứ tự trên trang này.

#### Tạo ECR repository

Console **ECR** → **Create repository**, làm ba lần:

| Tên repository | Thiết lập |
|---|---|
| `video-platform/api-service` | Private, bật scan on push |
| `video-platform/search-service` | Private, bật scan on push |
| `video-platform/qdrant` | Private, bật scan on push |

**Scan on push** chạy quét lỗ hổng cho mọi image bạn đẩy lên, hoàn toàn không tính thêm phí. Không có lý do gì để tắt nó.

![ecr repo](/images/5-Workshop/5.5-ECS-deployment/ecr-repo.png)

Các imgae sau khi được build bởi Github Action sẽ được lưu trữ tại đây.

![image trên ecr](/images/5-Workshop/5.5-ECS-deployment/ecr-images.png)

Tag `:latest` chấp nhận được cho lần deploy thủ công đầu tiên này. Mục 5.7 sẽ chuyển sang tag theo commit SHA, đó mới là cách làm đúng cho mọi thứ cần lặp lại được — `:latest` khiến bạn không thể biết task đang chạy đến từ bản build nào, và việc rollback trở thành đoán mò.

#### Tạo các IAM role

ECS dùng hai role riêng biệt, và nhầm lẫn giữa chúng là nguồn gây rối rất phổ biến.

**Task execution role** được ECS agent dùng *trước khi* container của bạn khởi động — để kéo image, đọc tham số SSM và tạo log stream. **Task role** được chính mã ứng dụng dùng *trong lúc chạy* — ví dụ để gọi S3.

Tạo `ecsTaskExecutionRole`: **IAM** → **Roles** → **Create role** → **AWS service** → **Elastic Container Service Task**. Gắn managed policy `AmazonECSTaskExecutionRolePolicy`, sau đó thêm một inline policy cho Parameter Store:

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

Khóa KMS ở đây là khóa mặc định `aws/ssm` của tài khoản, trừ khi bạn tự tạo khóa riêng; tìm ID của nó ở **KMS** → **AWS managed keys**.

Tiếp theo tạo `vspTaskRole` với cùng trust policy, và cấp cho nó những quyền mà bản thân ứng dụng cần:

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

Khối `ssmmessages` bật tính năng ECS Exec, cho phép bạn mở shell bên trong một task đang chạy. Trên private subnet không có bastion host, đó là cách duy nhất để gỡ lỗi container một cách tương tác — và nên chuẩn bị sẵn trước khi có sự cố, thay vì lúc đã xảy ra rồi mới quay lại thêm.

#### Tạo namespace cho service discovery

Ba service cần tìm thấy nhau theo tên. IP của task Fargate thay đổi sau mỗi lần deploy, nên ghi cứng địa chỉ là không khả thi.

**AWS Cloud Map** → **Create namespace**:

| Mục | Giá trị |
|---|---|
| Namespace name | `vsp.internal` |
| Instance discovery | API calls and DNS queries in VPCs |
| VPC | VPC đã tạo ở 5.3.1 |

![name space](/images/5-Workshop/5.5-ECS-deployment/namespace.png)

Thao tác này tạo ra một Route 53 private hosted zone gắn vào VPC. Khi một ECS service đăng ký vào đó, mỗi task nhận một bản ghi `A`, nhờ vậy `api.vsp.internal` phân giải tới đúng IP hiện hành của task từ bất kỳ đâu bên trong VPC. Đây chính là thứ khiến các tham số `GRPC_SEARCH_URL` và `QDRANT_URL` ở mục 5.4 hoạt động được.

#### Tạo cluster và log group

Console **ECS** → **Clusters** → **Create cluster**:

| Mục | Giá trị |
|---|---|
| Cluster name | `vsp-ecs-cluster` |
| Infrastructure | AWS Fargate (serverless) |
| Container Insights | Enabled |

Sau đó tạo ba CloudWatch log group — **CloudWatch** → **Log groups** → **Create**:

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

Hãy ghim image Qdrant vào một phiên bản cụ thể thay vì dùng `:latest`. Một vector database tự nâng cấp lặng lẽ trong lúc redeploy có thể đổi định dạng chỉ mục ngay dưới chân bạn.

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

Chú ý sự phân tách giữa `secrets` và `environment`. Các giá trị trong `secrets` được kéo từ Parameter Store lúc task khởi động và không bao giờ hiện ra trong console; còn giá trị trong `environment` được lưu thẳng trong task definition dưới dạng văn bản thuần. Bất cứ thứ gì chứa mật khẩu đều phải nằm ở `secrets`.

Giá trị `startPeriod` 120 giây rất quan trọng với container này. `fastembed` tải và nạp mô hình embedding trong lần khởi động đầu tiên, mất hơn một phút. Nếu không cho đủ thời gian khởi động, health check sẽ thất bại ngay trong lúc mô hình đang nạp và ECS giết task trước khi nó kịp sẵn sàng — tạo ra vòng lặp restart vô tận trông y hệt một lỗi crash.

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

#### Chạy migration cho database

Trước khi API khởi động, schema phải tồn tại. Hãy chạy migration như một task chạy một lần, thay vì nhét nó vào bước khởi động container — một migration chạy mỗi lần task boot sẽ tự đua với chính nó ngay khi bạn có nhiều hơn một task.

**Clusters** → `vsp-ecs-cluster` → **Tasks** → **Run new task**:

| Mục | Giá trị |
|---|---|
| Launch type | FARGATE |
| Task definition | `vsp-api-service` |
| Subnet | `10.0.10.0/24` |
| Security groups | `vsp-ecs-tasks-sg` |
| Public IP | Disabled |

Ở phần **Container overrides**, đặt command thành:

```
npx,prisma,migrate,deploy
```

Console yêu cầu danh sách ngăn cách bằng dấu phẩy, không phải một chuỗi shell. Chạy task rồi theo dõi log group `/ecs/vsp-api-service` trên CloudWatch cho tới khi nó kết thúc với exit code 0.

#### Tạo ba service

Giờ tạo các ECS service, **theo đúng thứ tự này**. Với từng service: **Clusters** → `vsp-ecs-cluster` → **Services** → **Create**.

**1. Qdrant**

| Mục | Giá trị |
|---|---|
| Task definition | `vsp-qdrant` |
| Service name | `qdrant` |
| Desired tasks | 1 |
| Subnet | `10.0.10.0/24` |
| Security groups | `vsp-qdrant-sg` |
| Public IP | Disabled |
| Service discovery | Bật, namespace `vsp.internal`, service name `qdrant` |
| Load balancing | Không |

Chờ task chuyển sang `RUNNING` rồi mới làm tiếp.

![qdrant](/images/5-Workshop/5.5-ECS-deployment/qdrant-service.png)

**2. search-service**

| Mục | Giá trị |
|---|---|
| Task definition | `vsp-search-service` |
| Service name | `search-service` |
| Desired tasks | 1 |
| Subnet | `10.0.10.0/24` |
| Security groups | `vsp-ecs-tasks-sg`, `vsp-grpc-search-sg` |
| Public IP | Disabled |
| Service discovery | Bật, service name `search-service` |
| Load balancing | Chưa gắn — sẽ thêm ở 5.6 |
| Deployment failure detection | Bật rollback khi thất bại |

![search](/images/5-Workshop/5.5-ECS-deployment/search.png)

**3. api-service**

| Mục | Giá trị |
|---|---|
| Task definition | `vsp-api-service` |
| Service name | `api-service` |
| Desired tasks | 1 |
| Subnet | `10.0.10.0/24` |
| Security groups | `vsp-ecs-tasks-sg`, `vsp-grpc-api-sg` |
| Public IP | Disabled |
| Service discovery | Bật, service name `api-service` |
| Load balancing | Chưa gắn — sẽ thêm ở 5.6 |
| Deployment failure detection | Bật rollback khi thất bại |

![api](/images/5-Workshop/5.5-ECS-deployment/api.png)

Mỗi service ứng dụng được gắn **hai** security group: `vsp-ecs-tasks-sg` dùng chung cho HTTP, cộng thêm group gRPC riêng của nó. Đây chính là lúc việc tách group ở mục 5.3.2 phát huy tác dụng.

**Deployment failure detection** bật cơ chế circuit breaker của ECS. Nếu một lần deploy không thể đạt trạng thái ổn định, ECS sẽ dừng lại và tự động rollback về task definition trước đó, thay vì để service mắc kẹt ở trạng thái deploy dở dang.

