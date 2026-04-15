# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| `data/raw/policy_export_dirty.csv` | Batch CSV (mỗi run đọc full snapshot) | duplicate row, thiếu ngày, doc_id lạ, stale content | `raw_records`, `cleaned_records`, `quarantine_records` trong log |
| `data/docs/*.txt` (canonical) | File-based reference dùng để đối chiếu nghiệp vụ | mismatch version với export raw | expectation `refund_no_stale_14d_window`, `no_legacy_policy_v3_marker` |
| Chroma collection `day10_kb` | Publish qua upsert/prune sau validate | vector stale còn sót do không prune hoặc skip validate | `embed_upsert count`, `embed_prune_removed`, eval `hits_forbidden` |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | stable id tạo từ `doc_id + chunk_text + seq` |
| doc_id | string | Có | allowlist gồm: `policy_refund_v4`, `sla_p1_2026`, `it_helpdesk_faq`, `hr_leave_policy` |
| chunk_text | string | Có | tối thiểu 8 ký tự, không chứa stale refund/legacy marker sau clean chuẩn |
| effective_date | date | Có | chuẩn `YYYY-MM-DD`; `DD/MM/YYYY` được normalize |
| exported_at | datetime | Có | normalize về UTC ISO có hậu tố `Z` |

---

## 3. Quy tắc quarantine vs drop

Toàn bộ record fail rule được đưa vào `artifacts/quarantine/quarantine_<run_id>.csv` với cột `reason` để trace được nguyên nhân.

Nhóm quy ước:

- Quarantine: các lỗi có thể điều tra/phục hồi như `duplicate_chunk_text`, `missing_effective_date`, `stale_hr_policy_effective_date`, `unknown_doc_id`, `duplicate_chunk_text_after_fix`.
- Drop silent: không áp dụng trong lab này để tránh mất lineage.

Quy trình approve:

- Cleaning & Quality Owner xác minh rule và root cause.
- Monitoring/Docs Owner cập nhật runbook nếu xuất hiện loại lỗi mới.
- Ingestion Owner quyết định rerun publish sau khi lỗi được xử lý.

---

## 4. Phiên bản & canonical

Source of truth cho refund là `data/docs/policy_refund_v4.txt` (window chuẩn 7 ngày làm việc). Mọi chunk chứa ngữ nghĩa cũ (`14 ngày`, marker `policy-v3`) được coi là stale và phải bị chặn bởi expectation halt trước khi publish.

Với HR leave policy, chỉ chấp nhận phiên bản có `effective_date >= 2026-01-01`; bản 2025 (10 ngày phép năm) bị quarantine để tránh xung đột version trong retrieval.
