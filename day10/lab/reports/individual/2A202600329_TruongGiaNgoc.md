# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Trương Gia Ngọc  
**Vai trò:** Embed & Idempotency Owner  
**Ngày nộp:** 2026-04-15  

---

## 1. Tôi phụ trách phần nào?

Trong bài lab Day 10, tôi phụ trách phần publish dữ liệu vào vector store và đảm bảo idempotency khi rerun. Phạm vi chính của tôi nằm ở đoạn embed trong `etl_pipeline.py`, gồm:
- Kết nối ChromaDB.
- Upsert dữ liệu theo `chunk_id`.
- Prune các vector id không còn xuất hiện trong cleaned snapshot.

Tôi chịu trách nhiệm bảo đảm rằng khi pipeline chạy lại nhiều lần (clean -> inject -> clean), collection không bị phình không kiểm soát và không giữ lại chunk stale từ run trước.

Ngoài code, tôi cũng phụ trách kiểm chứng bằng số liệu:
- `embed_upsert count` trong log.
- `embed_prune_removed` để chứng minh đã xóa id stale.
- Đối chiếu với `cleaned_records` trong manifest/log để xác nhận collection đồng bộ với snapshot sạch.

---

## 2. Một quyết định kỹ thuật

Quyết định kỹ thuật quan trọng nhất của tôi là dùng chiến lược "snapshot publish" thay vì chỉ "append/upsert đơn thuần".

Nếu chỉ upsert, các chunk cũ không còn trong cleaned run mới vẫn có thể tồn tại trong collection và đi vào top-k retrieval. Điều này đặc biệt nguy hiểm với case stale policy vì câu trả lời có thể đúng ở top-1 nhưng vẫn bị nhiễm context cấm trong top-k.

Vì vậy tôi chọn kết hợp:
1. Upsert theo `chunk_id` để đảm bảo idempotent khi dữ liệu không đổi.
2. Prune `prev_ids - current_ids` để xóa phần dư.

Lợi ích:
- Rerun an toàn, không duplicate vector.
- Collection luôn phản ánh đúng dữ liệu cleaned của run mới nhất.
- Kịch bản inject có thể rollback nhanh chỉ bằng rerun clean, không cần xóa toàn bộ DB.

---

## 3. Một lỗi hoặc anomaly đã xử lý

Anomaly tôi tập trung xử lý là stale vector tồn dư sau các lần chạy khác chế độ (inject rồi clean lại).

Triệu chứng:
- Sau run inject, collection có thêm chunk legacy refund.
- Nếu không prune đúng cách, chunk này có thể còn sót và tiếp tục ảnh hưởng retrieval ở run sau.

Trong quá trình test, log cho thấy:
- `run_inject-bad-02`: `embed_prune_removed=4`, `embed_upsert count=6`
- `run_clean-final`: `embed_upsert count=5`

Số liệu này xác nhận pipeline đã thực hiện đồng bộ snapshot thay vì giữ nguyên toàn bộ dữ liệu cũ. Khi kết hợp với eval CSV, kết quả sau run sạch có `hits_forbidden=no` cho `q_refund_window`, tức là stale context đã được xử lý triệt để.

Root cause của anomaly là dữ liệu inject cố ý cho qua `--skip-validate`, nên collection tạm thời chứa chunk không đạt chuẩn. Cách khắc phục đúng là rerun clean chuẩn với quality gate và publish lại snapshot.

---

## 4. Bằng chứng trước / sau

Bằng chứng kỹ thuật tôi sử dụng:

1. Log embed:
- `artifacts/logs/run_inject-bad-02.log`: có `embed_prune_removed=4`, `embed_upsert count=6`
- `artifacts/logs/run_clean-final.log`: có `embed_upsert count=5`, khớp `cleaned_records=5`

2. Eval retrieval:
- Trước (inject): `artifacts/eval/after_inject_bad.csv` -> `q_refund_window` có `hits_forbidden=yes`
- Sau (clean): `artifacts/eval/after_clean_final.csv` -> `q_refund_window` có `hits_forbidden=no`

3. Manifest:
- `manifest_clean-final.json` xác nhận run publish cuối cùng là `clean-final`, collection `day10_kb`.

Những bằng chứng này cho thấy phần embed/idempotency không chỉ chạy được mà còn hỗ trợ rollback dữ liệu xấu theo đúng mục tiêu observability.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi muốn thêm một bước verify post-embed tự động: đếm số vector thực trong collection và so sánh ngay với `cleaned_records`; nếu lệch sẽ raise halt. Việc này sẽ biến kiểm chứng idempotency thành guardrail tự động, tránh phụ thuộc vào kiểm tra log thủ công.
