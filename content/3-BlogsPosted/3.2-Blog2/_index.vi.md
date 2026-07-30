---
title: "Blog 2"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

# ZERO-SECRET DEPLOYMENT VỚI SECRETS MANAGER + IAM ROLE + IMDSV2 — BÀI HỌC 4 TIẾNG DEBUG VÌ MỘT KÝ TỰ XUỐNG DÒNG

Tháng đầu xây CI/CD, mình vẫn copy file `.env` lên server qua `scp` — sai lầm nghiêm trọng nhất kỳ thực tập. Vỡ lẽ sau 4 tiếng debug vì `FIREBASE_PRIVATE_KEY` bị corrupt do `jq` không escape newline đúng cách. Từ đó mình refactor để đạt **"zero-secret deployment"**: secret chỉ sống trong AWS Secrets Manager, EC2 lấy về qua IAM Role + IMDSv2 mỗi lần deploy, biến mất ngay sau khi container start.

## Quy tắc vàng

* Secret không nằm trong Dockerfile (`COPY`).
* Không commit lên git.
* Không lưu file `.env` cố định trên EC2.
* Sau khi container start, `rm -f` file `.env` tạm ngay lập tức.

## IAM Role vs access key

* **IAM User** có long-term credential — rủi ro cao nếu lộ.
* **IAM Role** cấp temporary credential qua STS, thời hạn 1 giờ, tự xoay vòng.
* **Setup:** tạo role `EC2-Backend-Role`, attach policy `SecretsManagerReadWrite`, gắn vào EC2 — instance tự lấy credential qua IMDSv2, không cần hardcode access key.

## Con bug newline

Private key thật chứa ký tự newline (`0x0A`), nhưng lưu vào JSON secret phải dùng escape `\n`. `jq` xem `\n` là text thường thay vì escape sequence → Firebase báo lỗi `"Invalid PEM formatted key"`.

Fix bằng Python heredoc (`python3 - <<'PYEOF'` + `json.loads()`) thay vì `jq`, vì Python hiểu đúng JSON spec.

## Nhắc lại vụ Capital One 2019

106 triệu record bị lộ vì kết hợp SSRF trên WAF + IMDSv1 chấp nhận GET đơn giản để lấy STS credential. IMDSv2 chặn bằng session-oriented protocol — bắt buộc PUT với header token trước khi GET, SSRF thường không gửi được PUT.

## Enforce IMDSv2

```
aws ec2 modify-instance-metadata-options --instance-id i-xxx \
  --http-tokens required --http-put-response-hop-limit 3 --http-endpoint enabled
```

Có thể enforce toàn account bằng SCP chặn `ec2:RunInstances` nếu thiếu `HttpTokens=required`.

## Rotation & audit

* Rotate tự động qua Lambda mỗi 90 ngày (`aws secretsmanager rotate-secret ...`).
* CloudTrail ghi lại mọi lần `GetSecretValue`.
* CloudWatch Alarm cảnh báo nếu gọi >10 lần/giờ từ cùng 1 instance.

## Bài học rút ra

Khi xử lý structured data (JSON/YAML/TOML), luôn dùng ngôn ngữ có parser đầy đủ (`json.loads`, `yaml.safe_load`), đừng dùng `jq`/`sed`/`awk`/`grep`. Và từ vụ Capital One: phòng thủ nhiều lớp là bắt buộc — bỏ qua IMDSv2 + IAM Role đúng cách, một lỗ hổng SSRF nhỏ cũng có thể thành thảm hoạ.

🔗 **Bài viết gốc trên AWS Study Group:** [Xem trên Facebook](https://www.facebook.com/groups/awsstudygroupfcj/posts/2226205278144432)
