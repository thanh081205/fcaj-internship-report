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

###  [Blog 3 - AMAZON BEDROCK MANAGED KNOWLEDGE BASE — RAG "MANAGED" CHO ENTERPRISE, KHÔNG CẦN TỰ DỰNG PIPELINE](3.3-Blog3/)
Tóm tắt tính năng Amazon Bedrock Managed Knowledge Base vừa được AWS công bố: native data connectors, Smart Parsing và Agentic Retriever giúp biến cả pipeline RAG (connector, parsing, embedding, retrieval, re-ranking) thành một primitive được quản lý sẵn, cùng khả năng tích hợp với AgentCore Gateway và không bị khóa vào một model cố định.