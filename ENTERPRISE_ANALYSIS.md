# DeepTutor: Enterprise Transformation & Architecture Analysis

Tài liệu này tổng hợp các phân tích về hệ thống DeepTutor hiện tại và lộ trình nâng cấp để phục vụ quy mô công ty với khả năng phân quyền người dùng.

## 1. Phân tích Hệ thống lưu trữ Hiện tại (Personal Mode)

DeepTutor được thiết kế theo triết lý **Local-First**, dữ liệu được lưu trữ hoàn toàn dưới dạng file tại thư mục `/data`:

### 📂 Cấu trúc Cơ sở dữ liệu (File-based DB)
*   **Hội thoại & Cấu hình (`data/user/`)**:
    *   `chat_sessions.json`: Lưu lịch sử chat toàn bộ.
    *   `settings/`: Lưu cấu hình Model và API Key.
*   **Kiến thức (RAG - `data/knowledge_bases/`)**:
    *   Sử dụng **LightRAG** để tạo đồ thị kiến thức (Knowledge Graph).
    *   Mỗi "Knowledge Base" là một thư mục riêng biệt chứa các file Vector Index và cấu trúc liên kết.

---

## 2. Giải pháp Kết nối CMS & Phân quyền (Corporate Mode)

Để triển khai cho công ty thông qua một hệ thống CMS riêng, chúng ta sẽ sử dụng chiến lược **Logical Isolation** (Cô lập logic).

### 🛡️ Chiến lược "Namespace" cho Knowledge Base
Để tránh xung đột dữ liệu giữa các User, CMS của bạn sẽ quản lý việc đặt tên KB theo quy tắc:
`kb_[USER_ID]_[DEPARTMENT]`

**Ví dụ:**
*   User 001 (Kế toán): `kb_001_finance`
*   User 002 (Nhân sự): `kb_002_hr`

Khi DeepTutor nhận lệnh, nó sẽ tự động tạo thư mục riêng:
*   `data/knowledge_bases/kb_001_finance/`
*   `data/knowledge_bases/kb_002_hr/`
=> **Dữ liệu hoàn toàn tách biệt, không bao giờ xảy ra xung đột.**

### 🔗 Sơ đồ kết nối API
CMS của bạn sẽ đóng vai trò là một **Proxy/Gateway**:

1.  **Quản lý Tài liệu**: CMS cho phép user upload file -> CMS gọi API `POST /knowledge/{kb_name}/upload` của DeepTutor.
2.  **Hỏi đáp (RAG)**: CMS mở kết nối WebSocket tới `/chat` và gửi kèm `kb_name` tương ứng với user đó.
3.  **Bảo mật**: CMS thực hiện kiểm tra quyền trước khi chuyển tiếp yêu cầu tới DeepTutor.

---

## 3. Lộ trình Triển khai (Roadmap)

### Giai đoạn 1: POC (Proof of Concept)
*   Tiếp tục dùng DeepTutor hiện tại làm "Engine" xử lý AI.
*   Xây dựng CMS đơn giản (Next.js/Laravel/Python) quản lý người dùng.
*   CMS gọi API DeepTutor bằng các tên KB khác nhau để kiểm chứng sự cô lập.

### Giai đoạn 2: Enterprise Hardening
*   **Auth Layer**: Bổ sung Keycloak hoặc Auth0 để quản lý đăng nhập tập trung.
*   **Database Migration**: Chuyển các file `.json` sang **PostgreSQL** để quản lý hàng ngàn lịch sử chat hiệu quả hơn.
*   **Security Shell**: Đặt DeepTutor đằng sau Nginx hoặc API Gateway để chặn mọi truy cập trái phép vào API.

### Giai đoạn 3: Scaling
*   Triển khai trên **Docker Swarm** hoặc **Kubernetes**.
*   Sử dụng chung một hệ thống LLM Key (như Gemini Key hiện tại) nhưng quản lý hạn ngạch (Quota) tại lớp CMS.

---

## 4. Tổng kết kỹ thuật (Verified)

*   **LLM Model:** `gemini-flash-latest` (Định danh ổn định nhất).
*   **Embedding:** `gemini-embedding-001` (Khổ 768).
*   **Khả năng mở rộng:** Rất cao nhờ kiến trúc API RESTful và WebSocket có sẵn.
