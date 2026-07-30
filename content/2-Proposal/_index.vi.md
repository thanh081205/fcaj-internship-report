---
title: "Bản đề xuất"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# VideoPlatformServer
## Nền tảng chia sẻ video với tìm kiếm ngữ nghĩa, triển khai trên AWS ECS Fargate

### 1. Tóm tắt điều hành
VideoPlatformServer là dự án cá nhân được xây dựng trong kỳ thực tập nhằm thực hành kiến trúc microservice thực chiến trên AWS. Hệ thống gồm 2 service độc lập giao tiếp qua gRPC song hướng: `api_service` (NestJS 11 + Apollo GraphQL) đảm nhận xác thực, quản lý video, upload/transcode; `search_service` (Python FastAPI) đảm nhận tìm kiếm lai (kết hợp từ khóa và vector ngữ nghĩa qua Qdrant). Toàn bộ hạ tầng chạy trên AWS ECS Fargate, đứng sau CloudFront + Application Load Balancer, dùng RDS PostgreSQL, ElastiCache (Valkey), Amazon MQ (RabbitMQ) cho giao tiếp bất đồng bộ liên service, và CI/CD tự động qua GitHub Actions.

### 2. Tuyên bố vấn đề
*Vấn đề hiện tại*
Phần lớn dự án cá nhân/học tập về AWS chỉ dừng ở mức serverless đơn giản (Lambda + API Gateway + S3), thiếu các bài toán khó gặp trong hệ thống production thật: xử lý file lớn bất đồng bộ, giao tiếp liên service qua gRPC/message queue, tìm kiếm ngữ nghĩa bằng vector database, và vận hành container trên nhiều môi trường (local Docker Compose, staging, production AWS).

*Giải pháp*
Hệ thống này giải quyết các bài toán đó bằng kiến trúc 2 service tách biệt: `api_service` xử lý luồng upload video qua presigned URL lên S3, sau đó enqueue job transcode bất đồng bộ bằng BullMQ (chạy trên Redis/ElastiCache Valkey) để chuyển video sang định dạng MPEG-DASH bằng FFmpeg. Khi metadata video thay đổi, `api_service` publish message qua RabbitMQ (Amazon MQ) để `search_service` đồng bộ lại chỉ mục; `search_service` lấy metadata đầy đủ qua gRPC, sinh embedding và upsert vào Qdrant vector database để phục vụ tìm kiếm lai (kết hợp điểm từ khóa và độ tương đồng vector). Toàn bộ được triển khai container hoá trên ECS Fargate, có CDN (CloudFront) và domain riêng (Route 53 + ACM) phía trước.

*Lợi ích và giá trị*
Dự án mang lại kỹ năng thực chiến về thiết kế microservice, giao tiếp bất đồng bộ, vector search/AI, và vận hành AWS production (IAM/OIDC, VPC networking, container orchestration, CI/CD) — vượt xa phạm vi một demo serverless đơn giản. Sản phẩm hoàn chỉnh có thể dùng làm dự án portfolio, đồng thời là nền tảng để mở rộng thêm tính năng (recommendation, live streaming...) sau kỳ thực tập.

### 3. Kiến trúc giải pháp
Người dùng truy cập qua domain riêng (Route 53) → CloudFront (CDN, TLS) → Application Load Balancer (định tuyến theo path `/graphql/*` cho `api_service`, `/api/*` cho `search_service`) → ECS Fargate cluster gồm 3 service: `api-service`, `search-service`, và `qdrant` (vector database tự triển khai, lưu dữ liệu bền vững trên EFS). Hai service ứng dụng giao tiếp gRPC song hướng và cùng kết nối tới RDS PostgreSQL (lưu metadata quan hệ), ElastiCache Valkey (session, hàng đợi BullMQ, cache), và Amazon MQ (đồng bộ metadata bất đồng bộ). Video gốc và bản đã transcode lưu trên S3, phục vụ qua CloudFront. Toàn bộ compute chạy trong private subnet, ra ngoài qua VPC Endpoints thay vì NAT Gateway để tối ưu chi phí.

![Solution Architecture](/images/2-Proposal/solution_architecture.jpg)

*Dịch vụ AWS sử dụng*
- *Amazon ECS (Fargate/Fargate Spot)*: chạy container cho `api-service`, `search-service`, `qdrant`.
- *Amazon ECR*: lưu trữ container image, quét lỗ hổng tự động (scan-on-push).
- *Application Load Balancer + Amazon CloudFront*: định tuyến HTTP/GraphQL và CDN cho video.
- *Amazon Route 53 + AWS Certificate Manager*: domain riêng và chứng chỉ TLS.
- *Amazon RDS for PostgreSQL*: cơ sở dữ liệu quan hệ cho cả 2 service.
- *Amazon ElastiCache (Valkey)*: session, hàng đợi BullMQ, cache.
- *Amazon MQ (RabbitMQ)*: giao tiếp bất đồng bộ liên service.
- *Amazon EFS*: lưu trữ bền vững cho Qdrant vector database.
- *Amazon S3*: lưu video gốc và bản đã transcode (MPEG-DASH).
- *VPC Endpoints*: cho phép ECS task truy cập ECR/CloudWatch/SSM/Amazon MQ mà không cần NAT Gateway.
- *AWS Systems Manager Parameter Store*: quản lý biến môi trường/secret dạng SecureString.
- *AWS IAM (OIDC)*: GitHub Actions xác thực với AWS qua federated role, không dùng access key tĩnh.
- *Amazon CloudWatch + AWS Backup*: log tập trung và sao lưu tự động cho EFS.

*Thiết kế thành phần*
- *api_service (NestJS + GraphQL)*: xác thực người dùng, quản lý video, sinh presigned URL upload, enqueue job transcode, expose GraphQL API, serve gRPC cho metadata.
- *search_service (FastAPI)*: consume message từ RabbitMQ, gọi gRPC lấy metadata, sinh embedding, upsert Qdrant, expose REST API tìm kiếm lai.
- *Qdrant*: vector database tự host trên Fargate, lưu embedding video, dữ liệu bền vững qua EFS mount.
- *Worker transcode*: BullMQ worker (trong `api_service`) tải video thô từ S3, chạy FFmpeg chuyển sang DASH, tải kết quả lên S3.

### 4. Triển khai kỹ thuật
*Các giai đoạn triển khai*
Dự án gồm 2 phần — xây dựng ứng dụng (2 service + database schema) và thiết lập hạ tầng và deploy lên AWS — mỗi phần trải qua 3 giai đoạn:
1. *Phát triển, kiểm thử*: code 2 service, viết test, đóng gói Docker, thiết lập CI/CD (GitHub Actions).
2. *Nghiên cứu và thiết kế kiến trúc, ước tính chi phí*: nghiên cứu hạ tầng trên aws, lựa chọn các dịch vụ sao tối ưu chi phí nhất.
3. *Xây dựng hạ tầng trên aws, deploy*: tạo các dịch vụ đã chọn kết lại với nhau, deploy và điều chỉnh.

*Yêu cầu kỹ thuật*
- *api_service*: NestJS 11, Prisma ORM, GraphQL (Apollo), BullMQ, FFmpeg, AWS SDK (S3 presigned URL), gRPC server/client.
- *search_service*: Python FastAPI, gRPC, thư viện sinh embedding (fastembed), Qdrant client, RabbitMQ consumer/publisher.
- *Hạ tầng*: Docker multi-stage build cho cả 2 service, ECS task definition (Fargate), GitHub Actions với OIDC role để build/push ECR và deploy ECS không cần static AWS key.

### 5. Ước tính ngân sách
Ước tính dựa trên bảng giá công khai AWS khu vực **ap-southeast-1** tại thời điểm viết đề xuất, theo đúng kiến trúc dự định (chưa trừ AWS Free Tier, chi phí thực tế có thể thay đổi theo lưu lượng sử dụng).

*Chi phí hạ tầng (ước tính hàng tháng)*
- Amazon ECS Fargate/Fargate Spot (3 task: api-service 2 vCPU/5GB, search-service 4 vCPU/9GB, qdrant 0,25 vCPU/2GB): ~56 USD
- VPC Interface Endpoints: ~20 USD
- Amazon MQ (mq.m7g.medium, single-instance): ~50 USD
- Amazon ElastiCache for Valkey (2× cache.t4g.micro, 1 cụm/service): ~30 USD
- Amazon RDS for PostgreSQL (db.t3.micro, 20GB gp3): ~17 USD
- Amazon CloudFront (CDN): ~3 USD
- Amazon EFS + AWS Backup: ~2 USD
- Amazon S3 (lưu trữ + request): ~1 USD
- Amazon Route 53 (2 hosted zone): ~1 USD
- Amazon ECR + Amazon CloudWatch Logs: ~12 USD
- AWS Certificate Manager, SSM Parameter Store, IAM/OIDC: miễn phí

*Tổng*: ~182 USD/tháng

### 6. Đánh giá rủi ro
*Ma trận rủi ro*
- Lỗi đồng bộ giữa 2 service qua gRPC/RabbitMQ: Ảnh hưởng cao, xác suất trung bình.
- Chi phí của dịch vụ Amaxon MQ khá cao nếu không quản lí tốt thì có thể gây tổn thất tiền.
- RDS/ElastiCache chạy single-AZ (không Multi-AZ): Ảnh hưởng cao nếu mất AZ, xác suất thấp.
- Vượt ngân sách cá nhân trong lúc thực tập: Ảnh hưởng trung bình, xác suất trung bình.

*Chiến lược giảm thiểu*
- Đồng bộ liên service: dùng dead-letter queue cho RabbitMQ, retry có giới hạn, log correlation ID để debug.
- Khả dụng: cân nhắc bật Multi-AZ cho RDS nếu ngân sách cho phép sau khi đánh giá chi phí thực tế.
- Ngân sách: bật AWS Budget Alert, ưu tiên Fargate Spot cho các task chịu được gián đoạn ngắn.

*Kế hoạch dự phòng*
- Rollback tự động qua ECS deployment circuit breaker nếu bản deploy mới lỗi health check.
- Giữ lại task definition revision trước đó để revert thủ công nếu cần.

### 7. Kết quả kỳ vọng
*Cải tiến kỹ thuật*: Có một hệ thống microservice hoàn chỉnh chạy trên aws, và CI/CD tự động.
*Giá trị dài hạn*: Nền tảng kỹ năng kiến trúc microservice + AWS production ops có thể tái sử dụng cho các dự án sau này, sản phẩm portfolio kỹ thuật thể hiện năng lực thiết kế hệ thống thực chiến.
