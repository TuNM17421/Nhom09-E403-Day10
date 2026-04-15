# Quality report — Lab Day 10 (nhóm)

**run_id:** `sprint2-fix` (Tốt) vs `inject-bad` (Xấu)  
**Ngày:** 2026-04-15

---

## 1. Tóm tắt số liệu

| Chỉ số | Xấu (`inject-bad`) | Tốt (`sprint2-fix`) | Ghi chú |
|--------|-------------------|-------------------|---------|
| raw_records | 20 | 20 | Tổng số bản ghi thô |
| cleaned_records | 9 |  | Bản ghi đạt chuẩn |
| quarantine_records | 11 |  | Bản ghi bị loại |
| Expectation halt? | **YES** (refund_no_stale_14d) | **NO** | Pipeline dừng nếu không skip |

---

## 2. Before / after retrieval (bắt buộc)

> Dẫn link tới: `artifacts/eval/after_inject_bad.csv` (Trước) và `artifacts/eval/after_fix_eval.csv` (Sau).

**Câu hỏi then chốt:** refund window (`q_refund_window`)  
**Trước (Xấu):** `top1_preview`: "Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc...", `hits_forbidden`: **yes**.  
**Sau (Tốt):** `top1_preview`: "Yêu cầu hoàn tiền được chấp nhận trong vòng 7 ngày làm việc...", `hits_forbidden`: **no**.

**Merit (khuyến nghị):** versioning HR — `q_leave_version`  
**Trước (Xấu):** `contains_expected`: yes, `hits_forbidden`: no, `top1_doc_expected`: yes.  
**Sau (Tốt):** `contains_expected`: yes, `hits_forbidden`: no, `top1_doc_expected`: yes.  
*(Ghi chú: Lỗi HR 2025 đã được lọc bởi baseline quarantine nên cả hai run đều cho kết quả tốt cho câu này).*

---

## 3. Freshness & monitor

- **Kết quả:** `FAIL` (age_hours: ~120h).  
- **Giải thích:** Dữ liệu mẫu có `exported_at` từ ngày 2026-04-10, trong khi SLA là 24 giờ. Hệ thống phát hiện chính xác dữ liệu đã cũ (stale data).

---

## 4. Corruption inject (Sprint 3)

- **Mô tả:** Cố ý sử dụng cờ `--no-refund-fix` để giữ lại thông tin chính sách hoàn tiền 14 ngày cũ trong tài liệu `policy_refund_v4`.
- **Cách phát hiện:** 
    1. Kiểm định `refund_no_stale_14d_window` trả về `FAIL` (mức độ `halt`).
    2. Đánh giá retrieval phát hiện `hits_forbidden=yes`, cảnh báo nội dung cấm đang tồn tại trong vector store.

---

## 5. Hạn chế & việc chưa làm

- Chưa tích hợp kiểm tra tự động định dạng `exported_at` ở mức expectation.
- Cần thêm các quy tắc làm sạch cho các ký tự đặc biệt trong tên riêng nếu có.
