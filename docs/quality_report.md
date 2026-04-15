# Quality report — Lab Day 10 (nhóm)

**run_id:** `sprint2-fix` (Tốt) vs `inject-bad` (Xấu)  
**Ngày:** 2026-04-15

---

## 1. Tóm tắt số liệu

| Chỉ số | Xấu (`inject-bad`) | Tốt (`sprint2-fix`) | Ghi chú |
|--------|-------------------|-------------------|---------|
| raw_records | 20 | 20 | Tổng số bản ghi thô |
| cleaned_records | 11 | 11 | Bản ghi đạt chuẩn |
| quarantine_records | 9 | 9 | Bản ghi bị loại |
| Expectation halt? | **YES** (refund_no_stale_14d) | **NO** | Pipeline dừng nếu không skip |

---

## 2. Before / after retrieval (bắt buộc)

> Dẫn link tới: [after_inject_bad.csv](file:///c:/Users/TuNM17421/Desktop/AI_Vin/Lab_DAY_10_Group/lab/artifacts/eval/after_inject_bad.csv) (Xấu) và [after_fix_eval.csv](file:///c:/Users/TuNM17421/Desktop/AI_Vin/Lab_DAY_10_Group/lab/artifacts/eval/after_fix_eval.csv) (Tốt).

**Câu hỏi then chốt:** refund window (`q_refund_window`)  
**Trước (Xấu):** `top1_preview`: "Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc...", `hits_forbidden`: **yes**.  
**Sau (Tốt):** `top1_preview`: "Hoàn tiền chỉ áp dụng cho đơn hàng có giá trị trên 100.000 VND...", `hits_forbidden`: **no**.

**Merit (khuyến nghị):** versioning HR — `q_leave_version`  
**Trước (Xấu):** `contains_expected`: yes, `hits_forbidden`: no, `top1_doc_expected`: yes.  
**Sau (Tốt):** `contains_expected`: yes, `hits_forbidden`: no, `top1_doc_expected`: yes.  
*(Ghi chú: Cả hai run đều trả về dữ liệu 12 ngày phép của năm 2026 do bản cũ 10 ngày đã bị filter ở bước quarantine).*

---

## 3. Freshness & monitor

- **Kết quả:** `FAIL` (age_hours: ~121h).  
- **Giải thích:** Dữ liệu mẫu `policy_export_dirty.csv` có `exported_at` là `2026-04-10`, trong khi thời điểm chạy là `2026-04-15`. SLA 24 giờ bị vi phạm, hệ thống cảnh báo dữ liệu cũ chính xác.

---

## 4. Corruption inject (Sprint 3)

- **Mô tả:** Sử dụng cờ `--no-refund-fix` và `--skip-validate` để ép hệ thống nhận dữ liệu lỗi 14 ngày.
- **Cách phát hiện:** 
    1. Kiểm định `refund_no_stale_14d_window` trả về `FAIL` (mức độ `halt`).
    2. Đánh giá retrieval phát hiện `hits_forbidden=yes`, chứng minh dữ liệu "độc hại" đã lọt vào vector store nếu không có pipeline ngăn chặn.

---

## 5. Hạn chế & việc chưa làm

- Cần bổ sung rule kiểm tra định dạng email hoặc số điện thoại trong các tài liệu IT Helpdesk.
- Cần tự động hóa việc đồng bộ `data_contract.yaml` với bộ gõ `expectations.py`.
