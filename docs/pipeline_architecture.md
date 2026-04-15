# Pipeline Architecture — Lab Day 10

Hệ thống xử lý dữ liệu cho RAG Knowledge Base với kiến trúc phân tầng đảm bảo tính quan sát được (Observability).

---

## 1. Sơ đồ luồng (Flow)

```text
[Raw CSV] -> (Ingest) -> [Log Raw]
                          |
                          v
[Cleaning Rules] <--- (Transform) ---> [Quarantine CSV]
                          |
                          v
[Expectations] <----- (Validation) ---> [Halt / Continue]
                          |
                          v
[Cleaned CSV] ------> (Embed - ChromaDB)
                          |
                          v
[Manifest JSON] <--- (Metadata)
                          |
                          v
(Freshness Check) ---> [PASS/WARN/FAIL]
```

---

## 2. Các tầng xử lý (Boundaries)

### Ingestion (Nhập dữ liệu)
- Đọc dữ liệu từ nguồn xuất thô (`policy_export_dirty.csv`).
- Ghi nhận `raw_records` để kiểm soát số lượng đầu vào.

### Transformation & Cleaning (Làm sạch)
- Tách biệt dữ liệu "Sạch" và dữ liệu "Cách ly" (Quarantine).
- Thực hiện chuẩn hóa ngày tháng, xóa ghi chú rác, nhất quán thuật ngữ IT.
- Cơ chế sửa lỗi tự động (Fix rule) cho các chính sách quan trọng (ví dụ: hoàn tiền 7 ngày).

### Validation (Kiểm định)
- Chặn đứng các bản ghi không đạt chuẩn (Expectation Halt).
- Kiểm tra tính nhất quán và độ dài nội dung.

### Publishing & Embedding (Phát hành)
- Upsert vào ChromaDB bằng `chunk_id` ổn định.
- Cơ chế **Prune**: Tự động xóa các vector cũ không còn tồn tại trong bản phát hành mới nhất để đảm bảo tính đồng bộ (Snapshot consistency).

---

## 3. Thành phần giám sát (Observability)

- **Log Run ID**: Mọi lượt chạy đều có ID duy nhất gắn với dấu thời gian.
- **Manifest**: Lưu trữ trạng thái cuối cùng của lần chạy, bao gồm các chỉ số quan trọng và thông tin Vector Store.
- **Freshness**: Kiểm tra độ trễ dữ liệu dựa trên tệp Manifest và SLA cấu hình.
