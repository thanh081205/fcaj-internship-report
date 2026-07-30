---
title: "Worklog Tuần 7"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.7. </b> "
---
### Mục tiêu tuần 7:

* Xây dựng hoàn chỉnh pipeline ML của `search_service` ở local: transcription, caption dự phòng, embedding, lưu trữ vector, và hybrid search.
* Benchmark các chiến lược retrieval khác nhau để chọn phương án tốt nhất cho production.
* Sửa các lỗi chất lượng dữ liệu và tương thích API phát hiện trong quá trình test.

### Công việc thực hiện trong tuần:
| Thứ | Công việc | Ngày | Nguồn tài liệu |
| --- | --- | --- | --- |
| 2 | Triển khai **transcription** bằng Whisper (`faster-whisper`, model "small"), lọc bằng `no_speech_prob` ở cả mức video và chunk | 13/07/2026 | |
| 3 | Triển khai **BLIP** làm caption dự phòng cho video không có audio hoặc audio im lặng | 14/07/2026 | |
| 4 | Triển khai **embedding**: `multilingual-e5-large` (dense vector 1024 chiều) + BM25/IDF (sparse); lưu vào **Qdrant** (collection `video_chunks`, point ID dạng `uuid5`) với `userId`/`visibility` được truyền xuyên suốt từ RabbitMQ → Qdrant (chủ sở hữu luôn thấy nội dung của mình; người khác chỉ thấy nội dung PUBLIC) | 15/07/2026 | |
| 5 | Triển khai và benchmark 3 **phương pháp search** — dense-only, sparse-only (BM25), và hybrid RRF fusion — bằng Precision@3, Recall@3, MRR | 16/07/2026 | |
| 6 | Debug các vấn đề chất lượng dữ liệu phát hiện khi benchmark; viết báo cáo benchmark (`bao-cao-dense-sparse-hybrid.md`) cho nhóm | 17/07/2026 | |

### Kết quả đạt được tuần 7:

* Hoàn thành pipeline ML của `search_service` chạy được end-to-end ở local (ingest → transcribe/caption → embed → index → search).
* Phát hiện **dense-only cho kết quả tốt hơn hybrid** (MRR ≈ 0.900 so với ≈ 0.684): RRF fusion không trọng số bị kéo tụt khi phần sparse (BM25) thất bại với truy vấn cross-lingual, nên chọn dense-only làm phương án mặc định an toàn hơn cho use case này.
* Sửa **lỗi bất đối xứng dấu tiếng Việt (diacritics)** trong BM25: việc bỏ dấu ở query nhưng index text có dấu nguyên bản làm giảm chất lượng retrieval; khắc phục bằng cách áp dụng `normalize_transcript_text()` (giữ nguyên dấu, dùng `re.UNICODE`) đối xứng ở cả bước index và query.
* Phát hiện và xử lý breaking change trong `qdrant-client==1.18.0` (API `FusionQuery`: `function=` → `fusion=`) ảnh hưởng đến `search_points()`.
* Sửa lỗi shadowing biến do đặt tên cả model e5 và Whisper đều là `model`; đổi tên thành `e5_model`/`whisper_model` cho rõ ràng.
* Học được rằng **cosine similarity có ngưỡng nhiễu ~0.82** với `multilingual-e5` bất kể nội dung text — đây là đặc tính vốn có của model (anisotropy), không phải bug.
* Hoàn thành báo cáo benchmark cho nhóm, ghi lại so sánh dense vs. sparse vs. hybrid và khuyến nghị dùng dense-only cho production.
