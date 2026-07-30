---
title : "Các bước chuẩn bị"
date : 2024-01-01
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

Trước khi bắt tay xây dựng, hãy chuẩn bị tài khoản AWS, các công cụ trên máy và mã nguồn. Làm xong phần này trước sẽ tránh bị gián đoạn về sau, vì mọi bước tiếp theo đều mặc định những thứ này đã sẵn sàng.

#### Region

Workshop này dùng region **Asia Pacific (Singapore) — `ap-southeast-1`** xuyên suốt. Hãy tạo mọi tài nguyên trong region này: ECS task, RDS, ElastiCache và Amazon MQ đều phải nằm trong cùng một VPC, và chọn nhầm region là nguyên nhân phổ biến nhất khiến các tài nguyên không nhìn thấy nhau.

{{% notice warning %}}
**AWS Certificate Manager là ngoại lệ duy nhất.** Chứng chỉ dùng cho CloudFront bắt buộc phải được tạo ở **US East (N. Virginia) — `us-east-1`**, bất kể phần còn lại của hệ thống nằm ở đâu. Phần 5.6 sẽ nói kỹ về điều này.
{{% /notice %}}

#### Quyền IAM

Cách tạo: mở console **IAM**, vào **Policies** → **Create policy**, chuyển sang tab **JSON** và dán nội dung đã được thay đỏi cho phù hợp vào.

Tạo quyền cho việc deploy 

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

Thêm permission vào role **ecsTaskExecutionRole**

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

và 

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

#### Một tên miền

Đặt CloudFront trước hệ thống, kèm domain riêng và chứng chỉ TLS. Bạn cần một tên miền do mình kiểm soát — đăng ký qua **Route 53**, hoặc đăng ký ở nơi khác rồi trỏ nameserver về một hosted zone của Route 53.

Đây là thứ chuẩn bị duy nhất tốn tiền ngay từ đầu (khoảng 10–15 USD/năm cho một TLD phổ thông). Nếu muốn bỏ qua, bạn vẫn hoàn thành được mọi phần bằng domain mặc định của CloudFront (`dxxxxxxxxxx.cloudfront.net`), nhưng các bước về TLS và DNS ở phần tiếp theo sẽ không áp dụng được.

#### Một repository GitHub

Phần tiếp theo xây dựng pipeline CI/CD với GitHub Actions xác thực vào AWS qua OIDC. Bạn cần một tài khoản GitHub và một repository chứa mã nguồn, có quyền thêm repository secret và chạy Actions workflow.

