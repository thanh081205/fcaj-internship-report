---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Triển khai hệ thống lên AWS ECS Fargate

#### Tổng quan

Hệ thống server gồm hai service độc lập: `api_service` (NestJS + Apollo GraphQL) đảm nhận xác thực, quản lý video và transcode bất đồng bộ; `search_service` (Python FastAPI) đảm nhận tìm kiếm lai, kết hợp điểm từ khóa với độ tương đồng vector ngữ nghĩa thông qua Qdrant. Hai service giao tiếp với nhau qua gRPC song hướng và đồng bộ metadata bất đồng bộ qua RabbitMQ.

Trong workshop này, chúng ta sẽ triển khai toàn bộ hệ thống đó lên AWS. Đây không phải một demo đơn service, mà đi qua đúng những bài toán gặp trong hệ thống production thật: chạy container trên **ECS Fargate**, kết nối chúng tới các dịch vụ dữ liệu được quản lý bên trong VPC riêng tư, xử lý job bất đồng bộ và giao tiếp liên service, đưa nền tảng ra Internet an toàn qua CDN, và tự động hóa việc deploy bằng CI/CD.

Một quyết định thiết kế xuyên suốt workshop là **toàn bộ compute chạy trong private subnet mà không dùng NAT Gateway**. Thay vào đó, ECS task truy cập các dịch vụ AWS thông qua **VPC Endpoints**, giúp lưu lượng không đi ra Internet công cộng.

#### Những gì chúng ta sẽ xây dựng

+ Một **VPC** với public/private subnet, security group, và VPC Endpoint, CloudWatch Logs, Systems Manager, Amazon MQ.
+ Tầng dữ liệu được quản lý: **Amazon RDS for PostgreSQL**, **Amazon ElastiCache for Valkey**, **Amazon S3** và **Amazon EFS**.
+ Ba service trên **Amazon ECS Fargate**: `api-service`, `search-service`, và **Qdrant** vector database tự host với dữ liệu lưu bền vững trên EFS.
+ Đường truy cập công khai qua **Application Load Balancer** và **Amazon CloudFront**, kèm domain riêng (Route 53) và chứng chỉ TLS (AWS Certificate Manager).
+ Một **pipeline CI/CD** tự động dùng GitHub Actions với IAM OIDC federated role, không lưu bất kỳ AWS access key tĩnh nào.

#### Nội dung

1. [Giới thiệu](5.1-Workshop-overview/)
2. [Chuẩn bị](5.2-Prerequisites/)
3. [Xây dựng nền tảng mạng](5.3-Network/)
4. [Tầng dữ liệu](5.4-Data-layer/)
5. [Triển khai service lên ECS Fargate](5.5-ECS-deployment/)
6. [Đưa hệ thống ra Internet với ALB và CloudFront](5.6-Public-access/)
7. [CI/CD với GitHub Actions](5.7-CICD/)
8. [Dọn dẹp tài nguyên](5.8-Cleanup/)
