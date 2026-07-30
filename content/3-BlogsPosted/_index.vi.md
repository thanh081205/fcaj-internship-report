---
title: "Các bài blogs đã đăng"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 3. </b> "
---

Tại đây sẽ là phần liệt kê, giới thiệu các blogs mà các bạn đã đăng trên [AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj). Ví dụ:

###  [Blog 1 - BIG BANG, ROLLING, BLUE/GREEN HAY CANARY — NÊN CHỌN CHIẾN THUẬT DEPLOY NÀO TRÊN AWS?](3.1-Blog1/)
Blog này so sánh 4 chiến thuật deploy phổ biến trên AWS — Big Bang, Rolling, Blue/Green và Canary — về cách hoạt động, ưu nhược điểm, và gợi ý nên chọn chiến thuật nào để giảm thiểu rủi ro và downtime.

###  [Blog 2 - ZERO-SECRET DEPLOYMENT VỚI SECRETS MANAGER + IAM ROLE + IMDSV2](3.2-Blog2/)
Bài viết dạng field-report về một sự cố thật: private key Firebase bị corrupt do `jq` escape newline sai cách, và quá trình refactor sang pipeline "zero-secret deployment" dùng AWS Secrets Manager, IAM Role và IMDSv2 — kèm nhắc lại vụ rò rỉ dữ liệu Capital One 2019 và cách IMDSv2 chặn đúng đường tấn công đó.

###  [Blog 3 - ...](3.3-Blog3/)
Blog này giới thiệu Amazon EKS Pod Identity vừa bổ sung tính năng session policies, cho phép bạn thu hẹp quyền IAM một cách linh hoạt và chính xác cho từng pod mà không cần tạo thêm nhiều IAM roles riêng biệt. Đây là bước tiến quan trọng giúp áp dụng nguyên tắc least privilege hiệu quả hơn trong môi trường Kubernetes quy mô lớn.