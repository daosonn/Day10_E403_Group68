# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Trọng Tín  
**Vai trò:** Monitoring / Docs Owner  
**Ngày nộp:** 2026-04-15  

---

## 1. Tôi phụ trách phần nào?

Tôi phụ trách monitoring và tài liệu vận hành cho Day 10. Vai trò của tôi là chuyển kết quả kỹ thuật thành quy trình quan sát được và xử lý được khi xảy ra incident. Cụ thể, tôi làm việc với các file:
- `monitoring/freshness_check.py`
- `docs/runbook.md`
- `docs/pipeline_architecture.md`
- `docs/data_contract.md`
- `docs/quality_report.md`

Tôi là người tổng hợp các chỉ số quan trọng từ artifact và biến chúng thành câu chuyện vận hành nhất quán. Ví dụ, với mỗi run tôi đối chiếu log, manifest, eval CSV để trả lời 3 câu:
1. Pipeline có chạy xong không?
2. Dữ liệu có đủ chất lượng để publish không?
3. Index publish xong có làm retrieval tốt hơn không?

Việc này giúp nhóm tránh sai lầm phổ biến là nhìn `PIPELINE_OK` rồi kết luận hệ thống ổn, trong khi freshness có thể FAIL hoặc top-k vẫn dính forbidden context.

---

## 2. Một quyết định kỹ thuật

Quyết định kỹ thuật cốt lõi tôi đưa ra là đo freshness tại boundary `publish` (manifest sau embed), thay vì chỉ đo ở lúc bắt đầu ingest.

Lý do:
- Nếu chỉ đo ingest timestamp, có thể xảy ra trường hợp raw mới nhưng publish chậm/không thành công, user vẫn thấy dữ liệu cũ.
- Đo ở publish giúp metric gắn trực tiếp với trạng thái mà agent thật sự nhìn thấy.

Trong repo hiện tại, kết quả `freshness_check` đều FAIL do dữ liệu mẫu cũ (`latest_exported_at=2026-04-10T08:00:00Z` và `sla_hours=24`). Tôi chủ động giữ FAIL trong báo cáo thay vì "làm đẹp" dữ liệu để PASS, vì mục tiêu observability là phản ánh đúng thực trạng.

Tôi cũng thống nhất cách diễn giải trong runbook:
- PASS: trong SLA.
- WARN: thiếu timestamp/metadata.
- FAIL: vượt SLA, cần triage nguồn trước khi tinh chỉnh model/prompt.

---

## 3. Một lỗi hoặc anomaly đã xử lý

Anomaly mà tôi tập trung xử lý là mâu thuẫn giữa cảm nhận thủ công và chỉ số có hệ thống.

Triệu chứng:
- Ở kịch bản inject, người kiểm thử có thể vẫn thấy câu trả lời refund "đúng" do top-1 có chunk 7 ngày.
- Tuy nhiên eval CSV báo `hits_forbidden=yes`, nghĩa là top-k vẫn nhiễm chunk stale (14 ngày).

Nếu không có monitor theo cột `hits_forbidden`, incident này rất dễ bị bỏ sót và chỉ lộ ra khi user phản hồi.

Cách tôi xử lý:
1. Bắt buộc đưa `hits_forbidden` thành chỉ số chính trong quality report.
2. Cập nhật runbook để bước diagnosis luôn đọc eval CSV thay vì chỉ nhìn top-1 preview.
3. Liên kết symptom -> detection -> diagnosis -> mitigation thành quy trình 1 chiều, ai cũng có thể làm theo.

Kết quả là nhóm có thể chứng minh rõ before/after thay vì mô tả định tính.

---

## 4. Bằng chứng trước / sau

Bằng chứng tôi sử dụng cho báo cáo:

1. Từ eval:
- `artifacts/eval/after_inject_bad.csv`: `q_refund_window` có `hits_forbidden=yes`
- `artifacts/eval/after_clean_final.csv`: cùng câu hỏi chuyển thành `hits_forbidden=no`

2. Từ log:
- `run_inject-bad-02.log`: 3 expectation halt fail
- `run_clean-final.log`: toàn bộ expectation pass

3. Từ manifest:
- `manifest_clean-final.json`: freshness FAIL với age_hours ~120, đúng với snapshot cũ trong lab.

Các bằng chứng này đủ để kết luận pipeline clean-final đã cải thiện chất lượng retrieval nhưng vẫn còn rủi ro vận hành về độ tươi nguồn dữ liệu.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ thêm cơ chế cảnh báo tự động khi `freshness=FAIL` hoặc `hits_forbidden=yes` (ghi ra một file alert summary hoặc gửi webhook). Nhờ đó nhóm không cần kiểm tra thủ công toàn bộ artifact sau mỗi run, giảm thời gian phát hiện incident trong thực tế vận hành.
