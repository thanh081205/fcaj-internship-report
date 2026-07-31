---
title: "Event 2"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.2. </b> "
---

# Bài thu hoạch: "AWS Study Group – FCAJ Tech Sharing Session"

**Thời gian:** 11/07/2026

**Địa điểm:** Tầng 26, Bitexco Tower, 02 Hải Triều, Phường Sài Gòn, TP. Hồ Chí Minh

**Vai trò:** Người tham dự

### Tổng quan sự kiện

Đây là buổi chia sẻ kỹ thuật được tổ chức tại văn phòng AWS trong khuôn khổ chương trình First Cloud AI Journey (FCAJ), gồm 3 bài trình bày liên tiếp từ các diễn giả cộng đồng về chứng chỉ, tự động hóa bảo mật và giám sát hệ thống.

### Bài nói 1 — "Inside The Exam: AWS Cloud Practitioner" (Diễn giả: Ngô Lê Tấn Huy)

Lộ trình chiến lược để chinh phục kỳ thi AWS Certified Cloud Practitioner (CLF-C02):

- **Cấu trúc đề thi:** 65 câu trắc nghiệm (đơn/đa đáp án), 90 phút (thí sinh không phải người bản ngữ tiếng Anh được cộng 30 phút), điểm đậu 700/1000, hiệu lực chứng chỉ 3 năm.
- **Trọng số các domain:** Cloud Concepts (24%), Security and Compliance (30%), Cloud Technology and Services (34%), Billing/Pricing/Support (12%).
- **Chiến lược ôn tập:** tư duy "map keyword" (gắn service với các từ khóa use-case thực tế), review lại câu sai thay vì chỉ làm đề thử, và thực hành trực tiếp trên AWS Free Tier.
- **Mẹo khi đi thi:** kỹ thuật loại trừ đáp án, không nghĩ quá phức tạp với các câu ở mức cơ bản, chú ý các từ khóa gây nhiễu ("not", "least cost", "most scalable"), và chuẩn bị giấy tờ tùy thân khi thi tại trung tâm Pearson VUE.

### Bài nói 2 — "Securing Your Web Apps With AWS Security Agent" (Diễn giả: Nguyễn Tuấn Thịnh)

Giới thiệu AWS Security Agent — công cụ kiểm thử bảo mật tự động, được xây dựng trên nền Amazon Bedrock:

- **Vấn đề cần giải quyết:** pentest thủ công tốn thời gian (nhiều tuần), chi phí cao ($5.000–$20.000 mỗi lần thuê ngoài), và kết quả không đồng đều tùy vào năng lực người test.
- **Khả năng:** bao phủ toàn bộ vòng đời bảo mật — Design Review (đối chiếu tài liệu kiến trúc với các chuẩn PCI DSS/NIST CSF/AWS Well-Architected), Code Review (tự quét Pull Request trên GitHub/GitLab, đề xuất fix), và Automated Penetration Testing (chuỗi khai thác nhiều bước, ví dụ IDOR → XSS, kèm bằng chứng khai thác có thể kiểm chứng).
- **Chi phí:** tính theo giờ sử dụng agent (khoảng $50/giờ), có gói dùng thử miễn phí 400 giờ trong 2 tháng; một case study thực tế tốn $1.500–$2.500 cho phần việc của agent, so với chi phí thuê đội pentest truyền thống.
- **Giới hạn:** bị chặn bởi các lớp xác thực mạnh (MFA/sinh trắc học/mTLS), khó phát hiện lỗi logic nghiệp vụ, và cần giám sát chặt để kiểm soát số giờ agent tiêu tốn với các ứng dụng phức tạp.

### Bài nói 3 — "SLA and Monitoring: From SLA to Monitoring, What Really Matters" (Diễn giả: Nguyễn Huỳnh Sơn)

Bài nói lý giải vì sao chỉ số hạ tầng "khỏe mạnh" không đồng nghĩa với trải nghiệm người dùng tốt, minh họa bằng demo trực tiếp:

- **Thông điệp chính:** *Healthy Infrastructure ≠ Healthy User Experience.* SLA của AWS chỉ cam kết cho hạ tầng cloud; trải nghiệm khách hàng là trách nhiệm của đội ngũ vận hành hệ thống.
- **Monitoring pyramid:** Cloud Provider → Infrastructure → Application → Business → Customer Experience. Các tầng dưới giúp chẩn đoán root cause, các tầng trên cho biết user/business có thực sự bị ảnh hưởng hay không.
- **Demo trực tiếp:** một ứng dụng 3-tier (User → ALB → EC2 → RDS) vẫn hiển thị dashboard "toàn xanh" (CPU 18%, ALB target healthy, `/health` trả về 200 OK) ngay cả sau khi diễn giả chủ động chặn security group giữa EC2 và RDS — vì health check không chạm tới database, trong khi luồng `/login` thật của user thì có. Tỷ lệ login thành công tụt từ 100% xuống 0% mà không có cảnh báo hạ tầng nào được kích hoạt.
- **Bài học về alerting:** cần có custom metric nghiệp vụ (ví dụ tỷ lệ login thất bại) đưa vào CloudWatch Alarm → SNS → Email/Slack, để đội ngũ biết trước khi khách hàng phàn nàn.

### Bài học rút ra

- Việc ôn thi chứng chỉ hiệu quả hơn khi tư duy theo mẫu (gắn từ khóa với service) thay vì học thuộc lòng.
- Kiểm thử bảo mật đang dần chuyển sang các AI agent có khả năng tự động thực hiện chuỗi khai thác và xác minh lỗ hổng, thay đổi bài toán đánh đổi giữa chi phí và tốc độ so với pentest truyền thống.
- Giám sát ở tầng hạ tầng (CPU, memory, health check) là chưa đủ — cần có chỉ số nghiệp vụ/hành trình người dùng (như tỷ lệ login hay checkout thành công) để biết khi nào user thực sự bị ảnh hưởng.

### Hình ảnh sự kiện

![Khán giả tại văn phòng AWS trong buổi chia sẻ](/fcaj-internship-report/images/4-EventParticipated/4.2-Event2/audience-overview.jpg)
*Khán phòng kín chỗ tại văn phòng AWS cho buổi tech sharing của FCAJ*

![Trò chơi trắc nghiệm chứng chỉ AWS trực tiếp trên màn hình](/fcaj-internship-report/images/4-EventParticipated/4.2-Event2/certification-quiz.jpg)
*Mini game trắc nghiệm chứng chỉ AWS xen giữa các phần trình bày*

![Trò chơi bảng điểm đội tại sự kiện](/fcaj-internship-report/images/4-EventParticipated/4.2-Event2/team-scoreboard-game.jpg)
*Hoạt động thi đua theo đội trong buổi chia sẻ*
