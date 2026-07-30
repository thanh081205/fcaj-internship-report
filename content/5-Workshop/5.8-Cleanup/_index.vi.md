---
title : "Dọn dẹp tài nguyên"
date : 2024-01-01
weight : 8
chapter : false
pre : " <b> 5.8. </b> "
---

Phần lớn kiến trúc này tính tiền theo giờ bất kể có ai dùng hay không. Amazon MQ, RDS, ElastiCache, ALB và tám interface endpoint vẫn tính phí trong lúc hoàn toàn nhàn rỗi — cộng lại vào khoảng 200 USD mỗi tháng nếu để nguyên.

Hãy xóa mọi thứ theo đúng thứ tự dưới đây. Thứ tự có ý nghĩa: AWS từ chối xóa một tài nguyên khi vẫn còn thứ khác phụ thuộc vào nó, và đi ngược chuỗi phụ thuộc sẽ tránh được một loạt lần xóa thất bại.

{{% notice warning %}}
Thao tác này xóa vĩnh viễn toàn bộ dữ liệu — database, chỉ mục vector, và mọi video đã upload. Nếu muốn giữ lại thứ gì, hãy tạo snapshot cuối cho RDS và tải các object trên S3 về trước khi bắt đầu.
{{% /notice %}}

#### 1. Dừng pipeline

Tắt workflow trước tiên, nếu không một lần push sau đó sẽ tạo lại tài nguyên ngay giữa lúc bạn đang dọn dẹp.

Trên GitHub: **Actions** → **Deploy to ECS** → **⋯** → **Disable workflow**.

#### 2. Xóa distribution CloudFront

CloudFront chậm, nên hãy bắt đầu sớm và để nó chạy trong lúc bạn xóa những thứ khác.

**CloudFront** → distribution của bạn → **Disable**, chờ trạng thái chuyển sang `Deployed` (có thể tới 15 phút), rồi **Delete**.

Không thể xóa một distribution khi nó còn đang bật. Không có cách nào bỏ qua bước disable hay làm nó nhanh hơn.

#### 3. Xóa bản ghi Route 53

**Route 53** → hosted zone của bạn → chọn bản ghi A tên `app` → **Delete record**.

Giữ lại chính hosted zone nếu bạn còn định dùng tên miền — nó tốn 0.50 USD mỗi tháng. Chỉ xóa khi bạn dứt hẳn với tên miền đó, và lưu ý rằng không thể xóa một zone vẫn còn bản ghi khác ngoài `NS` và `SOA` mặc định.

#### 4. Xóa load balancer và target group

**EC2** → **Load balancers** → `vsp-alb` → **Delete**.

Sau đó vào **Target groups** → xóa `vsp-api-tg` và `vsp-search-tg`.

Target group phải xóa sau ALB; một target group đang gắn vào listener thì không xóa được.

#### 5. Xóa ECS service và cluster

Hạ số task của từng service về 0, rồi xóa:

```
for SVC in api-service search-service qdrant; do
  aws ecs update-service --cluster vsp-ecs-cluster --service $SVC --desired-count 0
done

for SVC in api-service search-service qdrant; do
  aws ecs delete-service --cluster vsp-ecs-cluster --service $SVC --force
done

aws ecs delete-cluster --cluster vsp-ecs-cluster
```

`--force` cho phép xóa service mà không cần hạ số task trước, nhưng hạ về 0 trước vẫn khiến quá trình xóa nhanh và gọn hơn.

Task definition không tốn phí nên có thể để lại. Nếu vẫn muốn dọn, hãy deregister từng revision:

```
aws ecs list-task-definitions --family-prefix vsp-api-service --query "taskDefinitionArns" --output text
aws ecs deregister-task-definition --task-definition vsp-api-service:1
```

#### 6. Xóa Cloud Map namespace

**AWS Cloud Map** → `vsp.internal` → xóa ba service đã đăng ký trước (`api-service`, `search-service`, `qdrant`), rồi mới xóa namespace.

Một namespace còn service đăng ký thì không xóa được. Nếu các service không chịu xóa, khả năng cao vẫn còn một ECS service đang giữ đăng ký — hãy kiểm tra lại bước 5 đã hoàn tất chưa.

#### 7. Xóa tầng dữ liệu

Đây mới là chỗ thực sự tốn tiền.

**Amazon MQ** → `vsp-mq` → **Delete**. Việc xóa broker mất vài phút.

**RDS** → `vsp-rds-postgresql` → **Delete**. Bỏ tick **Create final snapshot** trừ khi bạn muốn giữ dữ liệu — snapshot vẫn bị tính phí lưu trữ. Bạn phải gõ `delete me` để xác nhận.

**ElastiCache** → xóa `vsp-api-redis` và `vsp-search-redis`, chọn không tạo backup cuối.

**EFS** → `vsp-qdrant-efs` → xóa mount target trước, rồi mới xóa file system.

**S3** → bucket phải rỗng thì mới xóa được:

```
aws s3 rm s3://<your-bucket> --recursive
aws s3 rb s3://<your-bucket>
```

#### 8. Xóa VPC endpoint

Tám interface endpoint, mỗi cái khoảng 9 USD một tháng, khiến đây là khoản lớn thứ hai sau tầng dữ liệu.

```
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=<your-vpc-id>" \
  --query "VpcEndpoints[].VpcEndpointId" --output text
```

```
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <id1> <id2> <id3> ...
```

Gateway endpoint của S3 tuy miễn phí nhưng cũng nên xóa — một gateway endpoint sẽ chặn việc xóa route table mà nó đang gắn vào.

#### 9. Xóa ECR repository

```
aws ecr delete-repository --repository-name video-platform/api-service --force
aws ecr delete-repository --repository-name video-platform/search-service --force
```

Cần `--force` vì trong repository vẫn còn image.

#### 10. Xóa phần mạng

Giờ mới tới lượt VPC. Xóa VPC từ console sẽ kéo theo subnet, route table và Internet Gateway, nhưng **không** kéo theo những security group còn tham chiếu lẫn nhau.

Xóa security group trước, theo thứ tự sau:

1. `vsp-vpce-sg`, `vsp-efs-sg`
2. `vsp-rds-sg`, `vsp-redis-api-sg`, `vsp-redis-search-sg`, `vsp-rabbitmq-sg`, `vsp-qdrant-sg`
3. `vsp-grpc-api-sg` và `vsp-grpc-search-sg` — **hãy gỡ inbound rule của cả hai trước khi xóa bất kỳ cái nào**, vì chúng tham chiếu lẫn nhau
4. `vsp-ecs-tasks-sg`
5. `vsp-alb-sg`

Sau đó vào **VPC** → VPC của bạn → **Delete VPC**.

{{% notice note %}}
Lỗi `DependencyViolation` trên một security group gần như luôn có nghĩa là vẫn còn một network interface tồn tại. ECS task và RDS instance sau khi xóa vẫn để lại ENI trong vài phút. Hãy chờ rồi thử lại — hoặc tìm ra thứ đang giữ nó bằng `aws ec2 describe-network-interfaces --filters "Name=group-id,Values=<sg-id>"`.
{{% /notice %}}

#### 11. Xóa chứng chỉ, tham số và IAM role

**ACM** → xóa cả hai chứng chỉ, ở `us-east-1` và `ap-southeast-1`. Một chứng chỉ đang được distribution sử dụng thì không xóa được, nên bước này phải làm sau bước 2.

**Systems Manager** → Parameter Store → xóa mọi thứ dưới `/vsp/`:

```
aws ssm get-parameters-by-path --path /vsp --recursive \
  --query "Parameters[].Name" --output text \
  | xargs -n 10 aws ssm delete-parameters --names
```

**CloudWatch** → xóa ba log group `/ecs/vsp-*`.

**IAM** → xóa `ecsTaskExecutionRole`, `vspTaskRole` và `GitHubActionsDeployRole`, cùng với OIDC provider nếu bạn không còn repository nào khác dùng tới nó.

#### Kiểm tra xem còn sót gì không

```
aws ecs list-clusters
aws rds describe-db-instances --query "DBInstances[].DBInstanceIdentifier"
aws elasticache describe-cache-clusters --query "CacheClusters[].CacheClusterId"
aws mq list-brokers --query "BrokerSummaries[].BrokerName"
aws elbv2 describe-load-balancers --query "LoadBalancers[].LoadBalancerName"
aws efs describe-file-systems --query "FileSystems[].FileSystemId"
aws s3 ls
```

Tất cả đều phải trả về rỗng, ngoại trừ những tài nguyên không liên quan tới workshop này.

#### Kiểm tra hóa đơn

Việc xóa tài nguyên không xóa đi những khoản đã phát sinh. Mở **AWS Billing** → **Bills** và xác nhận tháng hiện tại đúng như bạn hình dung, rồi kiểm tra lại sau một hai ngày nữa — một số dịch vụ báo cáo mức sử dụng có độ trễ.

Nếu budget alert từ mục 5.2 vẫn đang bật, hãy để nguyên. Nó không tốn gì và sẽ báo cho bạn nếu có thứ gì bị bỏ sót.

#### Tóm tắt thứ tự xóa

| Thứ tự | Tài nguyên | Vì sao ở vị trí này |
|---|---|---|
| 1 | GitHub workflow | Tránh deploy lại giữa lúc đang dọn |
| 2 | CloudFront | Chậm; chặn việc xóa chứng chỉ |
| 3 | Bản ghi Route 53 | Đang trỏ tới CloudFront |
| 4 | ALB, target group | Target group bị listener giữ |
| 5 | ECS service, cluster | Đang giữ ENI và đăng ký service |
| 6 | Cloud Map namespace | Bị chặn bởi các service đã đăng ký |
| 7 | RDS, ElastiCache, MQ, EFS, S3 | Phần lớn chi phí nằm ở đây |
| 8 | VPC endpoint | Gateway endpoint chặn route table |
| 9 | ECR repository | Độc lập |
| 10 | Security group, VPC | Bị chặn bởi ENI còn sót |
| 11 | ACM, SSM, log, IAM | Chứng chỉ bị CloudFront chặn |

#### Vậy là hết workshop

Bạn đã dựng một VPC riêng tư không dùng NAT Gateway, một tầng dữ liệu được quản lý, ba service container tìm thấy nhau qua Cloud Map, một đường vào công khai có CDN đứng trước, và một pipeline CI/CD không cần khóa truy cập — rồi tháo dỡ toàn bộ mà không để sót thứ gì âm thầm tính tiền phía sau.
