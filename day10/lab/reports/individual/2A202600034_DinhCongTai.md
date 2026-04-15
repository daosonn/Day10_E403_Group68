# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Đinh Công Tài  
**Vai trò:** Cleaning & Quality Owner  
**Ngày nộp:** 2026-04-15  

---

## 1. Tôi phụ trách phần nào?

Tôi phụ trách tầng cleaning và quality gate của pipeline Day 10. Công việc chính gồm thiết kế/triển khai rule làm sạch trong `transform/cleaning_rules.py` và expectation suite trong `quality/expectations.py` để quyết định dữ liệu nào được phép publish vào collection `day10_kb`.

Phạm vi tôi chịu trách nhiệm không chỉ là viết rule, mà còn phải chứng minh rule đó có tác động đo được theo yêu cầu chống trivial của rubric. Vì vậy tôi làm song song cả 3 việc: cập nhật rule, chạy run kiểm chứng (`clean-02`, `inject-bad-02`, `clean-final`), và đối chiếu artifact `quarantine_*.csv`, `run_*.log`, `after_*.csv`.

Các phần tôi phối hợp với đồng đội:
- Với Ingestion Owner: thống nhất format cột đầu vào và điểm fail hợp lệ.
- Với Embed Owner: đảm bảo dữ liệu sau clean phù hợp chiến lược upsert/prune.
- Với Monitoring Owner: chuyển kết quả expectation fail thành nội dung runbook dễ vận hành.

---

## 2. Một quyết định kỹ thuật

Quyết định quan trọng nhất tôi đưa ra là tách rõ expectation nào là `halt` và expectation nào là `warn`, theo triết lý "dữ liệu có thể không hoàn hảo, nhưng dữ liệu gây sai nghiệp vụ phải chặn publish".

Cụ thể:
- `halt`: các lỗi làm sai chính sách hoặc sai version (ví dụ `refund_no_stale_14d_window`, `policy_refund_single_canonical_chunk`, `no_legacy_policy_v3_marker`, `effective_date_iso_yyyy_mm_dd`).
- `warn`: các lỗi chất lượng phụ, không phá nghiệp vụ cốt lõi ngay lập tức (ví dụ `chunk_min_length_8`).

Tôi cũng bổ sung 3 rule mới có tác động thực:
1. Chuẩn hóa `exported_at` sang UTC ISO Z.
2. Canonical hóa chunk refund legacy về đúng câu policy v4.
3. Dedupe vòng 2 sau khi canonicalization để loại duplicate ngữ nghĩa.

Quyết định này giúp pipeline tránh 2 thái cực xấu: quá lỏng (để dữ liệu stale lọt) và quá cứng (halt vô lý cho lỗi nhẹ).

---

## 3. Một lỗi hoặc anomaly đã xử lý

Anomaly tiêu biểu tôi xử lý là "stale chunk không biến mất hoàn toàn dù đã thay 14 -> 7".

Triệu chứng khi inject:
- `run_inject-bad-02.log` báo fail đồng thời 3 expectation halt:
  - `refund_no_stale_14d_window`
  - `policy_refund_single_canonical_chunk`
  - `no_legacy_policy_v3_marker`
- `after_inject_bad.csv` cho `q_refund_window` có `hits_forbidden=yes`.

Root cause:
- Kịch bản `--no-refund-fix --skip-validate` cố tình cho dữ liệu cũ đi qua quality gate.
- Nếu không có expectation bổ sung, pipeline dễ "đậu giả" vì top-1 vẫn có chunk đúng, trong khi top-k còn chunk cấm.

Cách xử lý:
1. Bổ sung expectation mới để bắt đúng failure mode.
2. Chạy lại pipeline sạch `clean-final` không skip validate.
3. Kiểm tra lại eval để xác nhận `hits_forbidden` về `no`.

Kết quả:
- `run_clean-final.log` pass toàn bộ expectation.
- `after_clean_final.csv` xác nhận retrieval sạch với `q_refund_window`.

---

## 4. Bằng chứng trước / sau

Bằng chứng định lượng tôi dùng:

1. Đếm record trước/sau sửa rule:
- `clean-02`: `cleaned_records=6`, `quarantine_records=4`
- `clean-final`: `cleaned_records=5`, `quarantine_records=5`

Số liệu này cho thấy rule mới đã loại thêm 1 record duplicate sau canonicalization (`duplicate_chunk_text_after_fix`).

2. Bằng chứng retrieval:
- Trước (inject): `q_refund_window` -> `hits_forbidden=yes`
- Sau (clean-final): `q_refund_window` -> `hits_forbidden=no`

3. Merit line:
- `q_leave_version` giữ `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes`.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi muốn chuyển expectation sang framework có report chuẩn (ví dụ Great Expectations hoặc pydantic validation layer) để tự động xuất summary fail/pass theo nhóm rule và lưu baseline quality theo từng run_id. Việc này giúp reviewer đối chiếu nhanh hơn và giảm lỗi khi so sánh thủ công giữa nhiều file log.
