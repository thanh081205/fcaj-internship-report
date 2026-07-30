---
title: "Worklog Tuần 2"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.2. </b> "
---
### Mục tiêu tuần 2:

* Thực hành các dịch vụ storage và database cốt lõi của AWS.
* Host static website trên S3 và giám sát tài nguyên với CloudWatch.
* Thực hành quản lý DNS với Route 53 và AWS CLI.

### Công việc thực hiện trong tuần:
| Thứ | Công việc | Ngày | Nguồn tài liệu |
| --- | --- | --- | --- |
| 2 | Host static website với **Amazon S3**: bucket, object, bucket policy | 08/06/2026 | <https://000057.awsstudygroup.com> |
| 3 | Nền tảng database với **Amazon RDS**: engine, Multi-AZ, snapshot | 09/06/2026 | <https://000005.awsstudygroup.com> |
| 4 | Nền tảng NoSQL với **Amazon DynamoDB**: table, partition/sort key, read/write capacity | 10/06/2026 | <https://000060.awsstudygroup.com> |
| 5 | Giám sát với **Amazon CloudWatch**: metric, dashboard, alarm; Quản lý DNS với **Amazon Route 53** | 11/06/2026 | <https://000008.awsstudygroup.com>, <https://000010.awsstudygroup.com> |
| 6 | Thao tác dòng lệnh với **AWS CLI**: script hóa các tác vụ thường gặp trên S3, RDS, DynamoDB | 12/06/2026 | <https://000011.awsstudygroup.com> |

### Kết quả đạt được tuần 2:

* Triển khai thành công static website trên Amazon S3, hiểu khác biệt giữa bucket policy và IAM policy.
* Khởi tạo RDS instance và so sánh được với DynamoDB cho các use case truy cập dữ liệu khác nhau.
* Tạo bảng DynamoDB, thiết kế schema partition/sort key đơn giản, thực hiện các thao tác đọc/ghi cơ bản.
* Thiết lập dashboard và alarm trên CloudWatch để giám sát metric của EC2/RDS.
* Cấu hình hosted zone và DNS record trên Route 53.
* Thực hành tự động hóa các thao tác AWS thường dùng bằng CLI thay vì console.
