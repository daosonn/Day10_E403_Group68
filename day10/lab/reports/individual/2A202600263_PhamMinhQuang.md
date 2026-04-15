# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Phạm Minh Quang 
**Vai trò:** Quality Analytics & Eval Owner  
**Ngày nộp:** 2026-04-15  

---

## 1. Tôi phụ trách phần nào?

Trong nhóm 5 người, tôi phụ trách phần kiểm chứng chất lượng đầu ra bằng expectation và retrieval evaluation. Mục tiêu của phần việc này là trả lời được câu hỏi: pipeline đã clean xong thì dữ liệu thật sự an toàn để publish chưa, và khi publish thì retrieval có tốt hơn trước không.

Phạm vi tôi làm gồm ba mảng chính. Mảng đầu là expectation analytics: theo dõi expectation pass/fail theo từng run_id, tách rõ lỗi halt và warn để tránh publish bản dữ liệu có rủi ro nghiệp vụ. Mảng thứ hai là đánh giá before/after bằng các file CSV trong thư mục `artifacts/eval/`, tập trung vào cột `hits_forbidden`, `contains_expected` và `top1_doc_expected`. Mảng thứ ba là tổng hợp bằng chứng định lượng vào phần metric_impact và quality report để phục vụ chấm điểm.

Tôi phối hợp trực tiếp với bạn phụ trách cleaning rules để xác định rule mới nào cần expectation đi kèm, và phối hợp với bạn phụ trách embed để đảm bảo eval được chạy trên đúng snapshot publish cuối cùng.

---

## 2. Một quyết định kỹ thuật

Quyết định kỹ thuật chính của tôi là ưu tiên chỉ số `hits_forbidden` trên toàn top-k thay vì chỉ nhìn top-1 preview.

Lý do là ở bài toán refund policy, top-1 có thể đã đúng (7 ngày) nhưng top-k vẫn còn chunk stale (14 ngày). Nếu chỉ kiểm top-1 thì pipeline trông có vẻ ổn, nhưng thực tế context đưa cho agent vẫn có nguy cơ gây sai lệch ở vòng hội thoại tiếp theo.

Vì vậy tôi đặt nguyên tắc đánh giá như sau:

1. `contains_expected` phải đúng để đảm bảo thông tin cần thiết xuất hiện.
2. `hits_forbidden` phải bằng no để đảm bảo không còn context cấm.
3. Với câu versioning (`q_leave_version`), thêm `top1_doc_expected=yes` để kiểm chứng doc ưu tiên đúng nguồn.

Nguyên tắc này giúp phần quality analytics phản ánh đúng rủi ro vận hành, không chỉ phản ánh tính đúng bề mặt của câu trả lời.

---

## 3. Một lỗi hoặc anomaly đã xử lý

Anomaly tôi xử lý là trạng thái "đúng một phần" trong kịch bản inject. Khi chạy `inject-bad-02` với `--no-refund-fix --skip-validate`, kết quả eval cho `q_refund_window` ghi `contains_expected=yes` nhưng `hits_forbidden=yes`.

Nếu chỉ đọc một cột `contains_expected`, nhóm có thể kết luận sai rằng chất lượng vẫn đạt. Tôi đã đánh dấu đây là failure nghiêm trọng vì context stale vẫn tồn tại trong top-k.

Cách xử lý là yêu cầu rerun pipeline clean chuẩn (`clean-final`) rồi đánh giá lại cùng tập câu hỏi. Sau rerun, cùng câu `q_refund_window` chuyển về `hits_forbidden=no`, xác nhận quality gate hoạt động đúng.

Ngoài ra, tôi đối chiếu log expectation để liên kết triệu chứng với nguyên nhân: ở run inject có fail các expectation halt liên quan stale policy, còn run clean thì pass toàn bộ.

---

## 4. Bằng chứng trước / sau

Bằng chứng chính tôi dùng:

1. `artifacts/eval/after_inject_bad.csv`
- `q_refund_window`: `contains_expected=yes`, `hits_forbidden=yes`.

2. `artifacts/eval/after_clean_final.csv`
- `q_refund_window`: `contains_expected=yes`, `hits_forbidden=no`.
- `q_leave_version`: `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes`.

3. `artifacts/logs/run_inject-bad-02.log` và `artifacts/logs/run_clean-final.log`
- Run inject có expectation halt fail.
- Run clean pass toàn bộ expectation.

Bộ bằng chứng này chứng minh tác động quality theo cả hai lớp: expectation trong pipeline và hành vi retrieval sau publish.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ bổ sung một bảng tổng hợp tự động theo run_id để so sánh nhanh tất cả chỉ số quality (`raw_records`, `cleaned_records`, `quarantine_records`, số expectation fail, số dòng `hits_forbidden=yes`) trong một file duy nhất. Cải tiến này giúp phát hiện hồi quy nhanh hơn khi nhóm chỉnh rule hoặc thay đổi nguồn dữ liệu.
