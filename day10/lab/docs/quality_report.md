# Quality report — Lab Day 10 (nhóm)

**run_id:** inject-bad-02 -> clean-final  
**Ngày:** 2026-04-15

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (inject-bad-02) | Sau (clean-final) | Ghi chú |
|--------|------------------------|-------------------|---------|
| raw_records | 10 | 10 | Cùng một raw snapshot |
| cleaned_records | 6 | 5 | Sau khi thêm canonical + dedupe after fix, sạch hơn |
| quarantine_records | 4 | 5 | Tăng do bắt thêm `duplicate_chunk_text_after_fix` |
| Expectation halt? | Yes (3 halt fail) | No | `clean-final` pass toàn bộ expectation |

---

## 2. Before / after retrieval (bắt buộc)

Nguồn so sánh:

- `artifacts/eval/after_inject_bad.csv`
- `artifacts/eval/after_clean_final.csv`

**Câu hỏi then chốt:** refund window (`q_refund_window`)  
**Trước:** `contains_expected=yes`, `hits_forbidden=yes`  
**Sau:** `contains_expected=yes`, `hits_forbidden=no`

Diễn giải: ở bản inject, top-k vẫn chứa context cấm (14 ngày), dù top-1 đã có chunk 7 ngày. Sau run sạch, dữ liệu stale bị loại khỏi retrieval context.

**Merit (khuyến nghị):** versioning HR — `q_leave_version`

**Trước:** `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes`  
**Sau:** `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes`

Diễn giải: rule versioning HR giữ ổn định ngay cả khi inject refund corruption.

---

## 3. Freshness & monitor

Kết quả freshness ở `clean-final`:

- status: `FAIL`
- `latest_exported_at`: `2026-04-10T08:00:00Z`
- `age_hours`: ~120.205
- `sla_hours`: 24

Đây là hành vi kỳ vọng với snapshot dữ liệu lab cũ. Nhóm giữ fail này trong báo cáo để thể hiện monitoring đúng ngữ nghĩa vận hành, không "ép PASS" bằng cách sửa timestamp giả.

---

## 4. Corruption inject (Sprint 3)

Kịch bản inject: `python etl_pipeline.py run --run-id inject-bad-02 --no-refund-fix --skip-validate`

Mục tiêu: mô phỏng pipeline vẫn publish dù quality gate fail.

Kết quả quan sát trong log `run_inject-bad-02.log`:

- `refund_no_stale_14d_window`: FAIL
- `policy_refund_single_canonical_chunk`: FAIL
- `no_legacy_policy_v3_marker`: FAIL

Kết quả này khớp với eval khi `q_refund_window` xuất hiện `hits_forbidden=yes`.

---

## 5. Hạn chế & việc chưa làm

- Chưa có grading JSONL do workspace chưa có `data/grading_questions.json`.
- Chưa tích hợp cảnh báo tự động (Slack/Email).
- Chưa mở rộng benchmark retrieval thành nhiều data slice ngoài 4 câu baseline.