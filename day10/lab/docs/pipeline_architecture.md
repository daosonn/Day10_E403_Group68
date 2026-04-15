# Kiến trúc pipeline — Lab Day 10

**Nhóm:** Nhóm 68 -E403  
**Cập nhật:** 2026-04-15

---

## 1. Sơ đồ luồng (bắt buộc có 1 diagram: Mermaid / ASCII)

```
data/raw/policy_export_dirty.csv
	-> ingest (load_raw_csv)
	-> transform.clean_rows
			- allowlist doc_id
			- normalize effective_date
			- normalize exported_at (UTC Z)
			- canonical refund text + dedupe x2
			- quarantine invalid rows
	-> quality.run_expectations (warn/halt)
	-> embed (Chroma upsert by chunk_id + prune stale ids)
	-> artifacts:
			- logs/run_<run_id>.log
			- manifests/manifest_<run_id>.json
			- cleaned/cleaned_<run_id>.csv
			- quarantine/quarantine_<run_id>.csv
	-> serving collection: day10_kb
```

Điểm đo freshness nằm sau bước publish (đọc từ `manifest_<run_id>.json`, trường `latest_exported_at`).

---

## 2. Ranh giới trách nhiệm

| Thành phần | Input | Output | Owner nhóm |
|------------|-------|--------|--------------|
| Ingest | `data/raw/policy_export_dirty.csv` | danh sách record raw (10 dòng) | Đào Văn Sơn |
| Transform | raw rows | cleaned CSV + quarantine CSV | Đinh Công Tài |
| Quality | cleaned rows | expectation results (halt/warn) | Đinh Công Tài |
| Embed | cleaned CSV | Chroma collection `day10_kb` | Trương Gia Ngọc |
| Monitor | manifest + logs + eval CSV | freshness status + quality evidence | Nguyễn Trọng Tín |

---

## 3. Idempotency & rerun

Pipeline dùng chiến lược idempotent theo 2 cơ chế:

- Upsert theo `chunk_id`: chạy lại cùng dữ liệu không tạo bản ghi vector trùng.
- Prune id không còn trong snapshot cleaned: khi run mới có tập chunk nhỏ hơn, hệ thống xóa id stale khỏi collection để tránh retrieval dính context cũ.

Chứng cứ: log `run_clean-final.log` có `embed_upsert count=5`; sau run inject rồi clean lại, collection vẫn đồng bộ với cleaned run cuối.

---

## 4. Liên hệ Day 09

Day 10 cung cấp tầng dữ liệu sạch/tươi cho retrieval của Day 09. Nhóm giữ cùng domain tài liệu (`policy_refund_v4`, `sla_p1_2026`, `it_helpdesk_faq`, `hr_leave_policy`) nhưng tách collection `day10_kb` để đánh giá before/after độc lập. Khi cần tích hợp lại multi-agent, chỉ cần cấu hình worker retrieval trỏ sang collection sạch của Day 10.

---

## 5. Rủi ro đã biết

- Snapshot lab có `exported_at` cũ nên freshness luôn FAIL nếu SLA để 24h.
- Chưa có alert realtime; hiện mới ghi file artifact để giám sát thủ công.
- Chưa kiểm thử bộ câu hỏi mở rộng (>4 câu); hiện tập trung vào các câu then chốt của lab.
