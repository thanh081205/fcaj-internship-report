---
title: "Chia sẻ, đóng góp ý kiến"
date: 2024-01-01
weight: 7
chapter: false
pre: " <b> 7. </b> "
---

### Đánh giá chung

**1. Môi trường làm việc**
Không gian văn phòng AWS (tầng 26, Bitexco Tower) thoải mái và phù hợp để tập trung làm việc, đồng thời cũng là nơi diễn ra các sự kiện cộng đồng như Community Day, giúp kỳ thực tập cảm giác gắn kết với cộng đồng builder rộng hơn chứ không chỉ là ngồi làm việc một mình.

**2. Sự hỗ trợ của mentor / team admin**
Mentor cho mình không gian để tự chủ trọn vẹn tính năng semantic search — kể cả để mình tự xử lý các quyết định kỹ thuật và một merge conflict với nhánh của đồng đội — nhưng vẫn sẵn sàng hỗ trợ khi mình bị kẹt (ví dụ khi bàn về hướng deploy SageMaker Serverless Inference). Team admin xử lý việc đăng ký văn phòng và hậu cần sự kiện rất suôn sẻ.

**3. Sự phù hợp giữa công việc và chuyên ngành học**
Việc xây dựng pipeline `search_service` (transcription, embedding, vector search, benchmark) gắn liền trực tiếp với chuyên ngành AI của mình, còn phần triển khai trên AWS (SageMaker, IAM, budget, CloudWatch) đẩy mình sang mảng hạ tầng và MLOps mà trước đây ở trường mình chưa tiếp cận nhiều.

**4. Cơ hội học hỏi & phát triển kỹ năng**
Ngoài pipeline kỹ thuật, mình học được cách benchmark các chiến lược retrieval một cách nghiêm túc (Precision@3, Recall@3, MRR) thay vì tin theo cảm tính, cách truyền đạt kết quả cho nhóm qua báo cáo viết tay, và cách xử lý một dependency chéo nhóm thực tế (chờ bucket S3 từ đồng đội để deploy) mà không để nó làm chậm tiến độ của bản thân.

**5. Văn hóa & tinh thần đồng đội**
Cộng đồng FCAJ mang tính hợp tác hơn là cạnh tranh — các sự kiện như buổi chia sẻ Community Day tạo sân chơi để các thành viên FCJ dạy lẫn nhau, và việc các khóa trước sẵn lòng viết lại roadmap học tập của FCJ giúp việc bắt nhịp dễ dàng hơn nhiều.

**6. Chính sách / phúc lợi cho thực tập sinh**
Việc đăng ký làm việc tại văn phòng AWS cho các sự kiện linh hoạt, quy trình đăng ký/duyệt rõ ràng (qua portal FCAJ) giúp mình dễ sắp xếp giữa việc học và thực tập.

---

### Một số câu hỏi khác
- **Khoảnh khắc hài lòng nhất:** khi số liệu benchmark dense vs. hybrid xác nhận một quyết định kỹ thuật thật sự có ý nghĩa (chọn dense-only thay vì hybrid RRF), thay vì chỉ ship theo cấu hình mặc định.
- **Điều có thể cải thiện:** hướng dẫn rõ ràng hơn ngay từ đầu về quyền sở hữu hạ tầng dùng chung (ai phụ trách S3 bucket, giới hạn account như concurrency của SageMaker Serverless) để các điểm nghẽn liên nhóm được phát hiện sớm hơn.
- **Có giới thiệu cho bạn bè không:** có — sự kết hợp giữa một project kỹ thuật mở thực sự, mentor hỗ trợ tốt, và một cộng đồng năng động (sự kiện, blog, study group) là điều khó tìm thấy trong một kỳ thực tập ngắn.

---

### Đề xuất & mong muốn
- Một checklist ngắn, dùng chung cho cả nhóm về quyền sở hữu tài nguyên/account AWS (ai phụ trách S3, budget, IAM) sẽ giúp tránh những điểm nghẽn như mình gặp phải khi deploy SageMaker.
- Mình muốn tiếp tục đóng góp cho AWS Study Group ngay cả sau khi chương trình kết thúc.
- Nhìn chung, kỳ thực tập này giúp mình hình dung rõ hơn nhiều về việc đưa một ý tưởng ML từ notebook thành một service AWS được deploy, benchmark và kiểm soát chi phí đầy đủ — và mình rất muốn tiếp tục theo hướng này.
