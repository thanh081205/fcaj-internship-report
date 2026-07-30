---
title : "Giới thiệu"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

#### Amazon ECS trên AWS Fargate

+ **Amazon ECS** là dịch vụ điều phối container được quản lý hoàn toàn. Bạn mô tả ứng dụng của mình bằng một *task definition* (chạy image nào, cần bao nhiêu CPU và bộ nhớ, mở cổng nào), và ECS duy trì đúng số bản sao bạn yêu cầu dưới dạng một *service*.
+ **AWS Fargate** là engine tính toán serverless đứng sau ECS. Không có EC2 instance nào để vá lỗi hay scale: bạn khai báo CPU và bộ nhớ cho mỗi task, AWS lo phần hạ tầng bên dưới. Bạn chỉ trả tiền cho đúng lượng tài nguyên task yêu cầu, trong đúng khoảng thời gian nó chạy.
+ **Fargate Spot** cung cấp cùng năng lực tính toán với giá thấp hơn đáng kể, đổi lại AWS có thể thu hồi tài nguyên sau khi báo trước hai phút. Nó phù hợp với các service không lưu trạng thái, chịu được việc khởi động lại ngắn.
+ Mỗi Fargate task có một elastic network interface (ENI) riêng bên trong VPC của bạn, nên nó hoạt động như mọi tài nguyên khác trên mạng: security group áp dụng được cho nó, và nó kết nối trực tiếp tới các dịch vụ private như RDS.

#### VPC Endpoint và lý do không dùng NAT Gateway

Container chạy trong **private subnet** không có đường ra Internet. Tuy vậy chúng vẫn cần truy cập các dịch vụ AWS — để kéo image từ ECR, ghi log lên CloudWatch, hay đọc cấu hình từ Systems Manager. Có hai cách phổ biến để giải quyết:

+ **NAT Gateway** cấp cho private subnet đường ra Internet. Cách này đơn giản, nhưng bị tính phí theo giờ *và* theo mỗi gigabyte đi qua, đồng thời lưu lượng vẫn đi ra Internet công cộng.
+ **VPC Endpoint** giữ lưu lượng nằm trong mạng nội bộ của AWS. **Interface endpoint** (dùng AWS PrivateLink) đặt thẳng một ENI của dịch vụ vào subnet của bạn, còn **gateway endpoint** thêm một route cho Amazon S3 vào route table mà không mất phí.

Trong workshop này chúng ta dùng VPC Endpoint và **hoàn toàn không tạo NAT Gateway**. Cách này giữ workload tách biệt khỏi Internet công cộng, đồng thời làm đường đi của lưu lượng trở nên tường minh: mọi dịch vụ AWS mà task có thể chạm tới đều là dịch vụ bạn đã chủ động tạo endpoint cho nó.

#### Kiến trúc hệ thống

Hệ thống gồm hai application service độc lập cùng một vector database tự host, tất cả chạy dưới dạng ECS Fargate service trong cùng một private subnet:

+ **`api-service`** (NestJS + Apollo GraphQL) — xử lý xác thực, CRUD video, sinh presigned URL upload lên S3, và chạy worker transcode bất đồng bộ. Service lắng nghe cổng `8080` cho HTTP/GraphQL và `50051` cho gRPC.
+ **`search-service`** (Python FastAPI) — xử lý tìm kiếm lai và giữ cho chỉ mục vector luôn đồng bộ. Service lắng nghe cổng `8000` cho HTTP và `50052` cho gRPC.
+ **`qdrant`** — vector database tự host, lưu embedding của video, dữ liệu được ghi bền vững trên một file system Amazon EFS để chỉ mục không mất khi task khởi động lại.

Bao quanh chúng là các dịch vụ được quản lý: **Amazon RDS for PostgreSQL** cho metadata quan hệ, **Amazon ElastiCache for Valkey** cho session, cache và hàng đợi job BullMQ, **Amazon MQ (RabbitMQ)** cho message bất đồng bộ giữa hai service, và **Amazon S3** cho file gốc lẫn file video đã transcode.

Lưu lượng từ bên ngoài đi vào qua **Amazon CloudFront**, đứng trước **Application Load Balancer** cho phần API và phục vụ trực tiếp các file video từ S3. Domain riêng do **Route 53** cung cấp, kèm chứng chỉ TLS từ **AWS Certificate Manager**.

#### Các phần tiếp theo

Những phần còn lại sẽ dựng kiến trúc này từ dưới lên: trước hết là mạng, rồi tới tầng dữ liệu, sau đó là các container chạy bên trên, và cuối cùng là đường vào công khai cùng pipeline triển khai. Mỗi phần đều có thể làm độc lập, nhưng nên theo đúng thứ tự, vì các bước sau có tham chiếu tới tài nguyên đã tạo ở bước trước