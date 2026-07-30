---
title : "Tầng dữ liệu"
date : 2024-01-01
weight : 4
chapter : false
pre : " <b> 5.4. </b> "
---

Nền tảng mạng đã sẵn sàng, giờ ta đặt các dịch vụ lưu trữ trạng thái vào bên trong nó. Toàn bộ những gì trên trang này đều là dịch vụ được quản lý — không có thứ nào chạy dưới dạng container — và không thứ nào truy cập được từ Internet.

| Dịch vụ | Lưu gì |
|---|---|
| Amazon RDS for PostgreSQL | Người dùng, video, metadata — nguồn dữ liệu quan hệ chính thống |
| Amazon ElastiCache for Valkey (×2) | Session, hàng đợi job BullMQ, cache tìm kiếm |
| Amazon MQ for RabbitMQ | Thông điệp đồng bộ metadata giữa hai service |
| Amazon S3 | File upload gốc và output DASH sau khi transcode |
| Amazon EFS | Chỉ mục vector của Qdrant, để nó sống sót qua các lần restart task |

#### Amazon RDS for PostgreSQL

Tạo subnet group trước: console **RDS** → **Subnet groups** → **Create DB subnet group**.

| Mục | Giá trị |
|---|---|
| Name | `vsp-db-subnet-group` |
| VPC | VPC đã tạo ở 5.3.1 |
| Availability Zones | `ap-southeast-1a`, `ap-southeast-1b` |
| Subnets | `10.0.10.0/24`, `10.0.11.0/24` |

Tiếp theo vào **Databases** → **Create database**:

| Mục | Giá trị |
|---|---|
| Creation method | Standard create |
| Engine | PostgreSQL 16 |
| Template | Free tier (hoặc Dev/Test) |
| DB instance identifier | `vsp-rds-postgresql` |
| Master username | `postgres` |
| Credentials management | Self managed — đặt mật khẩu mạnh |
| Instance class | `db.t3.micro` |
| Storage | 20 GiB gp3, tắt autoscaling |
| Multi-AZ | Do not create a standby |
| VPC | VPC đã tạo ở 5.3.1 |
| Public access | **No** |
| VPC security group | `vsp-rds-sg` |
| Availability Zone | `ap-southeast-1a` |
| Backup retention | 7 ngày |
| Encryption | Enabled |

![tạo rds](/images/5-Workshop/5.4-Data-layer/create-rds.png)

**Public access: No** là thiết lập quan trọng nhất ở đây. Nếu để Yes, RDS sẽ gán một IP công khai và database trở nên tiếp cận được từ Internet, chỉ còn security group và mật khẩu của bạn đứng ra bảo vệ.

Quá trình tạo mất khoảng 10 phút. 

Xong xuôi, copy endpoint ở tab **Connectivity & security**:

```
vsp-rds-postgresql.xxxxxxxxxxxx.ap-southeast-1.rds.amazonaws.com:5432
```

#### Amazon ElastiCache for Valkey

Hai cụm riêng biệt, mỗi service một cụm. Về lý thì dùng chung một cụm cũng được, nhưng hai loại tải này hành xử rất khác nhau: `api_service` dùng Redis làm broker cho BullMQ, nơi một lần flush sẽ làm mất toàn bộ job transcode đang chờ; còn `search_service` chỉ dùng nó làm cache kết quả có thể vứt đi bất cứ lúc nào. Tách riêng nghĩa là service này không thể đẩy dữ liệu của service kia ra khỏi bộ nhớ, và mỗi bên có thể đổi kích thước độc lập.

Sau đó tạo hai cụm với engine là **Valkey**:

| Mục | Cụm cho `api-service` | Cụm cho `search-service` |
|---|---|---|
| Name | `vsp-api-redis` | `vsp-search-redis` |
| Deployment | Design your own cache — Cluster mode disabled | như trên |
| Node type | `cache.t4g.micro` | `cache.t4g.micro` |
| Replicas | 0 | 0 |
| Security group | `vsp-redis-api-sg` | `vsp-redis-search-sg` |
| Encryption in transit | Enabled | Enabled |
| Encryption at rest | Enabled | Enabled |

![tạo elasticache](/images/5-Workshop/5.4-Data-layer/create-valkey.png)

{{% notice warning %}}
Nếu bật **encryption in transit**, ứng dụng bắt buộc phải kết nối bằng TLS. Trên thực tế nghĩa là chuỗi kết nối phải dùng `rediss://` thay vì `redis://`, và với `ioredis` hay BullMQ thì client cần thêm `tls: {}` trong phần tùy chọn. Một client cấu hình dạng plaintext sẽ treo ở bước kết nối chứ không trả về lỗi rõ ràng — quá trình bắt tay đơn giản là không bao giờ hoàn tất.
{{% /notice %}}

Copy **Primary endpoint** của từng cụm khi chúng chuyển sang `available`.

#### Amazon MQ for RabbitMQ

RabbitMQ đảm nhận việc đồng bộ metadata giữa hai service: khi một video transcode xong, `api_service` publish vào `video.metadata.trans`, `search_service` tiêu thụ thông điệp đó, sinh embedding, rồi trả lời qua `video.metadata.res`.

Console **Amazon MQ** → **Create brokers** → **RabbitMQ**:

| Mục | Giá trị |
|---|---|
| Deployment mode | **Single-instance broker** |
| Broker name | `vsp-mq` |
| Instance type | `mq.m7g.medium` |
| Username | `vspadmin` |
| Password | từ 12 ký tự trở lên, không dùng dấu phẩy, hai chấm hay dấu bằng |
| Access type | **Private access** |
| VPC | VPC đã tạo ở 5.3.1 |
| Subnet | private subnet |
| Security group | `vsp-rabbitmq-sg` |

![tạo amazon mq](/images/5-Workshop/5.4-Data-layer/create-mq.png)

**Private access** đặt broker vào bên trong VPC mà không có endpoint công khai nào. Kết hợp với VPC endpoint `mq` đã tạo ở 5.3.3, các ECS task truy cập nó hoàn toàn qua địa chỉ riêng.

Mật khẩu có những ràng buộc ký tự mà console chỉ báo sau khi bạn bấm gửi và bị từ chối — hãy tránh `,` `:` `=` và khoảng trắng.

Broker mất khoảng 15 phút để tạo. Khi sẵn sàng, copy endpoint AMQP:

```
amqps://b-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.mq.ap-southeast-1.amazonaws.com:5671
```

{{% notice note %}}
Amazon MQ là thành phần đắt nhất trong workshop này mà không có free tier — khoảng 40 - 50 USD mỗi tháng cho `mq.m7g.medium` chạy liên tục. Nó tính tiền bất kể có thông điệp nào đi qua hay không. Mục 5.8 hướng dẫn cách xóa nó.
{{% /notice %}}

#### Amazon S3

Một bucket duy nhất chứa cả file upload gốc lẫn output sau transcode. Console **S3** → **Create bucket**:

| Mục | Giá trị |
|---|---|
| Bucket name | `video-platform-<your-account-id>-ap-southeast-1` |
| Region | `ap-southeast-1` |
| Block all public access | **Enabled** |
| Bucket versioning | Disabled |
| Encryption | SSE-S3 |

Tên bucket là duy nhất trên toàn bộ AWS, không riêng tài khoản của bạn, nên hãy chèn account ID vào để tránh trùng.

**Giữ nguyên Block all public access.** CloudFront sẽ đọc từ bucket này qua Origin Access Control ở mục 5.6, tức cấp quyền bằng bucket policy chứ không phải bằng cách để object ở chế độ công khai. Không bao giờ có lý do gì để mở public cho chính bucket.

Vào tab permission thêm vào Bucket Policy

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

![tạo S3](/images/5-Workshop/5.4-Data-layer/create-s3.png)

#### Amazon EFS

Qdrant lưu chỉ mục vector xuống đĩa. Bộ nhớ của task Fargate là tạm thời, nên nếu không có volume bền vững thì mỗi lần deploy sẽ xóa sạch chỉ mục và buộc phải sinh lại embedding cho toàn bộ kho video.

Console **EFS** → **Create file system** → **Customize**:

| Mục | Giá trị |
|---|---|
| Name | `vsp-qdrant-efs` |
| VPC | VPC đã tạo ở 5.3.1 |
| Storage class | Regional |
| Automatic backups | Enabled |
| Lifecycle management | Chuyển sang IA sau 30 ngày |
| Encryption | Enabled |

Ở bước cấu hình mạng, xóa các mount target mặc định và **chỉ giữ lại** cái ở `ap-southeast-1a` trên subnet `10.0.10.0/24`. Tạo cho nó một security group tên `vsp-efs-sg`:

| Security group | Port | Source |
|---|---|---|
| `vsp-efs-sg` | 2049 | `vsp-qdrant-sg` |

Port 2049 là NFS. Chỉ task Qdrant cần tới nó.

![tạo efs security group](/images/5-Workshop/5.4-Data-layer/efs-sg.png)

![tạo efs](/images/5-Workshop/5.4-Data-layer/create-efs.png)

#### Lưu cấu hình vào Parameter Store

Mọi endpoint và thông tin đăng nhập vừa tạo ở trên giờ cần đến được với container. Thay vì nhúng thẳng vào task definition, nơi chúng sẽ hiện ra dưới dạng văn bản thuần trong console, hãy đưa chúng vào **Systems Manager Parameter Store** và để ECS tiêm vào lúc chạy.

Vào **Systems Manager** → **Parameter Store** → **Create parameter**, làm lần lượt từng dòng:

![tạo parameter store](/images/5-Workshop/5.4-Data-layer/para-store.png)

Dùng **SecureString** cho mọi thứ chứa mật khẩu hoặc khóa. Tham số SecureString được mã hóa bằng khóa KMS, và đó chính là lý do task execution role của ECS cần quyền `kms:Decrypt`, cũng như lý do VPC endpoint `kms` tồn tại.

Các hostname `.vsp.internal` hiện chưa phân giải được — Cloud Map namespace sẽ được tạo ở mục 5.5. Cứ điền vào bây giờ; chúng sẽ phân giải được đúng lúc container đọc tới.


