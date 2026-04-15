# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| **Policy Export CSV** | `data/raw/policy_export_dirty.csv` | Duplicate text, stale HR version, sai định dạng ngày | `quarantine_records`, `raw_records` |
| **Canonical Docs** | `data/docs/` | Sai nội dung policy v4 (hoàn tiền), thiếu file nguồn | `cleaned_records`, `hits_forbidden` |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| `chunk_id` | string | Có | ID duy nhất được tạo từ hash (doc_id + text + seq) |
| `doc_id` | string | Có | Khóa logic tài liệu nguồn (vd: policy_refund_v4) |
| `chunk_text` | string | Có | Nội dung văn bản của chunk (độ dài tối thiểu 8 ký tự) |
| `effective_date` | date | Có | Ngày hiệu lực định dạng ISO (YYYY-MM-DD) |
| `exported_at` | datetime | Có | Thời điểm xuất dữ liệu từ hệ thống nguồn |

---

## 3. Quy tắc quarantine vs drop

- **Quarantine**: Các bản ghi không thuộc allowlist `doc_id`, sai định dạng ngày, hoặc thuộc phiên bản HR cũ. Các bản ghi này được lưu tại `artifacts/quarantine/` để hậu kiểm.
- **Drop**: Các bản ghi trùng lặp nội dung hoàn toàn sẽ bị loại bỏ (chỉ giữ bản ghi đầu tiên).

---

## 4. Phiên bản & canonical

- **Source of truth**: Chính sách hoàn tiền phiên bản 4 (`policy_refund_v4`) là nguồn chuẩn. Bất kỳ chunk nào chứa "14 ngày làm việc" sẽ bị tự động sửa thành "7 ngày làm việc" để đồng bộ với v4.
- **HR Policy**: Chỉ chấp nhận bản ghi có `effective_date` từ `2026-01-01`.
