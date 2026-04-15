# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Lê Tuấn Đạt  
**Vai trò:** Cleaning & Quality Owner  
**Ngày nộp:** 2026-04-15  

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Trong dự án Day 10, tôi đảm nhận vai trò **Cleaning & Quality Owner**, chịu trách nhiệm chính tại hai module cốt lõi: `transform/cleaning_rules.py` và `quality/expectations.py`. 

Nhiệm vụ của tôi là thiết kế các quy tắc biến đổi dữ liệu từ dạng thô sang dạng sạch (Cleaned) và xây dựng bộ tiêu chuẩn kiểm định để chặn đứng dữ liệu xấu trước khi chúng được đẩy vào Vector Store. Tôi phối hợp chặt chẽ với **Ingestion Owner** để hiểu các lỗi phổ biến trong file CSV thô và làm việc với **Embed Owner** để đảm bảo các thay đổi về nội dung văn bản được phản ánh chính xác qua cơ chế `upsert` và `prune` của ChromaDB.

**Bằng chứng:** Đã thêm 3 quy tắc làm sạch (`remove_metadata_notes`, `normalize_it_terminology`, `clean_extra_whitespace`) và 2 expectations mới (`max_chunk_length_500`, `no_suspicious_placeholders`).

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Một quyết định kỹ thuật quan trọng tôi đã thực hiện là việc phân loại mức độ nghiêm trọng (**Severity**) giữa `warn` và `halt` cho các quy tắc kiểm định trong `expectations.py`.

Cụ thể, đối với quy tắc `no_suspicious_placeholders` (kiểm tra các từ khóa như TODO, FIXME), tôi quyết định đặt mức độ là **`halt`**. Lý do là vì sự xuất hiện của các từ khóa này trong Knowledge Base công khai thể hiện sự rò rỉ dữ liệu nháp của kỹ thuật viên, làm giảm uy tín của hệ thống hỗ trợ và có thể gây nhầm lẫn nghiêm trọng cho khách hàng. Ngược lại, với quy tắc `max_chunk_length_500`, tôi chỉ đặt mức **`warn`**. Dù chunk quá dài có thể làm giảm hiệu quả retrieval, nhưng nó không phải là lỗi sai lệch thông tin chết người; do đó, pipeline vẫn nên tiếp tục chạy để đảm bảo tính liên tục của dữ liệu, trong khi đội ngũ vận hành sẽ nhận được cảnh báo để tối ưu hóa sau.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Trong Sprint 2, tôi đã phát hiện một lỗi logic (anomaly) khi triển khai quy tắc xóa ghi chú rác trong ngoặc đơn. 

**Triệu chứng:** Sau khi chạy pipeline, dòng 2 trong tệp `cleaned_sprint2.csv` vẫn còn sót lại đoạn ghi chú `(ghi chú: bản sync cũ...)`. 
**Nguyên nhân:** Biểu thức chính quy (Regex) ban đầu của tôi là `r"\s*[\(\[].*?[\)\]]\s*$"` chỉ khớp nếu dấu đóng ngoặc nằm ở cuối dòng. Tuy nhiên, dữ liệu thực tế lại có dấu chấm kết thúc câu sau dấu đóng ngoặc (`...migration).`). 
**Xử lý:** Tôi đã tinh chỉnh Regex thành `r"\s*[\(\[].*?[\)\]][\.\s]*$"` để bao phủ cả trường hợp có dấu chấm hoặc khoảng trắng thừa ở cuối. Sau khi sửa, log hệ thống báo `embed_prune_removed=1`, xác nhận rằng nội dung văn bản đã thay đổi, `chunk_id` cũ bị xóa và thay thế bằng bản ghi đã sạch hoàn toàn.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Tôi sử dụng kết quả từ câu hỏi `q_refund_window` để chứng minh hiệu quả của các quy tắc làm sạch.

- **Trước khi sửa (Run ID: `inject-bad`)**: 
  - `top1_preview`: "Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc..."
  - `hits_forbidden`: **yes**
- **Sau khi sửa (Run ID: `sprint2-fix`)**:
  - `top1_preview`: "Yêu cầu hoàn tiền được chấp nhận trong vòng 7 ngày làm việc..."
  - `hits_forbidden`: **no**

Sự thay đổi từ `yes` sang `no` ở cột `hits_forbidden` là minh chứng rõ nhất cho việc bộ quy tắc làm sạch đã loại bỏ thành công nội dung lỗi thời theo yêu cầu của Hợp đồng dữ liệu.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ triển khai thư viện **Pydantic** để thực hiện kiểm định schema chặt chẽ hơn thay vì dùng Dictionary thông thường. Việc ép kiểu và kiểm tra ràng buộc (constraints) ở cấp độ class sẽ giúp phát hiện lỗi kiểu dữ liệu ngay tại tầng Transform, thay vì đợi đến tầng Validation mới phát hiện ra.
