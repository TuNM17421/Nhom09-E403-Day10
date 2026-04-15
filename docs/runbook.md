# Runbook — Lab Day 10 (incident tối giản)

Tài liệu vận hành hệ thống Data Pipeline cho RAG.

---

## Symptom

- **Người dùng/Agent**: Thấy Agent trả lời thông tin cũ (ví dụ: "14 ngày hoàn tiền" thay vì "7 ngày").
- **Hệ thống**: Pipeline bị dừng đột ngột (Halt) hoặc có cảnh báo Freshness thất bại.

---

## Detection

- **Freshness Alert**: Kiểm tra kết quả `freshness_check` trong tệp Manifest. `FAIL` nếu dữ liệu cũ hơn 24 giờ.
- **Expectation Fail**: Log của `etl_pipeline.py` báo cáo lỗi cấp độ `halt` (ví dụ: `refund_no_stale_14d_window FAIL`).
- **Eval Alert**: `hits_forbidden=yes` xuất hiện trong kết quả chạy `eval_retrieval.py`.

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | `latest_exported_at` phải nhỏ hơn 24 giờ so với hiện tại. |
| 2 | Mở `artifacts/quarantine/*.csv` | Xem lý do tại sao các bản ghi quan trọng bị loại bỏ (sai định dạng ngày, doc_id lạ). |
| 3 | Chạy `python eval_retrieval.py` | Kiểm tra xem `top1_preview` có chứa thông tin gây lỗi không. |
| 4 | Kiểm tra `contracts/data_contract.yaml` | Xác nhận `allowed_doc_ids` có thiếu tài liệu mới nào không. |

---

## Mitigation

- **Lỗi Freshness**: Cập nhật lại thời gian xuất dữ liệu từ nguồn chuẩn (`exported_at`) và chạy lại pipeline.
- **Lỗi Expectation**: 
    1. Nếu lỗi do dữ liệu thô quá tệ: Yêu cầu nguồn cấp dữ liệu chuẩn hóa lại.
    2. Nếu lỗi do quy tắc kiểm định quá khắt khe: Điều chỉnh `expectations.py` hoặc hạ mức độ xuống `warn`.
- **Rerun Pipeline**: Chạy `python etl_pipeline.py run --run-id recovery` để cập nhật lại Vector Store. Hệ thống tự động xóa bản ghi cũ (Prune).

---

## Prevention

- **Thêm Expectation**: Bổ sung kiểm tra định dạng cho các cột dữ liệu quan trọng để chặn lỗi sớm hơn.
- **Cảnh báo sớm**: Thiết lập `alert_channel` để nhận thông báo ngay khi Freshness bị `FAIL`.
- **Đồng bộ Contract**: Luôn cập nhật `data_contract.yaml` khi có tài liệu hoặc phiên bản chính sách mới.
