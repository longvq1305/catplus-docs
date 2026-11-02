# CatPlus Platform — Chính sách Hệ thống Trợ giảng AI (AI Hints) & Ngân sách Gemini

Tài liệu này định nghĩa khung chính sách sư phạm, quy chuẩn kỹ thuật và phương án kiểm soát ngân sách cho tính năng Trợ giảng AI (AI Hints) tích hợp trong CatPlus Platform.

---

## 1. Triết lý thiết kế nghiệp vụ (Socratic Pedagogy)

- **Không giải hộ / Không cung cấp code hoàn chỉnh:** AI đóng vai trò định hướng bằng các câu hỏi gợi mở (Socratic questions), chỉ ra trường hợp biên (edge cases) chưa xét hoặc vị trí tiềm ẩn lỗi logic.
- **Xử lý kết quả chấm theo ai-hints-v2:** Đọc kết quả chấm chi tiết từ JudgeAttempt (WA, TLE, MLE, RTE, CE, OLE, System Error...).
- **Chẩn đoán chuyên sâu theo kết quả chấm (Verdict-driven Diagnosis - ai-hints-v2):** Mỗi yêu cầu gồm đúng 1 lượt sinh nội dung logic cho ca được chọn (không gọi riêng theo testcase; lượt thử HTTP vật lý có thể tăng do SDK retry/failover). Phân tích chính xác theo kết quả thực tế từ Judge:
  - `WA (Wrong Answer)`: Chỉ ra sai lệch logic, điều kiện biên, gợi ý test phản ví dụ để tự mô phỏng (chỉ dùng test công khai hoặc test đã được phép hiển thị).
  - `TLE (Time Limit Exceeded)`: Nhận diện điểm nghẽn thuật toán ($O(N^2)$ vs $O(N \log N)$), cấu trúc lặp vô hạn hoặc thiếu điều kiện dừng so với kích thước đầu vào.
  - `OLE (Output Limit Exceeded)`: Nhắc nhở loại bỏ lệnh debug log gây tràn giới hạn xuất dữ liệu, vòng lặp in vô hạn.
  - `Lỗi hạ tầng / System Error`: Tách biệt sự cố máy chấm, không quy kết lỗi cho code học viên.
  - `Unknown verdict`: Phân tích thận trọng chỉ dựa trên chứng cứ chắc chắn.

---

## 2. Mô hình và Kỹ thuật vận hành (Model & Production Resilience)

- **Lựa chọn mô hình AI:** Google Gemini (gemini-3.8-flash hiện hành; đề xuất kinh tế dài hạn: gemini-3.5-flash-lite).
- **Tự động xoay key (Auto-failover):** Khi một key gặp lỗi rate-limit (`HTTP 429` hoặc `RESOURCE_EXHAUSTED`), hệ thống đánh dấu key bị rate-limit tạm thời trên Redis và tự động chuyển sang key tiếp theo. Toàn bộ các key trong cùng một Google Cloud Project **chia sẻ chung hạn mức RPM/TPM/RPD**.
- **Cơ chế Cooldown hiện hành:** Giãn cách 600s giữa các lượt thành công theo bài; tự động mở khóa sớm khi có bài nộp mới hợp lệ.
- **Định dạng phản hồi:** Yêu cầu Gemini trả về JSON có cấu trúc nghiêm ngặt (Strict JSON Schema) chứa số dòng (`line`) và nội dung hướng dẫn (`message`).

---

## 3. Khung chính sách vận hành đề xuất & Kiểm soát chi phí $\le 100$ USD/tháng

- **Hạn mức ngày:** Khuyến nghị **4 lượt gợi ý mới thành công / học viên / ngày** đối với mô hình tiết kiệm **Gemini 3.5 Flash-Lite** (hoặc 3 lượt/ngày với 3.8 Flash), múi giờ `Asia/Ho_Chi_Minh`, không cộng dồn.
- **Giãn cách toàn cục:** 10 phút toàn cục xuyên suốt mọi bài toán, **không mở khóa sớm (no early unlock)** để rèn luyện thói quen tự suy ngẫm.
- **Khử trùng lặp ngữ nghĩa:** Tái sử dụng miễn phí kết quả khi cùng học viên, cùng phiên bản bài, cùng mã nguồn, cùng chứng cứ chấm bài, cùng phiên bản prompt và mô hình.
- **Ngân sách an toàn:** Dự trữ 80 USD chi phí API Google + 20 USD dự phòng thuế phí trong tổng ngân sách tối đa 100 USD/tháng cho quy mô 100 học viên (bao gồm 20% dự phòng lượt gọi bổ sung).
- **Rào cản pháp lý điều khoản độ tuổi (18+ Blocker):** Điều khoản dịch vụ Google Gemini API (kiểm tra 2026-11-02 tại `https://ai.google.dev/gemini-api/terms`) cấm dịch vụ phục vụ người dùng dưới 18 tuổi; gói Tier 1 và sự đồng ý của phụ huynh không miễn trừ. Việc triển khai trực tiếp cho học sinh phổ thông (< 18 tuổi) hiện là một **rào cản bắt buộc dừng (hard rollout blocker)**.
- *(Xem chi tiết đầy đủ chính sách và bảng dự báo chi phí tại [Báo cáo chức năng nền tảng](docs/CHUC_NANG.md) Mục 3.7).*
