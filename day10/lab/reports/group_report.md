# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Nhóm 68 -E403  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Đào Văn Sơn | Ingestion / Raw Owner | 26ai.sondv@vinuni.edu.vn |
| Đinh Công Tài | Cleaning Rules Owner | 26ai.taidc@vinuni.edu.vn |
| Phạm Minh Quang | Quality Analytics & Eval Owner | 26ai.quangpm@vinuni.edu.vn |
| Trương Gia Ngọc | Embed & Idempotency Owner | 26ai.ngoctg@vinuni.edu.vn |
| Nguyễn Trọng Tín | Monitoring / Docs Owner | 26ai.tinnt@vinuni.edu.vn |

**Ngày nộp:** 2026-04-15  
**Repo:** VinUni-AI20k/Lecture-Day-08-09-10  

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

### Phân chia nhiệm vụ 5 người

| Thành viên | Phần việc chính | Phần việc phối hợp |
|------------|-----------------|--------------------|
| Đào Văn Sơn | Ingestion, manifest, chuẩn hóa contract đầu vào | Tổng hợp log vận hành |
| Đinh Công Tài | Thiết kế và triển khai cleaning rules | Đồng bộ rule với expectation |
| Phạm Minh Quang | Thiết kế expectation mới, chạy eval before/after, kiểm `hits_forbidden` | Hỗ trợ quality report và metric_impact |
| Trương Gia Ngọc | Embed pipeline, idempotency, prune stale vectors | Xác nhận publish snapshot cuối |
| Nguyễn Trọng Tín | Freshness monitoring, runbook, tài liệu và tổng hợp báo cáo | Điều phối evidence trong docs |

---

## 1. Pipeline tổng quan (150–200 từ)

> Nguồn raw là gì (CSV mẫu / export thật)? Chuỗi lệnh chạy end-to-end? `run_id` lấy ở đâu trong log?

**Tóm tắt luồng:**

Nhóm sử dụng nguồn raw chính là file export CSV bẩn `data/raw/policy_export_dirty.csv` gồm 10 dòng để mô phỏng lỗi dữ liệu thường gặp trước khi ingest vào vector store. Luồng chạy thực tế: ingest đọc CSV -> áp dụng cleaning rules (allowlist doc_id, chuẩn hóa ngày, lọc bản HR stale, canonical hóa policy refund, dedupe 2 pha) -> expectation suite (halt/warn) -> embed vào Chroma collection `day10_kb` -> ghi manifest, log và kiểm freshness.

Mỗi lần chạy đều có `run_id` và ghi lại các chỉ số bắt buộc (`raw_records`, `cleaned_records`, `quarantine_records`) trong log thuộc `artifacts/logs/`. Ví dụ run sạch cuối cùng `clean-final` cho kết quả `raw=10`, `cleaned=5`, `quarantine=5` và `PIPELINE_OK`.

Để đánh giá ảnh hưởng tới retrieval, nhóm chạy thêm `eval_retrieval.py` tạo CSV trong `artifacts/eval/`, so sánh giữa kịch bản inject lỗi (`inject-bad-02`) và kịch bản clean (`clean-final`). Kết quả cho thấy cùng câu hỏi refund nhưng ở kịch bản inject có `hits_forbidden=yes`, còn kịch bản sạch là `hits_forbidden=no`, chứng minh quality gate ở tầng dữ liệu có tác động trực tiếp lên kết quả truy xuất của agent.

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**

python etl_pipeline.py run --run-id clean-final; python eval_retrieval.py --out artifacts/eval/after_clean_final.csv; python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_clean-final.json

---

## 2. Cleaning & expectation (150–200 từ)

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| canonical_refund_text_14_to_7 | Run `clean-02`: `cleaned_records=6` (vẫn giữ chunk legacy đã đổi số ngày) | Run `clean-final`: `cleaned_records=5`, `quarantine_records=5` do loại thêm duplicate sau canonical | `artifacts/logs/run_clean-02.log`, `artifacts/logs/run_clean-final.log`, `artifacts/quarantine/quarantine_clean-final.csv` |
| dedupe_after_fix | Trước khi thêm rule, không có `duplicate_chunk_text_after_fix` | Run `clean-final` xuất hiện `reason=duplicate_chunk_text_after_fix` với 1 dòng | `artifacts/quarantine/quarantine_clean-final.csv` |
| normalize_exported_at_iso_z | Run `clean-02` ghi `latest_exported_at` dạng không timezone (`2026-04-10T08:00:00`) | Run `clean-final` chuẩn hóa thành `2026-04-10T08:00:00Z`, dễ parse nhất quán | `artifacts/manifests/manifest_clean-02.json`, `artifacts/manifests/manifest_clean-final.json` |
| policy_refund_single_canonical_chunk (expectation) | Không có expectation này ở baseline | `clean-final` pass (`refund_chunk_count=1`); `inject-bad-02` fail (`refund_chunk_count=2`) | `artifacts/logs/run_clean-final.log`, `artifacts/logs/run_inject-bad-02.log` |
| no_legacy_policy_v3_marker (expectation) | Không có expectation này ở baseline | `clean-final` pass (`violations=0`); `inject-bad-02` fail (`violations=1`) | `artifacts/logs/run_clean-final.log`, `artifacts/logs/run_inject-bad-02.log` |

**Rule chính (baseline + mở rộng):**

- Baseline: allowlist `doc_id`, chuẩn hóa `effective_date`, quarantine HR cũ (< 2026-01-01), bỏ dòng thiếu thông tin, dedupe theo text, fix stale refund 14 -> 7.
- Mở rộng 1: chuẩn hóa `exported_at` về ISO UTC có hậu tố `Z`, quarantine nếu sai format.
- Mở rộng 2: canonical hóa câu refund legacy về câu chuẩn policy v4 để tránh mùi dữ liệu stale còn sót nghĩa.
- Mở rộng 3: dedupe sau fix/canonicalization nhằm chặn duplicate ngữ nghĩa phát sinh sau khi transform.

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**

Run `inject-bad-02` cố ý dùng `--no-refund-fix --skip-validate` khiến các expectation `refund_no_stale_14d_window`, `policy_refund_single_canonical_chunk`, `no_legacy_policy_v3_marker` fail. Cách xử lý là chạy lại pipeline chuẩn không bỏ quality gate (`python etl_pipeline.py run --run-id clean-final`) để loại stale chunk và trả index về trạng thái sạch trước khi eval/nộp bài.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**

Nhóm dùng đúng kịch bản Sprint 3: `python etl_pipeline.py run --run-id inject-bad-02 --no-refund-fix --skip-validate`. Kịch bản này mô phỏng lỗi migration policy: hệ thống vẫn cho dữ liệu legacy đi qua (14 ngày, marker policy-v3) và vẫn embed vào collection do bỏ halt.

**Kết quả định lượng (từ CSV / bảng):**

So sánh hai file eval:

- `artifacts/eval/after_inject_bad.csv`: với `q_refund_window` có `contains_expected=yes` nhưng `hits_forbidden=yes`, nghĩa là trong top-k vẫn còn ngữ cảnh cấm (14 ngày).
- `artifacts/eval/after_clean_final.csv`: cùng câu hỏi trên có `hits_forbidden=no`.
- Câu Merit `q_leave_version` ở cả bản sạch có `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes` (doc top-1 là `hr_leave_policy`).

Điểm quan trọng là inject không làm câu trả lời nhìn bề mặt sai ngay (vẫn có chunk 7 ngày), nhưng quan sát top-k cho thấy context bị nhiễm stale. Đây là đúng tinh thần observability: phát hiện rủi ro trước khi user nhìn thấy câu trả lời sai ở production.

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

SLA được cấu hình `FRESHNESS_SLA_HOURS=24` và đo tại boundary `publish` (theo manifest). Ở run `clean-final`, freshness cho kết quả FAIL với chi tiết `latest_exported_at=2026-04-10T08:00:00Z`, `age_hours≈120.205`, vượt ngưỡng 24 giờ.

Nhóm coi đây là tín hiệu hợp lệ vì bộ dữ liệu lab là snapshot cũ có chủ đích. Trong runbook, nhóm quy ước:

- PASS: dữ liệu mới trong SLA và có thể publish bình thường.
- WARN: thiếu timestamp hoặc metadata chưa đủ để kết luận.
- FAIL: vượt SLA, cần đánh dấu stale và ưu tiên triage nguồn ingest trước khi điều chỉnh prompt/model.

Việc cảnh báo FAIL nhưng pipeline vẫn hoàn tất giúp tách bạch giữa "khả năng chạy" và "chất lượng vận hành".

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

Dữ liệu sau clean/embed có thể phục vụ trực tiếp cho retrieval trong Day 09 vì cùng domain CS + IT Helpdesk. Tuy nhiên nhóm tách collection `day10_kb` để tránh ảnh hưởng chéo tới các thử nghiệm multi-agent trước đó, đồng thời giữ được khả năng rollback theo run_id. Cách này giúp so sánh trước/sau chất lượng dữ liệu mà không làm nhiễu luồng orchestrator đã ổn định ở Day 09.

---

## 6. Rủi ro còn lại & việc chưa làm

- Chưa tích hợp grading JSONL vì hiện tại chưa có `data/grading_questions.json` trong workspace.
- Freshness mới kiểm 1 boundary theo manifest; chưa đo tách riêng ingest_done và index_visible.
- Chưa thêm cảnh báo realtime (Slack/Email); mới dừng ở log + file artifact.
