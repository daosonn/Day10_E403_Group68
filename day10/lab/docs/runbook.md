# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

- Agent trả lời đúng bề mặt nhưng context top-k vẫn chứa chunk stale, dẫn tới rủi ro trả lời sai ở vòng hội thoại tiếp theo.
- Triệu chứng cụ thể đã tái hiện: `q_refund_window` có `hits_forbidden=yes` ở kịch bản inject (`artifacts/eval/after_inject_bad.csv`).

---

## Detection

- Expectation fail trong log run: `refund_no_stale_14d_window`, `policy_refund_single_canonical_chunk`, `no_legacy_policy_v3_marker`.
- Eval retrieval: cột `hits_forbidden` và `top1_doc_expected` trong CSV.
- Freshness monitor từ manifest: `PASS/WARN/FAIL` theo SLA 24 giờ.

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/manifest_<run_id>.json` | Xác định `run_id`, `cleaned_records`, `quarantine_records`, `latest_exported_at` |
| 2 | Mở `artifacts/quarantine/quarantine_<run_id>.csv` | Thấy rõ `reason` nào tăng đột biến (vd `duplicate_chunk_text_after_fix`) |
| 3 | Đọc `artifacts/logs/run_<run_id>.log` | Xác định expectation nào fail và severity halt/warn |
| 4 | Chạy `python eval_retrieval.py --out artifacts/eval/check_<run_id>.csv` | Kiểm tra `hits_forbidden` cho `q_refund_window` và `top1_doc_expected` cho `q_leave_version` |

---

## Mitigation

Quy trình khắc phục chuẩn của nhóm:

1. Nếu incident do quality (expectation halt fail): sửa rule/nguồn dữ liệu, chạy lại `python etl_pipeline.py run --run-id clean-final`.
2. Nếu đã lỡ publish bản xấu (đã dùng `--skip-validate`): rerun bản sạch để upsert/prune snapshot tốt, không cần xóa thủ công toàn bộ DB.
3. Nếu freshness FAIL kéo dài: gắn cờ data stale trong báo cáo vận hành, thông báo owner ingest, chưa thay prompt/model cho tới khi data tươi.
4. Sau khi khắc phục, luôn chạy lại eval và lưu CSV vào `artifacts/eval/` để có bằng chứng before/after.

---

## Prevention

- Duy trì tối thiểu 3 expectation halt cho stale policy: `refund_no_stale_14d_window`, `policy_refund_single_canonical_chunk`, `no_legacy_policy_v3_marker`.
- Chuẩn hóa timestamp ngay từ bước clean (`exported_at` về UTC Z) để freshness parser ổn định.
- Cố định quy trình sprint: inject có chủ đích chỉ dùng cho demo, trước khi nộp bắt buộc rerun clean.
- Mở rộng tiếp theo: thêm cảnh báo tự động (email/Slack) khi freshness FAIL hoặc `hits_forbidden=yes`.
