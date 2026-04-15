# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Nhom09-E403  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Nguyễn Văn A | Ingestion / Raw Owner | a.nv@example.com |
| Trần Thị B | Cleaning & Quality Owner | b.tt@example.com |
| Lê Văn C | Embed & Idempotency Owner | c.lv@example.com |
| Phạm Thị D | Monitoring / Docs Owner | d.pt@example.com |

**Ngày nộp:** 2026-04-15  
**Repo:** `Nhom09-E403-Day10`  

---

## 1. Pipeline tổng quan (150–200 từ)

Hệ thống sử dụng dữ liệu thô từ tệp `data/raw/policy_export_dirty.csv`, mô phỏng quá trình trích xuất dữ liệu từ các hệ thống quản lý chính sách và hỗ trợ IT. Quy trình bao gồm 4 giai đoạn chính: Ingest (đọc dữ liệu), Transform (làm sạch), Validate (kiểm định chất lượng qua bộ Expectation) và Embed (đẩy vào ChromaDB).

Mỗi lượt chạy được định danh bằng một `run_id` duy nhất (mặc định là timestamp UTC), giúp truy xuất log và artifact tương ứng trong thư mục `artifacts/`.

**Tóm tắt luồng:**
Dữ liệu thô đi qua các quy tắc làm sạch (xóa trùng, chuẩn hóa ngày, sửa lỗi chính sách), sau đó được kiểm tra bởi bộ Expectation. Nếu vi phạm các quy tắc nghiêm trọng (Halt), pipeline sẽ dừng lại để bảo vệ Vector Store khỏi dữ liệu bẩn. Cuối cùng, dữ liệu sạch được cập nhật vào ChromaDB theo cơ chế upsert lũy đẳng.

**Lệnh chạy một dòng:**
`python etl_pipeline.py run --run-id final_run`

---

## 2. Cleaning & expectation (150–200 từ)

Nhóm đã bổ sung 3 quy tắc làm sạch mới và 2 quy tắc kiểm định chất lượng mới để tăng cường độ tin cậy cho dữ liệu RAG.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| `remove_metadata_notes` | Dòng 2 chứa ghi chú rác | Ghi chú bị xóa, thay bằng dấu chấm | `cleaned_sprint2-fix.csv` |
| `normalize_it_terminology` | "Ticket P1" | "Yêu cầu hỗ trợ P1" | `cleaned_sprint2-fix.csv` |
| `max_chunk_length_500` | N/A | OK (0 long chunks) | `run_sprint2-fix.log` |
| `no_suspicious_placeholders` | N/A | OK (0 suspicious rows) | `run_sprint2-fix.log` |
| `refund_no_stale_14d_window` | 0 lỗi (khi fix) | **1 lỗi FAIL (halt)** | `run_inject-bad.log` |

**Rule chính (baseline + mở rộng):**
- Baseline: Chuẩn hóa ngày ISO, xóa trùng nội dung, lọc tài liệu không thuộc allowlist, sửa chính sách hoàn tiền 14 -> 7 ngày.
- Mở rộng: Xóa ghi chú metadata rác, nhất quán thuật ngữ IT, xóa khoảng trắng thừa.

**Ví dụ 1 lần expectation fail:**
Khi chạy với cờ `--no-refund-fix`, kiểm định `refund_no_stale_14d_window` đã báo `FAIL (halt)` do phát hiện 1 bản ghi chứa chính sách cũ. Pipeline đã in cảnh báo và yêu cầu xử lý (trừ khi dùng `--skip-validate`).

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

Chúng tôi đã thực hiện "tiêm lỗi" (corruption inject) bằng cách bỏ qua quy tắc sửa lỗi hoàn tiền để so sánh kết quả retrieval.

**Kịch bản inject:**
Chạy pipeline với `--no-refund-fix`. Kết quả là bản ghi "Yêu cầu hoàn tiền trong 14 ngày" được đẩy vào Vector Store thay vì bản ghi "7 ngày" đã được sửa.

**Kết quả định lượng:**
- **Kịch bản Xấu (`inject-bad`)**: Câu hỏi `q_refund_window` trả về kết quả sai (14 ngày). Cột `hits_forbidden` trong file `after_inject_bad.csv` báo **yes**, cho thấy RAG đã truy xuất trúng nội dung lỗi.
- **Kịch bản Tốt (`after_fix`)**: Câu hỏi tương tự trả về kết quả đúng (7 ngày). Cột `hits_forbidden` báo **no**, chứng minh dữ liệu sạch đã loại bỏ hoàn toàn các thông tin gây nhiễu.
- **Kết luận**: Pipeline làm sạch giúp cải thiện độ chính xác của câu trả lời từ Agent, tránh cung cấp thông tin chính sách cũ cho khách hàng.

---

## 4. Freshness & monitoring (100–150 từ)

Chúng tôi thiết lập SLA Freshness là **24 giờ**. Kết quả chạy trên dữ liệu mẫu báo `FAIL` vì thời gian xuất dữ liệu (`exported_at`) là từ 5 ngày trước. Điều này giúp đội vận hành biết rằng dữ liệu trong Knowledge Base hiện tại không còn phản ánh đúng trạng thái thực tế nhất của hệ thống nguồn và cần được cập nhật ngay lập tức.

---

## 5. Liên hệ Day 09 (50–100 từ)

Dữ liệu sau khi làm sạch được lưu vào collection `day10_kb`. Collection này hoàn toàn có thể được tích hợp vào Agent hỗ trợ khách hàng từ Day 09 bằng cách thay đổi cấu hình nguồn dữ liệu. Việc làm sạch ở tầng dữ liệu (Day 10) giúp các Agent ở tầng trên (Day 09) hoạt động ổn định hơn mà không cần phải tự xử lý các ngoại lệ về định dạng hay lỗi phiên bản.

---

## 6. Rủi ro còn lại & việc chưa làm

- Chưa có cơ chế tự động cảnh báo qua Slack/Email (mới chỉ ghi log `alert_channel`).
- Các quy tắc làm sạch text mới chỉ dừng ở mức dùng Regex đơn giản, có thể gặp khó khăn với các cấu trúc văn bản phức tạp hơn.
- Cần mở rộng bộ đánh giá retrieval (Golden Dataset) lên nhiều câu hỏi hơn để bao phủ hết các trường hợp biên.
