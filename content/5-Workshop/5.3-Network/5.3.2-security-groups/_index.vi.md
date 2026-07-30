---
title : "Tạo các security group"
date : 2024-01-01
weight : 2
chapter : false
pre : " <b> 5.3.2 </b> "
---

Security group mới chính là tầng kiểm soát truy cập thật sự trong kiến trúc này. Vì mọi thứ đều nằm chung trên một private subnet, ranh giới subnet không hề tách biệt database với cache hay với container — chính security group mới làm việc đó.

#### Nguyên tắc cốt lõi

Mọi rule trong mục này đều cho phép lưu lượng **từ một security group khác**, chứ không phải từ một dải CIDR. Ngoại lệ duy nhất là load balancer, vốn buộc phải nhận lưu lượng từ Internet.

Điều này đặc biệt quan trọng với Fargate. Task nhận một IP riêng mới sau mỗi lần deploy, và có thể có nhiều task cùng lúc phía sau một service đang auto-scaling. Một rule viết theo `10.0.10.0/24` vẫn chạy được, nhưng đồng thời cũng cho phép bất kỳ tài nguyên nào khác trên subnet đó truy cập database trong tương lai. Một rule viết theo security group của ECS task chỉ cấp quyền cho đúng thứ cần nó, và vẫn hoạt động bình thường khi IP thay đổi liên tục.

#### Danh sách các group

Tạo chín security group sau trong VPC đã dựng ở bước trước. Nhớ điền mô tả cho từng group — console bắt buộc phải có, và đó cũng là nơi duy nhất ghi lại mục đích của nó.

| Tên | Gắn cho |
|---|---|
| `vsp-alb-sg` | Application Load Balancer |
| `vsp-ecs-tasks-sg` | Task `api-service` và `search-service` |
| `vsp-qdrant-sg` | Task Qdrant |
| `vsp-grpc-api-sg` | Task `api-service` (cổng lắng nghe gRPC) |
| `vsp-grpc-search-sg` | Task `search-service` (cổng lắng nghe gRPC) |
| `vsp-rds-sg` | Instance RDS PostgreSQL |
| `vsp-redis-api-sg` | Cụm ElastiCache của `api-service` |
| `vsp-redis-search-sg` | Cụm ElastiCache của `search-service` |
| `vsp-rabbitmq-sg` | Broker Amazon MQ |

Cách tạo: console **VPC** → **Security groups** → **Create security group**. Điền tên, mô tả và chọn đúng VPC. Tạm thời để phần inbound rule trống.

![tạo security group](/images/5-Workshop/5.3-Network/create-sg.png)

{{% notice note %}}
Hãy tạo đủ cả chín group với rule rỗng trước, rồi mới quay lại thêm rule. Nhiều rule tham chiếu tới group khác, và có hai group tham chiếu lẫn nhau, nên một group phải tồn tại trước thì mới được chọn làm source.
{{% /notice %}}

#### Các inbound rule

Sau khi đủ chín group, thêm các inbound rule dưới đây. Mọi thứ không có trong bảng đều đóng; phần outbound cứ để nguyên mặc định cho phép tất cả.

| Security group | Port | Giao thức | Source | Mục đích |
|---|---|---|---|---|
| `vsp-alb-sg` | 443 | TCP | `0.0.0.0/0` | HTTPS từ CloudFront và client |
| `vsp-ecs-tasks-sg` | 8080 | TCP | `vsp-alb-sg` | ALB → `api-service` (GraphQL và REST) |
| `vsp-ecs-tasks-sg` | 8000 | TCP | `vsp-alb-sg` | ALB → `search-service` (FastAPI) |
| `vsp-grpc-api-sg` | 50051 | TCP | `vsp-grpc-search-sg` | `search-service` → `api-service` qua gRPC |
| `vsp-grpc-search-sg` | 50052 | TCP | `vsp-grpc-api-sg` | `api-service` → `search-service` qua gRPC |
| `vsp-qdrant-sg` | 6333 | TCP | `vsp-ecs-tasks-sg` | Qdrant HTTP API |
| `vsp-qdrant-sg` | 6334 | TCP | `vsp-ecs-tasks-sg` | Qdrant gRPC API |
| `vsp-rds-sg` | 5432 | TCP | `vsp-ecs-tasks-sg` | PostgreSQL |
| `vsp-redis-api-sg` | 6379 | TCP | `vsp-ecs-tasks-sg` | Valkey cho session, cache và BullMQ |
| `vsp-redis-search-sg` | 6379 | TCP | `vsp-ecs-tasks-sg` | Valkey cho cache tìm kiếm |
| `vsp-rabbitmq-sg` | 5671 | TCP | `vsp-ecs-tasks-sg` | AMQPS tới Amazon MQ |
| `vsp-rabbitmq-sg` | 443 | TCP | `vsp-ecs-tasks-sg` | Giao diện quản trị RabbitMQ qua HTTPS |

![danh sách security group](/images/5-Workshop/5.3-Network/sg-list.png)

#### Tham chiếu vòng giữa hai group gRPC

`vsp-grpc-api-sg` cho phép port 50051 từ `vsp-grpc-search-sg`, còn `vsp-grpc-search-sg` cho phép port 50052 từ `vsp-grpc-api-sg`. Mỗi group lấy group kia làm source.

AWS cho phép cấu hình này, nhưng không thể tạo trong một lượt — group đầu tiên không thể tham chiếu tới một group chưa tồn tại. Hãy tạo cả hai group rỗng trước, rồi thêm rule cho từng cái. Nếu bạn viết script bằng CLI hoặc CloudFormation thì thứ tự cũng tương tự: tạo group trước, gắn rule sau, như hai thao tác tách biệt.

Hai service này thực sự gọi lẫn nhau. `api_service` cung cấp một RPC về metadata cho `search_service` sử dụng, còn `search_service` cung cấp một RPC xóa video cho `api_service` sử dụng — nên lưu lượng đúng là chạy theo cả hai chiều, trên hai port khác nhau.

#### Vì sao phải tách riêng group cho gRPC

Hai task `api-service` và `search-service` vốn đã dùng chung `vsp-ecs-tasks-sg` cho các port HTTP, nên về lý thì mở luôn hai port gRPC ở đó sẽ đơn giản hơn. Lý do không làm vậy là: một group dùng chung có sẵn 50051 và 50052 sẽ đồng thời cho phép mỗi service truy cập chính port gRPC của mình, và cấp luôn cả hai port cho bất kỳ task nào gắn vào group đó về sau. Tách riêng giúp mỗi đường RPC được khai báo tường minh và chỉ đi một chiều trên mỗi port, để sau này đọc lại vẫn hiểu ngay ý đồ.

Một task có thể thuộc nhiều security group cùng lúc, nên `api-service` sẽ được gắn cả `vsp-ecs-tasks-sg` lẫn `vsp-grpc-api-sg`. Việc gắn này thực hiện khi tạo ECS service ở mục 5.5.

#### Kiểm chứng

Xác nhận các rule đã được tạo đúng:

```
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=<your-vpc-id>" \
  --query "SecurityGroups[].{Name:GroupName,ID:GroupId}" \
  --output table
```

Sau đó xem chi tiết một group, ví dụ group của database:

```
aws ec2 describe-security-groups \
  --group-ids <vsp-rds-sg-id> \
  --query "SecurityGroups[].IpPermissions"
```

Source phải xuất hiện trong `UserIdGroupPairs` kèm ID của group ECS task — chứ không phải trong `IpRanges`. Nếu nó hiện ra dưới dạng một dải CIDR, nghĩa là rule đã bị nhập theo dải địa chỉ và sẽ hỏng ngay lần tiếp theo IP của task thay đổi.
