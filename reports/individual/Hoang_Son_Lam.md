# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Hoàng Sơn Lâm  
**Vai trò:** Ingestion / Raw Owner  
**Ngày nộp:** 2026-04-15  

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Trong dự án Day 10, tôi đảm nhận vai trò **Ingestion / Raw Owner**, chịu trách nhiệm chính cho Sprint 1: thiết kế bộ dữ liệu thô, xây dựng data contract, và tạo manifest đầu tiên.

**File / module:**

- `data/raw/policy_export_dirty.csv` — thiết kế bộ raw với các lỗi đa dạng (duplicate, stale date, unknown doc_id, missing fields).
- `docs/data_contract.md` — source map, schema cleaned 5 cột, quy tắc quarantine vs drop.
- `artifacts/manifests/manifest_sprint1.json` — manifest đầu tiên ghi nhận kết quả pipeline.

**Kết nối với thành viên khác:**

Phối hợp với **Cleaning Owner** (Lê Tuấn Đạt) để raw data bao phủ đủ failure mode, và cung cấp schema cho **Embed Owner** (Nguyễn Mạnh Tú) thiết kế upsert theo `chunk_id`.

**Bằng chứng:** Commit `70e1e0f`, PR #1.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Quyết định quan trọng nhất tôi thực hiện trong Sprint 1 là thiết kế quy tắc **quarantine vs drop** trong data contract (`docs/data_contract.md` phần 3).

Cụ thể, bản ghi có `doc_id` không thuộc allowlist, sai định dạng ngày, hoặc thuộc phiên bản HR cũ (trước 2026-01-01) được đưa vào **quarantine** — lưu tại `artifacts/quarantine/` để hậu kiểm — thay vì xóa hoàn toàn. Lý do: dữ liệu bị loại có thể chứa thông tin hữu ích hoặc bị phân loại nhầm; quarantine cho phép kiểm tra lại và khôi phục nếu cần.

Ngược lại, bản ghi **trùng lặp nội dung hoàn toàn** thì **drop** (giữ bản đầu tiên). Duplicate gây nhiễu vector store nếu embed, nên loại bỏ là an toàn.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Triệu chứng:** Khi chạy pipeline Sprint 1 lần đầu với bộ raw CSV ban đầu (chỉ 10 dòng), `quarantine_records` chỉ đạt khoảng 4–5. Số lượng này quá ít để minh chứng cho tất cả các cleaning rule và failure mode mà pipeline cần xử lý (unknown doc_id, missing date, invalid format, stale HR, empty text).

**Metric phát hiện:** So sánh `quarantine_records` trong manifest với danh sách failure mode trong data contract cho thấy nhiều trường hợp chưa được bao phủ — pipeline thiếu dữ liệu test cho các nhánh xử lý quan trọng.

**Xử lý:** Bổ sung 10 dòng vào `policy_export_dirty.csv`, mỗi dòng đại diện một failure mode: dòng 12 thiếu `exported_at`, dòng 16 ngày sai format ("Feb 2026"), dòng 18 HR cũ (`2025-06-15`), dòng 19 `doc_id` lạ, dòng 20 `chunk_text` rỗng. Kết quả: manifest sprint1 ghi `raw_records=20`, `cleaned_records=9`, `quarantine_records=11`.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Tôi sử dụng manifest Sprint 1 (`run_id: sprint1`) để minh chứng kết quả:

- **Trước (raw CSV ban đầu):** Data contract còn placeholder (`…`), raw chỉ 10 dòng, pipeline chạy nhưng quarantine ít, không đủ bằng chứng cho failure mode.
- **Sau (run_id: `sprint1`):**
  - `raw_records=20`, `cleaned_records=9`, `quarantine_records=11`
  - `latest_exported_at=2026-04-10T08:00:00`
  - Data contract hoàn chỉnh: source map 2 nguồn, schema 5 cột bắt buộc, quarantine rules rõ ràng

Manifest: `artifacts/manifests/manifest_sprint1.json`. Diff commit `70e1e0f` cho thấy data contract chuyển từ placeholder sang nội dung đầy đủ.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ thêm bước **schema validation tự động** bằng thư viện `pandera` ngay tại tầng ingest, trước khi dữ liệu đi vào cleaning rules. Điều này cho phép phát hiện lỗi cấu trúc (thiếu cột, sai kiểu dữ liệu, giá trị null bất thường) ngay khi đọc CSV, thay vì đợi đến bước expectation mới bắt được — giảm thời gian debug và tăng tính tin cậy của pipeline.
