---
title : "Đưa hệ thống ra Internet với ALB và CloudFront"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

Ba service đã chạy nhưng chưa có gì bên ngoài VPC tiếp cận được chúng. Mục này dựng đường vào: một load balancer trong public subnet, distribution CloudFront đã thiết kế ở 5.3.4, và bản ghi DNS gắn nó với tên miền của bạn.

Chuỗi hoàn chỉnh trông như sau:

```
Trình duyệt → Route 53 → CloudFront → ALB      → ECS task
                                    → S3 (OAC) → file video
```

#### Tạo target group

Target group là nơi ALB chuyển tiếp lưu lượng tới. Tạo hai cái — console **EC2** → **Target groups** → **Create target group**.

| Mục | `api-service` | `search-service` |
|---|---|---|
| Target type | **IP addresses** | **IP addresses** |
| Name | `vsp-api-tg` | `vsp-search-tg` |
| Protocol / port | HTTP / 8080 | HTTP / 8000 |
| VPC | VPC đã tạo ở 5.3.1 | như trên |
| Protocol version | HTTP1 | HTTP1 |
| Health check path | `/health` | `/api/v1/health` |
| Healthy threshold | 2 | 2 |
| Unhealthy threshold | 3 | 3 |
| Timeout | 5 giây | 5 giây |
| Interval | 30 giây | 30 giây |

**Target type bắt buộc phải là IP addresses**, không phải Instances. Task Fargate không có EC2 instance nào đứng sau, nên target group kiểu instance hoàn toàn không đăng ký được chúng.

Đừng tự tay đăng ký target nào cả. ECS sẽ tự đăng ký và gỡ đăng ký IP của task mỗi khi task sinh ra hay biến mất — đó chính là mục đích của việc gắn service vào target group.

![api target](/images/5-Workshop/5.6-Public-access/api-target.png)

![search target](/images/5-Workshop/5.6-Public-access/search-target.png)

#### Tạo Application Load Balancer

Console **EC2** → **Load balancers** → **Create load balancer** → **Application Load Balancer**.

| Mục | Giá trị |
|---|---|
| Name | `vsp-alb` |
| Scheme | **Internet-facing** |
| IP address type | IPv4 |
| VPC | VPC đã tạo ở 5.3.1 |
| Mappings | `ap-southeast-1a` → `10.0.1.0/24`, `ap-southeast-1b` → `10.0.2.0/24` |
| Security group | `vsp-alb-sg` |
| Listener | HTTPS : 443 |
| Default action | Forward tới `vsp-api-tg` |
| Certificate | chứng chỉ `ap-southeast-1` từ mục 5.3.4 |
| Security policy | `ELBSecurityPolicy-TLS13-1-2-2021-06` |

Chọn **cả hai public subnet**. Đây chính là lý do mục 5.3.1 tạo ra hai cái — ALB không thể được tạo với chỉ một subnet.

ALB để chế độ internet-facing dù chỉ CloudFront được phép nói chuyện với nó. Một ALB nội bộ không dùng làm origin cho CloudFront được, vì CloudFront tiếp cận origin qua mạng công cộng. Sự bảo vệ đến từ security group và từ việc không bao giờ công bố tên miền của ALB, chứ không đến từ việc đặt nó thành internal.

#### Thêm listener rule cho search service

Default action đang gửi mọi thứ về API. Thêm một rule để lưu lượng tìm kiếm đi đúng service — mở listener HTTPS → **Rules** → **Add rule**:

| Mục | Giá trị |
|---|---|
| Priority | 100 |
| Condition | Path là `/api/v1/search*` |
| Action | Forward tới `vsp-search-tg` |

Rule được xét theo priority, số nhỏ trước, và default action chỉ chạy khi không rule nào khớp.

![alb](/images/5-Workshop/5.6-Public-access/alb.png)

#### Gắn ECS service vào target group

Quay lại **ECS** → `vsp-ecs-cluster` → **Services** → `api-service` → **Update**:

| Mục | Giá trị |
|---|---|
| Load balancer type | Application Load Balancer |
| Load balancer | `vsp-alb` |
| Container to load balance | `api-service-container` 8080 |
| Target group | `vsp-api-tg` |
| Health check grace period | 120 |

![ser tar api](/images/5-Workshop/5.6-Public-access/ser-tar-api.png)

![ser tar search](/images/5-Workshop/5.6-Public-access/ser-tar-search.png)

Làm tương tự với `search-service`: container `search-service-container` 8000, target group `vsp-search-tg`, và grace period **180** giây.

**Health check grace period** bảo ECS bỏ qua kết quả health check của load balancer trong khoảng thời gian đó sau khi task khởi động. Không có nó, ALB sẽ đánh dấu task đang boot là unhealthy, ECS giết nó đi, khởi động cái khác, và service không bao giờ ổn định được. Search service cần khoảng thời gian dài hơn vì phải nạp mô hình.

Chờ cả hai target group hiển thị `healthy`:

![target khỏe mạnh](/images/5-Workshop/5.6-Public-access/health.png)


#### Tạo Origin Access Control

Trước khi tạo distribution, hãy tạo danh tính mà CloudFront dùng để đọc từ S3. Console **CloudFront** → **Origin access** → **Create control setting**:

| Mục | Giá trị |
|---|---|
| Name | `vsp-s3-oac` |
| Origin type | S3 |
| Signing behavior | Sign requests (recommended) |

Origin Access Control là bản thay thế cho Origin Access Identity cũ, và nó chính là thứ cho phép bucket giữ trạng thái hoàn toàn riêng tư trong khi CloudFront vẫn đọc được.

![origin access s3](/images/5-Workshop/5.6-Public-access/oas3.png)

#### Tạo distribution

**CloudFront** → **Create distribution**.

**Origins** — thêm hai cái:

| | ALB origin | S3 origin |
|---|---|---|
| Origin domain | `vsp-alb-…elb.amazonaws.com` | bucket của bạn |
| Origin path | để trống | để trống |
| Protocol | HTTPS only | — |
| Origin access | — | Origin access control, `vsp-s3-oac` |
| Name | `alb-origin` | `s3-origin` |

**Default cache behavior** — cái này phục vụ API:

| Mục | Giá trị |
|---|---|
| Origin | `alb-origin` |
| Viewer protocol policy | Redirect HTTP to HTTPS |
| Allowed methods | GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE |
| Cache policy | **CachingDisabled** |
| Origin request policy | **AllViewerExceptHostHeader** |

`AllViewerExceptHostHeader` chuyển tiếp mọi header, cookie và query string về ALB, trừ `Host`. Phần ngoại lệ này rất quan trọng: chuyển tiếp header `Host` của người dùng tới một ALB origin sẽ phá vỡ định tuyến, vì ALB mong đợi chính hostname của nó.

**Additional behaviors** — thêm hai cái nữa:

| Path pattern | Origin | Cache policy | Methods |
|---|---|---|---|
| `/public/*` | `s3-origin` | CachingOptimized | GET, HEAD |
| `/private/*` | `s3-origin` | CachingOptimized | GET, HEAD |

**Settings**:

| Mục | Giá trị |
|---|---|
| Price class | Use only North America, Europe, Asia |
| Alternate domain name (CNAME) | `app.example.com` |
| Custom SSL certificate | chứng chỉ **`us-east-1`** từ mục 5.3.4 |
| Security policy | TLSv1.2_2021 |
| Default root object | để trống |

![behavior của cloudfront](/images/5-Workshop/5.6-Public-access/cloudfront-behaviors.png)

#### Cập nhật bucket policy cho S3

Sau khi distribution được tạo, CloudFront sẽ hiển thị một bucket policy sinh sẵn — copy nó, hoặc tự viết:

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

Dán vào **S3** → bucket của bạn → **Permissions** → **Bucket policy**. Thực hiện bước phía trên.

#### Trỏ tên miền về CloudFront

**Route 53** → hosted zone của bạn → **Create record**:

| Mục | Giá trị |
|---|---|
| Record name | `app` |
| Record type | A |
| Alias | Yes |
| Route traffic to | Alias to CloudFront distribution |
| Distribution | distribution của bạn |

![bản ghi alias route53](/images/5-Workshop/5.6-Public-access/record.png)

#### Kiểm tra toàn bộ đường đi

```
dig app.example.com +short
curl -I https://app.example.com/health
curl -s https://app.example.com/api/v1/health
```

Lệnh đầu phải trả về các địa chỉ IP của CloudFront; lệnh thứ hai trả `200 OK` từ API qua ALB; lệnh thứ ba trả về search service, xác nhận listener rule hoạt động.

![ping](/images/5-Workshop/5.6-Public-access/ping.png)
