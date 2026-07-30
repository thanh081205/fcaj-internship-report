---
title: "Worklog Tuần 4"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.4. </b> "
---
### Mục tiêu tuần 4:

* Học các pattern messaging bất đồng bộ cho hệ thống decoupled.
* Thực hành backup/reliability và containerization cơ bản.
* Dựng pipeline CI/CD đầu tiên.

### Công việc thực hiện trong tuần:
| Thứ | Công việc | Ngày | Nguồn tài liệu |
| --- | --- | --- | --- |
| 2 | Hệ thống messaging với **Amazon SQS và SNS**: queue, topic, pattern fan-out | 22/06/2026 | <https://000077.awsstudygroup.com> |
| 3 | Bảo vệ dữ liệu với **AWS Backup**: backup plan, vault, retention | 23/06/2026 | <https://000013.awsstudygroup.com> |
| 4 | Containerization với **Docker**: image, container, Dockerfile cơ bản | 24/06/2026 | <https://000015.awsstudygroup.com> |
| 5 | Container Orchestration với **Amazon ECS**: task definition, service, cluster | 25/06/2026 | <https://000016.awsstudygroup.com> |
| 6 | CI/CD Pipeline với **AWS CodePipeline**: tự động build và deploy | 26/06/2026 | <https://000017.awsstudygroup.com> |

### Kết quả đạt được tuần 4:

* Dựng workflow decoupled dùng SQS để buffer và SNS để fan-out notification.
* Cấu hình AWS Backup plan với retention policy cho tài nguyên EC2/RDS.
* Viết Dockerfile, build image, và chạy thử container ở local.
* Triển khai container service trên Amazon ECS với task definition cơ bản.
* Dựng CodePipeline tự động build và deploy khi có commit mới.
* Hiểu được cách messaging, backup và CI/CD kết hợp giúp tăng độ tin cậy và tốc độ vận hành của hệ thống.
