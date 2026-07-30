---
title: "Worklog Tuần 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.3. </b> "
---
### Mục tiêu tuần 3:

* Củng cố bảo mật: IAM policy chi tiết, mã hóa dữ liệu, quản lý secrets.
* Bắt đầu làm quen với Infrastructure as Code (IaC).
* Thực hành quản trị hệ thống mà không cần SSH trực tiếp.

### Công việc thực hiện trong tuần:
| Thứ | Công việc | Ngày | Nguồn tài liệu |
| --- | --- | --- | --- |
| 2 | Kiểm soát truy cập với **IAM Policies and Conditions**; Giới hạn quyền với **IAM Permission Boundaries** | 15/06/2026 | <https://000044.awsstudygroup.com>, <https://000030.awsstudygroup.com> |
| 3 | Mã hóa với **AWS KMS**; Quản lý credentials với **AWS Secrets Manager** | 16/06/2026 | <https://000033.awsstudygroup.com>, <https://000096.awsstudygroup.com> |
| 4 | Truy cập server từ xa với **Systems Manager Session Manager** (không cần SSH key/bastion) | 17/06/2026 | <https://000058.awsstudygroup.com> |
| 5 | Infrastructure as Code với **AWS CloudFormation**: template, stack, change set | 18/06/2026 | <https://000037.awsstudygroup.com> |
| 6 | Làm quen **AWS CDK**: định nghĩa hạ tầng bằng code | 19/06/2026 | <https://000038.awsstudygroup.com> |

### Kết quả đạt được tuần 3:

* Viết được IAM policy có điều kiện (condition) và permission boundary để áp dụng least privilege chính xác hơn.
* Mã hóa dữ liệu at-rest bằng KMS và chuyển các credential hard-code sang Secrets Manager.
* Kết nối vào EC2 instance qua Systems Manager Session Manager mà không cần mở port SSH inbound.
* Triển khai và cập nhật một stack nhỏ bằng AWS CloudFormation template.
* Viết app CDK (TypeScript) đầu tiên để dựng hạ tầng bằng code thay vì thao tác trên console.
* Hiểu rõ sự đánh đổi giữa CloudFormation (khai báo YAML/JSON) và CDK (code mệnh lệnh, sinh ra CloudFormation).
