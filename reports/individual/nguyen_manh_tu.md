# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Mạnh Tú  
**Vai trò:** Embed Owner & Team Coordinator  
**Ngày nộp:** 2026-04-15  
**Độ dài:** ~550 từ
**Repo bài cá nhân**: https://github.com/GDGoC-FPTU/data-pipeline-observability-TuNM17421

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Trong dự án Lab Day 10, tôi đảm nhận vai trò **Điều phối nhóm (Lead)** và phụ trách kỹ thuật phần **Embed & Idempotency Owner**.

**Các đầu việc cụ thể:**
- **Điều phối:** Khởi tạo repository `Nhom09-E403-Day10`, phân chia task cho các thành viên dựa trên năng lực, theo dõi tiến độ từng Sprint và hỗ trợ giải quyết các lỗi Unicode khi chạy pipeline trên Windows.
- **Kỹ thuật:** Hoàn thiện hàm `cmd_embed_internal` trong `etl_pipeline.py`, đảm bảo dữ liệu sạch được đẩy vào ChromaDB một cách an toàn.
- **Dữ liệu:** Chủ trì thảo luận nhóm để mở rộng bộ câu hỏi `grading_questions.json` lên 10 câu hỏi đa dạng (bao quát từ chính sách nhân sự đến FAQ kỹ thuật IT) để tăng độ phủ cho bước đánh giá.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

**Quyết định: Thực hiện Image/Snapshot Pruning (Xóa ID thừa) thay vì chỉ Upsert đơn thuần.**

Trong vai trò Embed Owner, tôi nhận thấy nếu chỉ sử dụng lệnh `upsert` của ChromaDB, chúng ta chỉ giải quyết được việc cập nhật các chunk cũ có cùng ID. Tuy nhiên, nếu một tài liệu bị thay đổi nội dung hoàn toàn hoặc bị xóa (như trường hợp sửa lỗi refund window khiến nội dung thay đổi dẫn đến `chunk_id` hash cũng thay đổi), các bản ghi cũ từ lần chạy trước sẽ vẫn tồn tại trong Vector Store như những "bóng ma" (stale vectors).

Tôi đã quyết định triển khai logic so sánh danh sách `ids` hiện có trong collection với danh sách `ids` vừa được làm sạch (`cleaned`). Những ID không thuộc snapshot hiện tại sẽ bị xóa bỏ hoàn toàn thông qua lệnh `col.delete(ids=drop)`. Quyết định này giúp đảm bảo tính **Consistency (Nhất quán)** giữa file `cleaned.csv` và Vector Store sau mỗi lần chạy pipeline thành công.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Lỗi: AI trả về đồng thời cả thông tin 7 ngày và 14 ngày (Stale Context persistence).**

**Triệu chứng:** Trong Sprint 3 (Inject corruption), hệ thống đánh giá `eval_retrieval.py` liên tục báo lỗi `hits_forbidden=yes` cho câu hỏi hoàn tiền, ngay cả khi tôi đã chạy lại pipeline bản sạch (`clean-run`). 

**Nguyên nhân:** Qua kiểm tra trực tiếp trong Chroma, tôi phát hiện do cơ chế sinh ID dựa trên mã băm nội dung (`_stable_chunk_id`), nên chunk "14 ngày" và chunk "7 ngày" có 2 ID khác nhau. Lệnh `upsert` chỉ thêm cái mới mà không tự mất đi cái cũ, khiến top-k retrieval kéo về cả hai, gây nhiễu cho AI.

**Xử lý:** Tôi đã tích hợp bước `embed_prune_removed` vào hàm embedding. Log hệ thống sau đó đã ghi nhận chính xác: `embed_prune_removed=1`. Điều này chứng minh hệ thống đã tự động dọn dẹp các kiến thức lỗi thời sau khi nội dung được sửa đổi.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Dưới đây là kết quả đối chiếu từ file evaluation của tôi (Run ID: `clean-2026-04-15`):

- **Trước (Xấu - `inject-bad`):** 
  `q_refund_window | top1: "Yêu cầu hoàn tiền... 14 ngày..." | hits_forbidden: yes`
- **Sau (Tốt - `clean-run`):** 
  `q_refund_window | top1: "Yêu cầu... 7 ngày..." | hits_forbidden: no`

**Dòng log chứng minh (etl_pipeline.py):**
`embed_upsert count=11 collection=day10_kb`
`embed_prune_removed=1`

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ triển khai **Semantic Overlap check** trước khi embed. Thay vì chỉ dedupe dựa trên chuỗi văn bản (exact match), tôi sẽ dùng model embedding để tính cosine similarity giữa các chunk mới và cũ. Nếu độ tương đồng >98%, hệ thống sẽ tự động gộp hoặc báo cáo để tránh lãng phí dung lượng vector store và giảm nhiễu retrieval.
