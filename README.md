# CatPlus Platform — Hệ thống Quản trị Đào tạo & Nền tảng Chấm bài Trực tuyến (Online Judge) tích hợp Trợ giảng AI

> **Dự án thực tế:** Nền tảng nội bộ toàn diện phục vụ quản trị trung tâm đào tạo lập trình, học tập tương tác (Learning Web), quy trình trợ giảng/vận hành, tích hợp Codeforces Polygon, hạ tầng chấm bài phân tán DMOJ và **hệ thống Trợ giảng AI (AI Hints)** trên nền Google Gemini 3.8 Flash.

[![Architecture](https://img.shields.io/badge/Architecture-Microservices-blue.svg)](docs/CHUC_NANG.md)
[![Backend](https://img.shields.io/badge/Backend-Python%203.13%20%7C%20Django%20%7C%20uv-green.svg)](docs/CHUC_NANG.md)
[![Frontend](https://img.shields.io/badge/Frontend-Next.js%2015%20%7C%20React%2019%20%7C%20TypeScript-black.svg)](docs/CHUC_NANG.md)
[![AI Integration](https://img.shields.io/badge/AI%20Engine-Gemini%203.8%20Flash%20(High%20Thinking)-orange.svg)](docs/CHUC_NANG.md)
[![Judge Engine](https://img.shields.io/badge/Judge-DMOJ%20%2B%20Polygon%20Sync-purple.svg)](docs/SUA_LOI_DONG_BO.md)

---

## 📌 Mục lục

1. [Tổng quan dự án](#-tổng-quan-dự-án)
2. [Điểm nhấn kỹ thuật mới: Hệ thống Trợ giảng AI (AI Hints)](#-điểm-nhấn-kỹ-thuật-mới-hệ-thống-trợ-giảng-ai-ai-hints)
3. [Kiến trúc hệ thống & Thiết kế hướng miền (Architecture & DDD)](#-kiến-trúc-hệ-thống--thiết-kế-hướng-miền-architecture--ddd)
4. [Hạ tầng Online Judge & Đồng bộ Polygon](#-hạ-tầng-online-judge--đồng-bộ-polygon)
5. [Ngăn xếp công nghệ (Tech Stack)](#-ngăn-xếp-công-nghệ-tech-stack)
6. [Tài liệu kỹ thuật chi tiết](#-tài-liệu-kỹ-thuật-chi-tiết)
7. [Hướng dẫn cài đặt & Khởi chạy (Local Development)](#-hướng-dẫn-cài-đặt--khởi-chạy-local-development)
8. [Tài khoản thử nghiệm & Dữ liệu mẫu](#-tài-khoản-thử-nghiệm--dữ-liệu-mẫu)

---

## 🌟 Tổng quan dự án

**CatPlus Platform** là giải pháp phần mềm quản trị và đào tạo lập trình thi đấu (Competitive Programming) end-to-end, giải quyết triệt để bài toán đồng bộ giữa nghiệp vụ vận hành trung tâm và môi trường học tập thực chiến của học viên.

Hệ thống được thiết kế theo kiến trúc **Microservices** hướng module độc lập với ranh giới nghiệp vụ (domain boundary) nghiêm ngặt:
- **Quản trị vận hành (Operations Web):** Quản lý học viên, lớp học, buổi học, điểm danh, bài tập, tuyển sinh, học phí, công nợ, sổ quỹ và phân quyền nhân sự.
- **Không gian học tập (Learning Web):** Cung cấp không gian làm bài trực tiếp với Monaco Editor, tra cứu kho chủ đề/bộ bài tập, nộp bài thời gian thực và nhận gợi ý thông minh từ AI.
- **Lõi chấm bài chuẩn quốc tế (OJ & DMOJ Engine):** Tích hợp sâu với hệ thống chuẩn hóa bài toán Codeforces Polygon và cụm máy chủ chấm DMOJ phân tán có cơ chế sandbox bảo mật.

---

## 🤖 Điểm nhấn kỹ thuật mới: Hệ thống Trợ giảng AI (AI Hints)

Tính năng **AI Hints** được phát triển nhằm đóng vai trò **Trợ giảng định hướng (Socratic AI Tutor)** thông minh trong không gian làm bài của học viên, giúp giải tỏa bế tắc khi giải thuật toán mà **không làm mất đi tư duy tự học của học sinh**.

```
Học viên (Monaco Editor)
   │
   ▼
Learning Web (Next.js Proxy / Rate-limit UI)
   │  Gửi code hiện tại hoặc submission_id
   ▼
Dịch vụ OJ (Backend Coordinator)
   ├── Xác thực học viên & kiểm tra đề bài authoritative
   ├── Đọc kết quả chấm chi tiết từ JudgeAttempt (WA, TLE, MLE, RTE, CE, OLE, System Error...)
   ├── Redis Multi-Key Pool (chọn API Key tối ưu, xoay key khi 429)
   ├── Redis Distributed Lock (ngăn click đúp, chống spam)
   └── Xây dựng Prompt chuyên biệt ai-hints-v2 theo từng loại lỗi
           │
           ▼
   Google Gemini (gemini-3.8-flash hiện hành; đề xuất kinh tế dài hạn: gemini-3.5-flash-lite)
           │
           ▼
   Gợi ý cấu trúc theo dòng (Line-level comments & hints)
           │
           ▼
Hiển thị trực quan trên Monaco Editor (Glyph margin, amber highlight, hover tooltip)
```

### 1. Triết lý thiết kế nghiệp vụ (Socratic Pedagogy)
- **Không giải hộ / Không cung cấp code hoàn chỉnh:** AI đóng vai trò định hướng bằng các câu hỏi gợi mở (Socratic questions), chỉ ra trường hợp biên (edge cases) chưa xét hoặc vị trí tiềm ẩn lỗi logic.
- **Hai chế độ hỗ trợ linh hoạt:**
  - **Gợi ý trên code đang soạn thảo (Current Code Hints):** Dành cho học viên đang viết dở bài, giúp định hướng cấu trúc thuật toán tiếp theo mà không bắt buộc phải nộp bài.
  - **Chẩn đoán chuyên sâu theo kết quả chấm (Verdict-driven Diagnosis - ai-hints-v2):** Mỗi yêu cầu gồm đúng 1 lượt sinh nội dung logic cho ca được chọn (không gọi riêng theo testcase; lượt thử HTTP vật lý có thể tăng do SDK retry/failover). Phân tích chính xác theo kết quả thực tế từ Judge:
    - `WA (Wrong Answer)`: Chỉ ra sai lệch logic, điều kiện biên, gợi ý test phản ví dụ để tự mô phỏng (chỉ dùng test công khai hoặc test đã được phép hiển thị).
    - `TLE (Time Limit Exceeded)`: Nhận diện điểm nghẽn thuật toán ($O(N^2)$ vs $O(N \log N)$), cấu trúc lặp vô hạn hoặc thiếu điều kiện dừng so với kích thước đầu vào.
    - `MLE (Memory Limit Exceeded)`: Định hướng tối ưu không gian bộ nhớ (ví dụ: chuyển từ ma trận kề $N \times N$ sang danh sách kề).
    - `RTE (Runtime Error)`: Phát hiện truy cập mảng ngoài giới hạn (out-of-bounds), tràn số, chia cho 0 hoặc lỗi con trỏ.
    - `CE (Compile Error)`: Phân tích thông báo của trình biên dịch $C++$ và hướng dẫn sửa lỗi cú pháp.
    - `OLE (Output Limit Exceeded)`: Nhắc nhở loại bỏ lệnh debug log gây tràn giới hạn xuất dữ liệu, vòng lặp in vô hạn.
    - `Lỗi hạ tầng / System Error`: Tách biệt sự cố máy chấm, không quy kết lỗi cho code học viên.
    - `Unknown verdict`: Phân tích thận trọng chỉ dựa trên chứng cứ chắc chắn.

### 2. Kỹ thuật sản xuất, Khung chính sách & Kiểm soát ngân sách (Production Resilience & Policy)
- **Multi-Key Failover Pool thông minh trên Redis:**
  - Quản lý danh sách nhiều khóa API Google Gemini (`GEMINI_API_KEYS`).
  - **Sticky Key Affinity:** Giữ nguyên key đang active khi các lượt gọi thành công để tận dụng tối đa cache.
  - **Tự động xoay key (Auto-failover):** Khi một key gặp lỗi rate-limit (`HTTP 429` hoặc `RESOURCE_EXHAUSTED`), hệ thống đánh dấu key bị rate-limit tạm thời trên Redis và tự động chuyển sang key tiếp theo. Toàn bộ các key trong cùng một Google Cloud Project **chia sẻ chung hạn mức RPM/TPM/RPD**.
  - **Bảo mật danh tính key:** Key bên ngoài bộ nhớ tiến trình chỉ được định danh bằng tiền tố mã băm **SHA-256 fingerprint**, tuyệt đối không lưu plaintext key vào Redis/Log.
- **Chống lạm dụng & Khóa phân tán (Anti-abuse & Concurrency Control):**
  - **Atomic Inflight Lock (`90s`):** Sử dụng Redis key `ai-hint:inflight:{actor_id}:{problem_id}` để chặn gửi yêu cầu trùng lặp hoặc spam click đa tab.
  - **Cơ chế Cooldown hiện hành:** Giãn cách 600s giữa các lượt thành công theo bài; tự động mở khóa sớm khi có bài nộp mới hợp lệ.
- **Khung chính sách vận hành đề xuất & Kiểm soát chi phí $\le 100$ USD/tháng:**
  - **Hạn mức ngày:** Khuyến nghị **4 lượt gợi ý mới thành công / học viên / ngày** đối với mô hình tiết kiệm **Gemini 3.5 Flash-Lite** (hoặc 3 lượt/ngày với 3.8 Flash), múi giờ `Asia/Ho_Chi_Minh`, không cộng dồn.
  - **Giãn cách toàn cục:** 10 phút toàn cục xuyên suốt mọi bài toán, **không mở khóa sớm (no early unlock)** để rèn luyện thói quen tự suy ngẫm.
  - **Khử trùng lặp ngữ nghĩa:** Tái sử dụng miễn phí kết quả khi cùng học viên, cùng phiên bản bài, cùng mã nguồn, cùng chứng cứ chấm bài, cùng phiên bản prompt và mô hình.
  - **Ngân sách an toàn:** Dự trữ 80 USD chi phí API Google + 20 USD dự phòng thuế phí trong tổng ngân sách tối đa 100 USD/tháng cho quy mô 100 học viên (bao gồm 20% dự phòng lượt gọi bổ sung).
  - **Rào cản pháp lý điều khoản độ tuổi (18+ Blocker):** Điều khoản dịch vụ Google Gemini API (kiểm tra 2026-11-02 tại `https://ai.google.dev/gemini-api/terms`) cấm dịch vụ phục vụ người dùng dưới 18 tuổi; gói Tier 1 và sự đồng ý của phụ huynh không miễn trừ. Việc triển khai trực tiếp cho học sinh phổ thông (< 18 tuổi) hiện là một **rào cản bắt buộc dừng (hard rollout blocker)**.
  - *(Xem chi tiết đầy đủ chính sách và bảng dự báo chi phí tại [Báo cáo chức năng nền tảng](docs/CHUC_NANG.md) Mục 3.7).*
- **Phối hợp ngân sách thời gian (Timeout Budget Alignment):**
  - Thiết kế thời gian timeout đa tầng nghiêm ngặt chống treo kết nối:
    $$\text{Gemini SDK (62s)} < \text{Gunicorn Worker (70s)} < \text{Nginx Gateway (80s)} < \text{Redis Lock (90s)}$$
- **Xác thực cấu trúc & Tích hợp Editor hoàn chỉnh:**
  - Yêu cầu Gemini trả về JSON có cấu trúc nghiêm ngặt (Strict JSON Schema) chứa số dòng (`line`) và nội dung hướng dẫn (`message`).
  - Giao diện Monaco Editor hiển thị glyph margin tròn màu hổ phách, highlight nền dòng code và bảng thông báo rõ ràng; tự động hủy gắn kết nếu học viên đã sửa code trong thời gian chờ AI phản hồi.
---

## 🏛 Kiến trúc hệ thống & Thiết kế hướng miền (Architecture & DDD)

Hệ thống tuân thủ nguyên tắc **Single Owning Service**: mỗi thực thể dữ liệu có đúng một service sở hữu duy nhất. Không có hiện tượng cross-database queries, cross-model imports hay cross-service foreign keys. Liên kết giữa các service sử dụng External UUIDs, Event Snapshots hoặc Read-Model Projections.

### Bảng phân định ranh giới Service

| Thành phần | Đường dẫn | Công nghệ | Trách nhiệm chính |
| :--- | :--- | :--- | :--- |
| **Operations Web** | `apps/web` | Next.js 15, Tailwind CSS | Ứng dụng quản trị dành cho Quản lý, CSKH và Trợ giảng. |
| **Learning Web** | `apps/learning-web` | Next.js 15, Monaco Editor | Không gian học tập, làm bài, nộp bài và tương tác AI của học viên. |
| **Identity Service** | `services/identity` | Python 3.13, Django | Xác thực tập trung RS256 JWT, Refresh token rotation, RBAC, User lifecycle và Audit log. |
| **Academic Service** | `services/academic` | Python 3.13, Django | Quản lý học viên, lớp học, lộ trình, điểm danh, bài tập và ngữ cảnh lớp. |
| **Customer Ops Service** | `services/customer-operations` | Python 3.13, Django | Tuyển sinh, hành trình học viên, học phí, công nợ, hóa đơn và sổ quỹ. |
| **Reporting Service** | `services/reporting` | Python 3.13, Django | Mô hình đọc (Read-model boundary), tổng hợp chỉ số báo cáo và phân tích. |
| **OJ Service** | `services/oj` | Python 3.13, Django, Redis | Danh mục đề bài, Topics, Problem Sets, Submissions, điều phối AI Hints. |
| **Polygon Sync Service**| `services/polygon-sync` | Python 3.13, Django | Đồng bộ gói đề thi từ Codeforces Polygon, giải mã package và quản lý phiên bản. |
| **Judge Bridge** | `services/judge-bridge`| Python 3.13, Django | Cầu nối điều phối giữa CatPlus OJ và cụm máy chủ chấm DMOJ. |
| **DMOJ Judge Runtime** | `infrastructure/dmoj` | C++, Python, DMOJ | Cụm máy chấm phân tán, Dynamic problem discovery và Prewarm trusted checker. |
| **API Gateway** | `infrastructure/gateway`| Nginx | Điểm tiếp nhận traffic tập trung, phân luồng reverse proxy và điều chỉnh timeout. |

---

## ⚡ Hạ tầng Online Judge & Đồng bộ Polygon

Một trong những thế mạnh cốt lõi của nền tảng là khả năng tích hợp trực tiếp với chuẩn bài toán công nghiệp **Codeforces Polygon**:

```
Codeforces Polygon API
        │ (Sync Pipeline / Package Ingestion)
        ▼
services/polygon-sync
        │ (Authoritative Package Projection)
        ▼
services/oj (Judge Adapter Projection)
        │
        ▼
services/judge-bridge
        ├── DMOJ Judge Worker 1 (Sandbox C++17)
        └── DMOJ Judge Worker 2 (Sandbox C++17)
```

- **Đồng bộ đề bài Authoritative:** Khắc phục triệt để độ trễ render revision từ Polygon, tự động phát hiện lệch revision và fallback tải gói hoàn chỉnh.
- **Hạ tầng Judge phân tán:**
  - Hỗ trợ Dynamic problem discovery không cần khởi động lại Judge.
  - Persistent checker cache volume tách biệt cho từng worker.
  - Background prewarm checker executables đáng tin cậy.
  - Cô lập hoàn toàn mã nguồn học viên trong sandbox bảo mật.

---

## 💻 Ngăn xếp công nghệ (Tech Stack)

- **Frontend:** Next.js 15 (App Router), React 19, TypeScript, Tailwind CSS, Monaco Editor (VS Code core), Lucide Icons.
- **Backend & APIs:** Python 3.13, Django 5, Django REST Framework, `uv` (Ultra-fast Python package manager), Gunicorn.
- **Trí tuệ nhân tạo (AI):** Google Gemini API (`gemini-3.8-flash` với High Thinking Level), Google GenAI SDK, Structured JSON Outputs.
- **Cơ sở dữ liệu & Bộ nhớ đệm:** PostgreSQL 16 (Logical multi-database theo service), Redis 7 (Distributed lock, Cooldown tracking, Rate-limit coordinator).
- **Online Judge & Sandbox:** DMOJ Judge Core, Polygon API Client, C++17 Runtime Sandbox.
- **DevOps & Hạ tầng:** Docker Engine, Docker Compose v2, Nginx Gateway, GNU Make, GitHub Actions CI/CD.

---

## 📚 Tài liệu kỹ thuật chi tiết

Các tài liệu nghiên cứu, đặc tả và báo cáo chi tiết được lưu trữ trực tiếp trong repository này:

- 📄 **[Báo cáo toàn diện chức năng nền tảng & Trợ giảng AI](docs/CHUC_NANG.md):** Khảo sát chi tiết tính năng toàn hệ thống kèm ảnh chụp minh họa thực tế giao diện và các ca chẩn đoán AI thật (WA, TLE, MLE, RTE, CE, OLE).
- 📄 **[Báo cáo kỹ thuật sửa lỗi và nâng cấp đồng bộ Polygon](docs/SUA_LOI_DONG_BO.md):** Phân tích lỗi lệch revision, thiết kế pipeline đồng bộ bất đồng bộ và cơ chế bảo vệ package.
- **[Báo cáo Nâng cấp tính năng Trợ giảng AI](docs/BAO_CAO_NANG_CAP_TRO_GIANG_AI.md):** Trình bày chức năng gợi ý khi làm bài, cách xử lý theo kết quả chấm, bảo vệ dữ liệu và kiểm soát sử dụng.

---

## 🚀 Hướng dẫn cài đặt & Khởi chạy (Local Development)

### Yêu cầu hệ thống
- Docker Engine & Docker Compose v2
- Node.js 22 & pnpm 10
- Python 3.13 & `uv`
- GNU Make (Tùy chọn)

### Các bước khởi chạy nhanh

1. **Khởi tạo môi trường & Dependencies:**
   ```powershell
   python scripts/bootstrap_local.py
   ```

2. **Xây dựng và khởi chạy Docker Compose stack:**
   ```powershell
   docker compose build
   docker compose up -d
   ```

3. **Chạy Migration và Seed dữ liệu hệ thống:**
   ```powershell
   docker compose exec identity-api .venv/bin/python manage.py migrate --noinput
   docker compose exec academic-api .venv/bin/python manage.py migrate --noinput
   docker compose exec customer-operations-api .venv/bin/python manage.py migrate --noinput
   docker compose exec reporting-api .venv/bin/python manage.py migrate --noinput
   docker compose exec identity-api .venv/bin/python manage.py seed_identity
   ```

4. **Kích hoạt tính năng AI Hints (Tùy chọn trong `.env`):**
   ```env
   AI_HINTS_ENABLED=true
   GEMINI_API_KEYS=your_google_gemini_api_key_here
   GEMINI_MODEL=gemini-3.8-flash
   GEMINI_THINKING_LEVEL=high
   ```

### Địa chỉ truy cập cục bộ (Local URLs)

- **Cổng Quản trị Vận hành (Internal Web):** [http://localhost:8080](http://localhost:8080)
- **Cổng Học viên (Learning Web):** [http://localhost:8081](http://localhost:8081)
- **OJ Problem Sets / Library:** [http://localhost:8080/oj/problem-sets](http://localhost:8080/oj/problem-sets)
- **API Health Checks:**
  - Identity: `http://localhost:8080/api/identity/health/live`
  - Academic: `http://localhost:8080/api/academic/health/live`
  - Customer Ops: `http://localhost:8080/api/customer-operations/health/live`
  - OJ / Hints: `http://localhost:8080/api/oj/health/live`

---

## 👥 Tài khoản thử nghiệm & Dữ liệu mẫu

Sau khi chạy lệnh khởi tạo tài khoản mẫu:
```sh
python scripts/seed_internal_test_accounts.py
```

| Người dùng / Tài khoản | Email / Tên đăng nhập | Vai trò | Phân hệ ứng dụng |
|:---|:---|:---|:---|
| **Vũ Quốc Long, ADMIN** | `long.admin@catplus.local` | Quản lý cấp cao (Manager) | Internal Operations |
| **Vũ Quốc Long, OPERATIONAL** | `long.operational@catplus.local` | Trợ giảng & CSKH | Internal Operations |
| **Đặng Quang Minh, ADMIN** | `manager@catplus.local` | Quản lý hệ thống | Internal Operations |
| **Nguyễn Thị Nhật Đăng, ADMIN**| `nhat-dang.admin@catplus.local` | Quản lý vận hành | Internal Operations |
| **Học viên mẫu (Learner)** | `dmkhangHN2013` | Học viên (Student) | Learning Web |

*Mật khẩu thử nghiệm mặc định:* `Catplus@123` *(hoặc giá trị biến môi trường `CATPLUS_TEST_USER_PASSWORD`)*.

---

## 👨‍💻 Tác giả & Thông tin liên hệ

- **Tác giả:** Vũ Quốc Long (Long VQ)
- **GitHub:** [@longvq1305](https://github.com/longvq1305)
- **Repository:** [catplus-docs](https://github.com/longvq1305/catplus-docs)

---
*Tài liệu được cập nhật phục vụ mục đích thẩm định kỹ thuật và giới thiệu hồ sơ năng lực dự án.*
