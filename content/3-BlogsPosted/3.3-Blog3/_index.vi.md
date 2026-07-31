---
title: "Blog 3"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---
# AMAZON BEDROCK MANAGED KNOWLEDGE BASE — RAG "MANAGED" CHO ENTERPRISE, KHÔNG CẦN TỰ DỰNG PIPELINE

Xin chào mọi người!

AWS vừa công bố (17/06/2026) một tính năng mới trong Bedrock mà mình thấy khá đáng chú ý cho ai đang làm agentic AI / RAG: **Amazon Bedrock Managed Knowledge Base**. Mình tóm tắt lại những điểm chính để chia sẻ với mọi người.

## 1. Vấn đề mà nó giải quyết

Khi tự xây knowledge base cho agent, dev thường gặp 3 khó khăn:

* Kết nối dữ liệu doanh nghiệp: dữ liệu nằm rải rác ở nhiều hệ thống (SharePoint, Confluence, Drive…), mỗi nơi có access control và định dạng khác nhau → phải tự viết connector riêng.
* Tối ưu độ chính xác của RAG: phải liên tục thử nghiệm parsing strategy, chunking, embedding model… mới ra được câu trả lời đúng và đủ ngữ cảnh.
* Vận hành hạ tầng ở quy mô lớn: hàng triệu tài liệu hoặc hàng nghìn knowledge base nhỏ đều cần hạ tầng ổn định, bảo mật, kiểm soát chi phí.

## 2. Ba tính năng chính

* **Native data connectors**: 6 connector dựng sẵn (Amazon S3, SharePoint, Confluence, Web Crawler, Google Drive, OneDrive), tự động kéo dữ liệu và cả quyền truy cập từ các ứng dụng SaaS.
* **Smart Parsing**: tự động chọn chiến lược parsing phù hợp cho từng loại dữ liệu — giữ cấu trúc HTML/bảng/ảnh khi crawl web, giữ hierarchy tài liệu khi lấy từ SharePoint, tự nhận diện bounding box để trích xuất và caption ảnh/video.
* **Agentic Retriever**: xử lý câu hỏi phức tạp cần suy luận nhiều bước (multi-hop) — tự lập kế hoạch truy vấn từng bước, tra cứu qua nhiều knowledge base rồi tổng hợp câu trả lời.

## 3. Tích hợp với AgentCore Gateway

Managed Knowledge Base có sẵn như một pre-built target type trong Amazon Bedrock AgentCore Gateway, tự động expose qua chuẩn MCP nên các framework như Strands Agents, LangChain, CrewAI, LlamaIndex, LangGraph dùng được ngay, không cần code tích hợp riêng.

## 4. Không bị khóa vào một model cố định

Managed Knowledge Base tách riêng hạ tầng (connector, parsing, storage, retrieval) khỏi phần chọn model, nên bạn luôn dùng được model mới nhất, chọn model rẻ/nhanh cho câu hỏi đơn giản và model mạnh cho câu hỏi phức tạp trên cùng hạ tầng. Nếu đã dùng Bedrock Knowledge Bases API cũ thì migrate không cần đổi code.

## 5. Giá & khu vực

Tính phí theo dung lượng dữ liệu index và số lượt retrieval (on-demand), không cam kết trả trước. Hiện có tại US East (N. Virginia), US West (Oregon), Asia Pacific (Sydney, Tokyo), Europe (Dublin, Frankfurt, London), AWS GovCloud (US-West).

## Vậy nên nhớ gì?

Amazon Bedrock Managed Knowledge Base biến cả pipeline RAG (connector + parsing + embedding + retrieval + re-ranking) thành một primitive được quản lý sẵn, giúp dev tập trung vào logic nghiệp vụ thay vì tự dựng hạ tầng.

Nguồn: [aws.amazon.com/blogs/aws/introducing-amazon-bedrock-managed-knowledge-base](https://aws.amazon.com/blogs/aws/introducing-amazon-bedrock-managed-knowledge-base-for-faster-more-accurate-enterprise-ai-applications/)

🔗 **Bài viết gốc trên AWS Study Group:** [Xem trên Facebook](https://www.facebook.com/groups/awsstudygroupfcj/posts/2226890134742613)