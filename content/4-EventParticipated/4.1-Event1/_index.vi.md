---
title: "Event 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.1. </b> "
---

# Bài thu hoạch: "First Cloud Journey (FCJ) Community Day"

**Thời gian:** 11/07/2026

**Địa điểm:** Tầng 26, Bitexco Tower, 02 Hải Triều, Phường Sài Gòn, TP. Hồ Chí Minh

**Vai trò:** Người tham dự

### Tổng quan sự kiện

Community Day là sự kiện chia sẻ đa chủ đề tại văn phòng AWS, thuộc cộng đồng First Cloud Journey (FCJ), với 6 bài trình bày liên tiếp từ các thành viên và cựu thành viên FCJ, xoay quanh bảo mật, nền tảng DevOps, phát triển sự nghiệp, networking trong game, làm việc nhóm, và generative AI.

### Bài nói 1 — "WAF + ML for Cyber Attack Detection" (Diễn giả: Lê Hoàng Gia Đại)

Xây dựng hệ thống phát hiện xâm nhập mạng (NIDS) dựa trên Machine Learning để bổ trợ cho AWS WAF:

- **Vì sao chỉ dùng WAF là chưa đủ:** cơ chế rule-based của AWS WAF xử lý tốt các mẫu tấn công đã biết, nhưng gặp khó với tấn công zero-day, hybrid/spoofing, và các hành vi bất thường chưa từng thấy.
- **Cách tiếp cận NIDS:** huấn luyện model trên bộ dữ liệu CSE-CIC-IDS2018 (UNB/CIC), bao gồm các nhãn tấn công như DoS, brute force, SQL injection, DDoS; dùng LightGBM, đánh giá bằng confusion matrix, và xử lý mất cân bằng lớp dữ liệu để cải thiện phát hiện các loại tấn công thiểu số.
- **Kiến trúc AWS:** kết hợp VPC, EC2, ALB, WAF, S3, Kinesis Data Firehose, Lambda, Security Hub, GuardDuty, Inspector, SNS, IAM, Config và CloudWatch, đối chiếu dự đoán của NIDS với sự kiện từ WAF trên một dashboard thời gian thực.
- **Bài học rút ra:** chất lượng dữ liệu quyết định hiệu năng ML; chỉ dựa vào signature-based là chưa đủ; kết hợp ML-based NIDS với AWS WAF tạo ra lớp phòng thủ thích ứng, nhiều tầng.

### Bài nói 2 — "Docker – A Containerization Technology" (Diễn giả: Bao Huynh)

Giới thiệu containerization dễ tiếp cận cho người mới:

- **Virtualization vs. containerization:** mỗi VM mang theo cả một hệ điều hành (nặng, cập nhật chậm); container chỉ đóng gói ứng dụng cùng những gì nó cần, nhẹ và nhất quán trên mọi máy.
- **Kiến thức nền Docker:** image, container, Dockerfile, và cơ chế cache theo layer (layer không đổi được tái sử dụng; layer thay đổi sẽ khiến layer đó và các layer sau phải build lại).
- **Use case:** pipeline CI/CD, kiến trúc microservices, môi trường dev/test, ứng dụng cloud-native, và hiện đại hóa ứng dụng cũ.

### Bài nói 3 — "From IT Helpdesk to Senior Sysadmin" (Diễn giả: Tran Trung Vinh)

Chia sẻ hành trình sự nghiệp dành cho sinh viên và nhân sự IT junior:

- **Con đường đi:** bắt đầu từ IT Helpdesk không có lợi thế đặc biệt, xây dựng kỹ năng Linux/networking, lab thực hành, và troubleshooting dưới áp lực trước khi chuyển hẳn sang vai trò Sysadmin.
- **Cuộc sống của một Sysadmin:** provisioning server, quản lý mạng, vá lỗi bảo mật, capacity planning — với nguyên tắc cốt lõi "không bao giờ test trên production".
- **Chuyển đổi sang Cloud/DevOps:** từ on-premise, scale thủ công sang tư duy cloud (AWS, elastic scaling, managed service), Infrastructure as Code (Terraform), và văn hóa DevOps (CI/CD, Docker).
- **Lời khuyên sự nghiệp:** đào sâu 1-2 kỹ năng cốt lõi trước khi mở rộng, xây dựng portfolio thực chiến (quan trọng hơn chỉ có chứng chỉ), và kiên trì bất kể điểm xuất phát.

### Bài nói 4 — "Multiplayer in the Cloud: Connecting Godot Clients with AWS WebSockets" (Diễn giả: Nguyen Quoc Bao)

Xây dựng multiplayer thời gian thực bằng dịch vụ serverless của AWS kết hợp game engine Godot:

- **Chọn kiến trúc:** so sánh UDP/ENet (độ trễ thấp nhất, phù hợp FPS/đua xe), WebSocket (full-duplex, tin cậy, phù hợp game theo lượt/lobby/chat), và HTTP Polling (đơn giản nhưng độ trễ cao) — WebSocket được chọn cho game theo lượt.
- **Kiến trúc AWS:** API Gateway WebSocket API (route `$connect`/`$disconnect`/`$default`) → Lambda (Node.js) → DynamoDB (dùng `connectionId` làm partition key, lưu trạng thái trận đấu) → CloudWatch để log.
- **Tích hợp Godot client:** dùng `WebSocketPeer` để kết nối, poll mỗi frame, gửi/nhận message JSON để điều khiển matchmaking và trạng thái game.
- **Thách thức gặp phải:** kết nối "chết" (stale) gây lỗi `GoneException`, chi phí scan toàn bảng DynamoDB cho matchmaking, và bản chất stateless của Lambda buộc mọi trạng thái game phải đi qua lại DynamoDB.
- **Hướng tiếp theo:** AWS GameLift cho các game cần đồng bộ liên tục, tần suất cao (so với WebSocket + Lambda phù hợp hơn cho game theo lượt/lobby).

### Bài nói 5 — "The Art of Effective Teamwork" (Diễn giả: Truong Huy Phuoc)

Bài chia sẻ kỹ năng mềm về làm việc nhóm hiệu quả:

- **4 quy tắc vàng:** mục tiêu rõ ràng và được chia sẻ chung, đúng người đúng việc, giao tiếp cởi mở & lắng nghe chủ động, và trách nhiệm cá nhân.
- **Công cụ số hỗ trợ teamwork:** ClickUp, Trello, Slack, Google Workspace, và Discord để phối hợp và giao tiếp.

### Bài nói 6 — "GraphRAG: Xây dựng ứng dụng GraphRAG với Amazon Bedrock và Amazon Neptune" (Diễn giả: Viet Phat)

Vượt qua giới hạn của RAG thông thường với các câu hỏi cần suy luận nhiều bước (multi-hop):

- **Giới hạn của RAG:** pipeline RAG thông thường gặp khó với câu hỏi cần nối chuỗi thông tin qua nhiều thực thể/tài liệu (ví dụ: "Trụ sở công ty được mua lại bởi công ty do Jeff Bezos sáng lập nằm ở đâu?").
- **Giải pháp của GraphRAG:** lưu quan hệ tường minh dưới dạng cạnh (edge) trong đồ thị và dùng graph traversal để suy luận multi-hop thay vì chỉ dựa vào độ tương đồng vector.
- **Hai hướng triển khai trên AWS:**
  - *Fully managed:* Amazon Bedrock Knowledge Bases (chunking, trích xuất entity, sinh embedding) + Amazon Neptune Analytics (lưu trữ đồ thị, khám phá quan hệ).
  - *Custom:* pipeline LlamaIndex để chuẩn bị dữ liệu/xây dựng knowledge graph + Amazon Neptune để lưu trữ, truy vấn multi-hop, và Cypher query.

### Bài học rút ra

- Bảo mật nhiều lớp (WAF rule-based + phát hiện bất thường bằng ML) bắt được các mối đe dọa mà từng lớp riêng lẻ có thể bỏ sót.
- Containerization giải quyết vấn đề "chạy được trên máy tôi" mà virtualization dựa trên VM truyền thống gặp khó khăn.
- Một portfolio dự án thực chiến mạnh quan trọng cho sự nghiệp hơn chỉ có chứng chỉ — và "không bao giờ test trên production" là nguyên tắc vận hành phổ quát.
- Chọn đúng kiến trúc networking (UDP, WebSocket, hay HTTP polling) phụ thuộc hoàn toàn vào yêu cầu thời gian thực của use case, không có một lựa chọn "tốt nhất" chung cho mọi trường hợp.
- Làm việc nhóm hiệu quả phụ thuộc vào mục tiêu chung, đúng người đúng việc, và trách nhiệm cá nhân — không chỉ là công cụ phối hợp.
- GraphRAG mở rộng RAG thông thường bằng cách mô hình hóa quan hệ tường minh, cho phép suy luận multi-hop mà vector retrieval thuần túy không hỗ trợ được.

### Hình ảnh sự kiện

![Slide về Network Intrusion Detection System (NIDS) trong bài nói WAF + ML](/fcaj-internship-report/images/4-EventParticipated/4.1-Event1/nids-talk.jpg)
*Slide kiến trúc NIDS từ bài nói "WAF + ML for Cyber Attack Detection"*

![Slide so sánh kiến trúc UDP/ENet, WebSocket và HTTP Polling](/fcaj-internship-report/images/4-EventParticipated/4.1-Event1/godot-websocket-talk.jpg)
*Chọn kiến trúc networking cho multiplayer, từ bài nói Godot + AWS WebSocket*

![Slide disclaimer mở đầu bài nói về Docker với giọng điệu thân thiện](/fcaj-internship-report/images/4-EventParticipated/4.1-Event1/docker-talk.jpg)
*Mở đầu bài nói về containerization với Docker*
