---
title: "Worklog Tuần 8"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.8. </b> "
---
### Mục tiêu tuần 8:

* Deploy embedding model lên AWS SageMaker Serverless Inference.
* Xử lý merge conflict với nhóm và thiết lập kiểm soát chi phí.
* Hoàn thành test, blog, và các phần còn lại của báo cáo thực tập.

### Công việc thực hiện trong tuần:
| Thứ | Công việc | Ngày | Nguồn tài liệu |
| --- | --- | --- | --- |
| 2 | Đóng gói `multilingual-e5-large` thành `model.tar.gz`; viết `inference.py` tùy chỉnh mô phỏng đúng hành vi của fastembed (prefix query/passage, mean pooling có attention-mask, chuẩn hóa L2) để tương thích với các vector đã upsert sẵn trong Qdrant | 20/07/2026 | |
| 3 | Thiết lập deploy bằng **SageMaker Python SDK** (`HuggingFaceModel` + `ServerlessInferenceConfig`, CPU) — nút "Deploy on SageMaker AI" one-click của HuggingFace chỉ nhắm tới real-time GPU endpoint, không hỗ trợ Serverless Inference | 21/07/2026 | |
| 4 | Cấu hình **AWS Budget Alert** (~$20/tháng, ngưỡng 50%/80% chi tiêu thực tế và 100% dự báo) và bật CloudWatch billing/Free Tier alert trước khi deploy tiếp | 22/07/2026 | |
| 5 | Xử lý merge conflict giữa nhánh `feat/semantic-search` của mình và nhánh `feat/refactor` của nhóm (hai file `search.py` viết độc lập trùng tên class `SearchService`); hủy merge, push code của mình, và chuyển quyết định kiến trúc cho nhóm quyết định | 23/07/2026 | |
| 6 | Test pipeline `search_service` end-to-end ở local; viết và đăng blog lên AWS Study Group; hoàn thiện Self-evaluation, Sharing & Feedback, rà soát toàn bộ báo cáo | 24/07/2026 – 30/07/2026 | [Blogs Posted](/vi/3-blogsposted/), [Self-evaluation](/vi/6-self-evaluation/), [Feedback](/vi/7-feedback/) |

### Kết quả đạt được tuần 8:

* Hoàn thành đóng gói `model.tar.gz` cho `multilingual-e5-large` và file `inference.py` tùy chỉnh tương thích.
* Rút ra cách deploy SageMaker đúng để không vượt ngân sách: dùng SDK với `ServerlessInferenceConfig` thay vì nút UI one-click (vốn chỉ hỗ trợ real-time GPU endpoint).
* Thiết lập AWS Budget Alert và CloudWatch billing/Free Tier alert để kiểm soát chi phí AWS của project.
* Xử lý đúng cách một merge conflict trên code dùng chung: hủy merge thay vì tự ý resolve, và chuyển quyết định kiến trúc hợp nhất `search.py` cho cả nhóm.
* Ghi nhận việc deploy embedding model lên SageMaker **vẫn đang bị chặn do chờ xác nhận S3 bucket** từ bạn phụ trách quản trị AWS account, vì Cloudflare R2 (nơi nhóm lưu video) không tương thích với cách SageMaker fetch artifact; endpoint cho Whisper và BLIP vẫn còn phải deploy sau đó.
* Hoàn thành test end-to-end pipeline search ở local và hoàn tất các phần còn lại của báo cáo thực tập (blog, self-evaluation, feedback).
