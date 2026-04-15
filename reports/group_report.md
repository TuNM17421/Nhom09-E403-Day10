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

## 1. Pipeline tổng quan (150-200 từ)

Nhóm sử dụng nguồn raw là tệp `data/raw/policy_export_dirty.csv`, mô phỏng một lần export từ hệ thống policy và IT helpdesk với nhiều lỗi dữ liệu: `doc_id` lạ, ngày hiệu lực không theo ISO, bản HR cũ, duplicate, bản refund stale 14 ngày, và các dòng thiếu dữ liệu. Pipeline được chạy qua `etl_pipeline.py` theo chuỗi ingest → clean → validate → embed → manifest/freshness. Trong lần chạy chuẩn của nhóm, `run_id=clean-2026-04-15`, log và artifact được ghi lần lượt vào `artifacts/logs/run_clean-2026-04-15.log`, `artifacts/cleaned/cleaned_clean-2026-04-15.csv`, `artifacts/quarantine/quarantine_clean-2026-04-15.csv`, và `artifacts/manifests/manifest_clean-2026-04-15.json`.

Luồng xử lý thực tế như sau: raw CSV được đọc vào và đếm `raw_records`; các cleaning rule sẽ chuẩn hóa ngày, bỏ các `doc_id` ngoài allowlist, loại duplicate, sửa refund window 14 → 7 ngày, và đưa các bản ghi lỗi vào quarantine. Bộ expectation sau đó quyết định có `halt` hay không trước khi publish. Nếu không có lỗi nghiêm trọng, cleaned CSV được upsert vào Chroma theo `chunk_id`, đồng thời prune các vector không còn tồn tại trong cleaned snapshot để tránh stale context. Cuối cùng manifest lưu lại `run_id`, số lượng record, đường dẫn artifact, collection Chroma, và kết quả freshness.

**Lệnh chạy một dòng:**

`python etl_pipeline.py run --run-id clean-2026-04-15`

---

## 2. Cleaning & expectation (150-200 từ)

Ngoài baseline của đề bài, nhóm bổ sung 3 cleaning rule mới và 2 expectation mới để tăng khả năng quan sát dữ liệu trước khi embed. Trong `transform/cleaning_rules.py`, rule thứ nhất xóa ghi chú metadata ở cuối câu như `(ghi chú: bản sync cũ...)`; rule thứ hai chuẩn hóa thuật ngữ IT bằng cách đổi `Ticket/ticket` thành `Yêu cầu hỗ trợ/yêu cầu hỗ trợ`; rule thứ ba nén khoảng trắng thừa để chunk text có dạng canonical trước khi sinh `chunk_id`. Trong `quality/expectations.py`, nhóm thêm `max_chunk_length_500` ở mức `warn` để phát hiện chunk quá dài, và `no_suspicious_placeholders` ở mức `halt` để chặn các nội dung kiểu `TODO`, `FIXME`, `MISSING`.

Các rule/expectation mới này không thay đổi số lượng cleaned/quarantine của run chuẩn nhưng tạo ra tác động đo được trên nội dung và log. Ví dụ, trong raw có 1 chuỗi chứa `ghi chú`, sau clean còn 0; trong raw có 3 lượt xuất hiện `Ticket/ticket`, sau clean còn 0 và được thay bằng 3 lượt `Yêu cầu hỗ trợ/yêu cầu hỗ trợ`. Khi nhóm cố ý chạy `inject-bad`, expectation `refund_no_stale_14d_window` chuyển từ `OK` sang `FAIL (halt)` với `violations=1`, chứng minh pipeline có khả năng chặn dữ liệu stale trước khi publish.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| `remove_metadata_notes` | `metadata_note_rows=1` trong raw | `metadata_note_rows=0` trong cleaned | `data/raw/policy_export_dirty.csv`, `artifacts/cleaned/cleaned_clean-2026-04-15.csv` |
| `normalize_it_ticket_terms` | `Ticket/ticket=3` trong raw | `Ticket/ticket=0`, `Yêu cầu hỗ trợ/yêu cầu hỗ trợ=3` trong cleaned | `data/raw/policy_export_dirty.csv`, `artifacts/cleaned/cleaned_clean-2026-04-15.csv` |
| `collapse_internal_whitespace` | text chưa có canonical normalization trước embed | 11/11 cleaned rows đi qua bước chuẩn hóa trước khi sinh `chunk_id` | `transform/cleaning_rules.py`, `artifacts/cleaned/cleaned_clean-2026-04-15.csv` |
| `max_chunk_length_500` | chưa có cảnh báo độ dài | `FAIL (warn): long_chunks=1` ở cả run sạch và inject | `artifacts/logs/run_clean-2026-04-15.log`, `artifacts/logs/run_inject-bad-2026-04-15.log` |
| `no_suspicious_placeholders` | chưa có guardrail chặn placeholder | `OK (halt): suspicious_rows=0` | `artifacts/logs/run_clean-2026-04-15.log` |

**Rule chính (baseline + mở rộng):**

- Baseline: allowlist `doc_id`, chuẩn hóa `effective_date`, quarantine bản HR cũ, loại duplicate, fix refund 14 → 7 ngày.
- Mở rộng: xóa metadata note cuối câu, chuẩn hóa thuật ngữ IT, nén khoảng trắng thừa trước khi hash `chunk_id`.

**Ví dụ 1 lần expectation fail và cách xử lý:**

Ở run `inject-bad-2026-04-15`, nhóm cố ý dùng `--no-refund-fix --skip-validate` để giữ lại nội dung stale. Log ghi `expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1`, sau đó chỉ tiếp tục embed vì có cờ `--skip-validate`. Khi chạy lại run chuẩn `clean-2026-04-15` không dùng cờ inject, expectation này trở về `OK`, còn `hits_forbidden` của câu hỏi refund trong file eval giảm từ `yes` xuống `no`.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200-250 từ)

Kịch bản inject của nhóm bám đúng README Sprint 3: chạy `python etl_pipeline.py run --run-id inject-bad-2026-04-15 --no-refund-fix --skip-validate`, sau đó đánh giá bằng `python eval_retrieval.py --out artifacts/eval/before_inject_bad.csv`. Mục tiêu của lần chạy này không phải để có pipeline “đúng”, mà để chứng minh rằng nếu publish dữ liệu chưa clean thì retrieval vẫn có thể trả lời bằng context stale dù top-k nhìn qua có vẻ hợp lý. Sau đó nhóm chạy lại pipeline chuẩn bằng `python etl_pipeline.py run --run-id clean-2026-04-15` và xuất `artifacts/eval/after_clean.csv`.

Tác động rõ nhất nằm ở câu `q_refund_window`. Ở file `artifacts/eval/before_inject_bad.csv`, `top1_doc_id=policy_refund_v4`, `contains_expected=yes` nhưng `hits_forbidden=yes`; preview top-1 chứa câu “14 ngày làm việc”, chứng minh stale chunk vẫn lọt vào retrieval. Ở file `artifacts/eval/after_clean.csv`, câu hỏi này vẫn `contains_expected=yes` nhưng `hits_forbidden=no`, vì chunk refund đã được sửa về “7 ngày làm việc” trước khi embed. Điều này đúng tinh thần observability của lab: câu trả lời chỉ được xem là an toàn khi top-k không còn kéo theo forbidden context.

Với `q_leave_version`, cả hai run đều cho `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes`. Điều này phù hợp với cleaning baseline của repo vì các bản HR 2025 đã bị đưa vào quarantine trước bước embed. Như vậy, before/after trong repo hiện tại cho thấy refund window là lỗi dữ liệu có tác động retrieval rõ nhất, còn HR version là case đã được chặn ổn định từ lớp clean/quarantine.

**Kịch bản inject:**

- Run xấu: `inject-bad-2026-04-15`
- Lệnh: `python etl_pipeline.py run --run-id inject-bad-2026-04-15 --no-refund-fix --skip-validate`
- Eval: `artifacts/eval/before_inject_bad.csv`

**Kết quả định lượng (từ CSV / bảng):**

- `q_refund_window`: `hits_forbidden=yes` ở run xấu → `hits_forbidden=no` ở run sạch
- `q_leave_version`: `top1_doc_expected=yes` ở cả hai run
- Run xấu có `refund_no_stale_14d_window FAIL (halt) :: violations=1`

---

## 4. Freshness & monitoring (100-150 từ)

Nhóm giữ cấu hình SLA mặc định `FRESHNESS_SLA_HOURS=24` như trong `.env.example`. Ở cả hai manifest `manifest_inject-bad-2026-04-15.json` và `manifest_clean-2026-04-15.json`, trường `latest_exported_at` đều là `2026-04-10T08:00:00`, trong khi thời điểm chạy pipeline là ngày `2026-04-15`. Vì vậy `python etl_pipeline.py freshness --manifest ...` trả về `FAIL` với `reason=freshness_sla_exceeded` và `age_hours` khoảng 121.5 giờ.

Nhóm xem đây là hành vi đúng, không phải lỗi hệ thống. Freshness ở repo hiện tại đang đo “độ mới của snapshot dữ liệu nguồn” chứ không đo “độ mới của lần chạy pipeline”. Do sample CSV cố ý dùng `exported_at` cũ, pipeline vẫn `PIPELINE_OK` nhưng monitoring báo `FAIL` để cảnh báo corpus đang stale. Trong runbook, nhóm ghi rõ ý nghĩa này để tránh hiểu nhầm rằng rerun thành công đồng nghĩa dữ liệu đã mới.

---

## 5. Liên hệ Day 09 (50-100 từ)

Day 10 phục vụ trực tiếp cho Day 09 ở lớp dữ liệu: cùng case CS + IT helpdesk, cùng bộ canonical docs trong `data/docs/`, nhưng pipeline Day 10 bổ sung thêm boundary clean/validate/publish trước khi agent truy xuất. Repo đang dùng collection riêng `day10_kb` để tách khỏi các thí nghiệm Day 09 trước đó và tránh stale vector làm nhiễu đánh giá. Nếu tích hợp lại với multi-agent Day 09, nhóm sẽ giữ nguyên retrieval flow nhưng đổi nguồn ingest sang cleaned snapshot của Day 10.

---

## 6. Rủi ro còn lại & việc chưa làm

- `max_chunk_length_500` vẫn đang `FAIL (warn)` vì còn 1 chunk quy trình P1 quá dài; nhóm chưa tách nhỏ chunk này trước khi embed.
- `.env` chưa được commit trong repo, nên nhóm đang dựa vào mặc định của `.env.example`; khi demo trên máy khác cần tạo `.env` rõ ràng.
- `docs/quality_report.md` cần được đồng bộ tuyệt đối với run cuối cùng nếu nhóm đổi `run_id` trước lúc nộp.
- Repo chưa có `data/grading_questions.json`, nên chưa thể xuất `artifacts/eval/grading_run.jsonl` trên bản local hiện tại.

---

## 7. Peer Review 3 Câu Hỏi

1. Nếu `freshness_check=FAIL` nhưng pipeline vẫn `PIPELINE_OK`, nhóm có cho phép publish hay phải chặn phát hành ở môi trường production?
2. Với case `q_p1_sla`, vì sao top-1 hiện chưa phải `sla_p1_2026` dù `contains_expected=yes`, và nhóm có cần cải thiện chunking hoặc retrieval ranking không?
3. Rule chuẩn hóa nội dung nào nên giữ ở mức auto-fix, và rule nào nên chuyển thành quarantine để tránh “sửa quá tay” nguồn dữ liệu?
