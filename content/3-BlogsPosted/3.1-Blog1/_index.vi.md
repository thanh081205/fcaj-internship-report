---
title: "Blog 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# BIG BANG, ROLLING, BLUE/GREEN HAY CANARY — NÊN CHỌN CHIẾN THUẬT DEPLOY NÀO TRÊN AWS?

Xin chào mọi người!

Sau một thời gian làm việc với AWS, mình học được rằng có 4 chiến thuật deploy thường được sử dụng: **Big Bang**, **Rolling**, **Blue/Green** và **Canary**. Việc chọn đúng chiến thuật phù hợp với dự án là điều cần thiết để giảm thiểu rủi ro, tối thiểu downtime và duy trì niềm tin của user.

## 1. Big Bang

* **Cách hoạt động:** dừng hệ thống cũ → deploy toàn bộ → bật phiên bản mới.
* **Ưu điểm:** đơn giản, không phải duy trì 2 phiên bản song song, chi phí thấp vì chỉ cần 1 môi trường.
* **Nhược điểm:** có downtime; nếu xảy ra lỗi thì toàn bộ user đều bị ảnh hưởng và cần chuẩn bị sẵn bước rollback.

## 2. Rolling

* **Cách hoạt động:** update một nhóm nhỏ instance/pod, chờ health check xanh, rồi mới đi tiếp nhóm sau. Cứ thế update dần dần toàn bộ hệ thống.
* **Ưu điểm:** gần như không có downtime; nếu xảy ra lỗi thì chỉ ảnh hưởng một phần nhỏ.
* **Nhược điểm:** rollout lâu, và trong lúc rollout thì 2 version chạy song song → dễ gây ra vấn đề tương thích (API cũ/mới, format message trong queue, schema DB…).

## 3. Blue/Green

* **Cách hoạt động:** duy trì 2 môi trường giống hệt nhau. Blue đang phục vụ user, bạn deploy version mới lên Green, test thoải mái, ưng thì chuyển traffic sang Green. Blue giữ lại làm backup.
* **Ưu điểm:** gần như zero downtime, rollback chỉ là chuyển traffic ngược lại, tính bằng giây.
* **Nhược điểm:** tốn chi phí gấp đôi cho hạ tầng trong lúc chạy song song, và phải đảm bảo dữ liệu + config của 2 bên tương đương nhau.

## 4. Canary

* **Cách hoạt động:** release cho một phần rất nhỏ user hoặc hạ tầng trước (1%), theo dõi tín hiệu thật, ổn thì tăng dần 5% → 25% → 100%.
* **Ưu điểm:** phát hiện vấn đề sớm, giảm thiểu ảnh hưởng đến user, tăng tốc độ linh hoạt.
* **Nhược điểm:** đòi hỏi khả năng giám sát và điều hướng traffic đủ tốt.

## Vậy nên chọn cách nào?

| Chiến thuật | Downtime | Rollback | Chi phí | Phù hợp với |
|---|---|---|---|---|
| Big Bang | Cao | Khó | Thấp | Hệ thống đơn giản, ít release, chấp nhận được downtime |
| Rolling | Thấp | Trung bình | Thấp | Điểm khởi đầu cân bằng cho hầu hết team |
| Blue/Green | Gần như zero | Dễ nhất | Tăng tạm thời (gấp đôi) | Hệ thống mission-critical |
| Canary | Tối thiểu | Dễ | Trung bình | CI/CD liên tục, hệ thống traffic cao |

Không có chiến lược nào là tốt nhất cả. Nên chọn chiến thuật phù hợp với mức độ phức tạp của hệ thống, khả năng chịu downtime và mức độ rủi ro chấp nhận được.

🔗 **Bài viết gốc trên AWS Study Group:** [Xem trên Facebook](https://www.facebook.com/groups/awsstudygroupfcj/posts/2224001141698179)
