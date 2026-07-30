---
title: "Worklog Tuần 5"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.5. </b> "
---
### Mục tiêu tuần 5:

* Xây dựng backend serverless hoàn chỉnh và kết nối với frontend.
* Tự động hóa deploy serverless với AWS SAM.
* Thêm authentication cho ứng dụng serverless.

### Công việc thực hiện trong tuần:
| Thứ | Công việc | Ngày | Nguồn tài liệu |
| --- | --- | --- | --- |
| 2 | Backend serverless với **Lambda, S3 và DynamoDB** | 29/06/2026 | <https://000078.awsstudygroup.com> |
| 3 | Phát triển frontend gọi Serverless API (gọi API Gateway từ frontend tĩnh) | 30/06/2026 | <https://000079.awsstudygroup.com> |
| 4 | Tự động hóa deploy với **AWS SAM**: `sam build` / `sam deploy` | 01/07/2026 | <https://000080.awsstudygroup.com> |
| 5 | Xác thực người dùng với **Amazon Cognito**: user pool, hosted UI, JWT | 02/07/2026 | <https://000081.awsstudygroup.com> |
| 6 | Xử lý sự kiện với **SQS và SNS** trong workflow serverless | 03/07/2026 | <https://000083.awsstudygroup.com> |

### Kết quả đạt được tuần 5:

* Xây dựng backend CRUD serverless với Lambda, API Gateway và DynamoDB.
* Kết nối frontend tĩnh (host trên S3) với API serverless.
* Tự động hóa chu trình build/deploy của serverless stack bằng AWS SAM thay vì thao tác thủ công trên console.
* Thêm chức năng đăng ký/đăng nhập bằng Amazon Cognito và validate JWT ở phía API.
* Tích hợp SQS/SNS vào workflow serverless để xử lý sự kiện bất đồng bộ.
* Hiểu rõ vòng đời ứng dụng serverless từ phát triển local đến deploy tự động.
