# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Nhom09-E403  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Hoàng Sơn Lâm | Ingestion / Raw Owner | hoangsonlamwk4@gmail.com |
| Lê Tuấn Đạt | Cleaning & Quality Owner | letuandat220@gmail.com |
| Nguyễn Mạnh Tú | Embed & Idempotency Owner | tunm17421@gmail.com |
| Lưu Linh Ly | Monitoring / Docs Owner | linhluuly.work@gmail.com |

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

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**

_________________

**Kết quả định lượng (từ CSV / bảng):**

_________________

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

_________________

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

_________________

---

## 6. Rủi ro còn lại & việc chưa làm

- …
