---
title: "Worklog Tuần 6"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.6. </b> "
---
### Mục tiêu tuần 6:

* Học container orchestration trên AWS ở mức nâng cao hơn ECS cơ bản.
* Xác định phạm vi và viết Proposal cho project Workshop cá nhân: tính năng **semantic search** cho `VideoPlatformServer`, một nền tảng chia sẻ/livestream video của nhóm (mình phụ trách riêng `search_service` bằng Python/FastAPI, một bạn khác phụ trách S3/quản trị AWS account, một bạn phụ trách `api_service` bằng NestJS/TypeScript).
* Tham gia buổi chia sẻ kỹ thuật tại văn phòng AWS.

### Công việc thực hiện trong tuần:
| Thứ | Công việc | Ngày | Nguồn tài liệu |
| --- | --- | --- | --- |
| 2 | Containerization với **Amazon ECS và AWS Fargate**: container serverless | 06/07/2026 | <https://000067.awsstudygroup.com> |
| 3 | Làm quen **Amazon EKS**: cluster, node group, kubectl | 07/07/2026 | <https://000126.awsstudygroup.com> |
| 4 | Soạn Proposal cho Workshop: semantic search trên nội dung video (transcription + caption hình ảnh + embedding + vector search) | 08/07/2026 | [Proposal](/vi/2-proposal/) |
| 5 | Thiết kế kiến trúc `search_service`: hàng đợi RabbitMQ, vector database Qdrant, ranh giới service với `api_service` của nhóm | 09/07/2026 | |
| 6 | Dựng hạ tầng nền cho `search_service`: RabbitMQ, Qdrant, PostgreSQL, Redis qua Docker Compose; tạo queue riêng `video-semantic-indexing` tách biệt với queue `video-processing` (NestJS BullMQ) của nhóm | 10/07/2026 | |
| 7 (T7) | Tham gia **AWS Study Group – FCAJ Tech Sharing Session** tại văn phòng AWS (xem [Event 1](/vi/4-eventparticipated/4.2-event2/)) | 11/07/2026 | |

### Kết quả đạt được tuần 6:

* Triển khai container trên Amazon ECS với Fargate và dựng thử cluster Amazon EKS đầu tiên.
* Hoàn thành bản nháp đầu tiên của Proposal, xác định rõ phạm vi tính năng semantic search cho nền tảng video của nhóm.
* Thiết kế `search_service` như một microservice Python/FastAPI độc lập, tách biệt với `api_service` của nhóm.
* Rút ra bài học về **cô lập hàng đợi (queue isolation)**: queue RabbitMQ riêng `video-semantic-indexing` giúp tránh việc job bị chia lẫn với queue `video-processing` sẵn có của nhóm.
* Tham gia buổi chia sẻ kỹ thuật bên ngoài về chiến lược ôn thi chứng chỉ AWS, AWS Security Agent, và best practice về SLA/monitoring — xem chi tiết ở mục [Events Participated](/vi/4-eventparticipated/).
