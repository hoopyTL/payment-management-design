# 02. Architecture Design (AS-IS & TO-BE) — `payment-management`

> **Trạng thái thiết kế**: `Design Pending Approval`

---

## 1. Khảo sát Kiến trúc Hiện tại (AS-IS Analysis)

Theo quy định tại `.cursor/agents/design.md` (Step 2 — không giả định thành phần tồn tại nếu chưa kiểm tra thực tế):

- **Mã nguồn `src/Qtkd.Api/`**: **Chưa tồn tại** trong workspace hiện tại (chỉ có `.cursor/` và `docs/`).
- **Database & Tests**: Chưa được khởi tạo.
- **Kết luận AS-IS**: Thiết kế mới hoàn toàn (Greenfield) cho module `PaymentManagement` trong dự án `Qtkd.Api`.

---

## 2. Kiến trúc Giải pháp Đề xuất (TO-BE High-Level Design)

### 2.1. Architecture Overview & Architectural Style
- **Architectural Style**: **Modular Monolith với Clean / Layered Architecture tinh gọn** (ASP.NET Core Web API + PostgreSQL).
- **Lý do lựa chọn**:
  - Đặc tả (`BAITAP_TA_1-2.pdf`, Trang 1, Mục 1) nêu rõ: *"Module này chỉ bao gồm: lập, trình duyệt, duyệt/từ chối, điều chỉnh và hủy bảng thanh toán. Quản lý hợp đồng, bảng giá và sản lượng là module khác; module này chỉ đọc dữ liệu từ chúng."*
  - Kiến trúc Modular Monolith cho phép cô lập hoàn toàn miền nghiệp vụ `PaymentManagement` và giao tiếp với các module `Contract`, `Pricing`, `Volume` thông qua các **Read-Only Port / Query Interfaces** (`IContractReadService`, `IPriceListReadService`, `IVolumeReadService`), vừa đơn giản, dễ kiểm thử, vừa đáp ứng NFR phản hồi `< 3 giây` cho hợp đồng `< 200 dòng` (`Mục 6`).

### 2.2. Component Responsibilities (Trách nhiệm các thành phần)

| Thành phần | Layer | Trách nhiệm chính |
| :--- | :--- | :--- |
| **`PaymentStatementsController`** | Presentation (API) | Cung cấp REST API cho lập bảng (`US-PAY-01`), cập nhật bảng nháp (`US-PAY-01`), trình duyệt (`US-PAY-02`), duyệt/từ chối (`US-PAY-03`), hủy bảng nháp (`US-PAY-05`). |
| **`PaymentStatementAppService`** | Application | Điều phối luồng nghiệp vụ, gọi các Read Ports đọc Hợp đồng/Bảng giá/Sản lượng, thực thi validation (`PAY-01`..`PAY-08`), quản lý Transaction, ghi `PaymentAuditLog` và gửi thông báo qua `INotificationService`. |
| **`PaymentStatement` & `PaymentLine`** | Domain | Đóng gói dữ liệu bảng thanh toán, dòng thanh toán (`Thành tiền = Sản lượng × Đơn giá` theo `PAY-03`), quy tắc chuyển trạng thái, kiểm tra `PAY-04`, `PAY-05`, `PAY-07`. |
| **`IContractReadService`, `IPriceListReadService`, `IVolumeReadService`** | Application Ports (Read-Only) | Giao diện chỉ đọc dữ liệu từ các module Hợp đồng (trạng thái `Active`, kỳ thanh toán, dịch vụ bắt buộc), Bảng giá (đơn giá hiệu lực), Sản lượng (trạng thái `Đã đối soát`). |
| **`IPaymentStatementRepository` & `IPaymentAuditLogRepository`** | Infrastructure (Persistence) | Lưu trữ `PaymentStatement`, `PaymentLine`, `PaymentAuditLog` vào PostgreSQL; đảm bảo Unique Constraint và Optimistic Locking. |
| **`INotificationService`** | Application Port | Gửi thông báo cho người duyệt (`KTT` hoặc `GĐKD`) khi NVKD trình duyệt bảng thanh toán (`US-PAY-02`). |

### 2.3. Integration Points
1. **Inbound**: REST Client / UI gọi vào `PaymentStatementsController`.
2. **Outbound (Read-only sang các module nội bộ)**:
   - Đọc thông tin Hợp đồng & danh sách dịch vụ bắt buộc qua `IContractReadService`.
   - Đọc Bảng giá hiệu lực qua `IPriceListReadService`.
   - Đọc Sản lượng thực tế (`Reconciled`) trong kỳ qua `IVolumeReadService`.
3. **Outbound (Side-effects)**:
   - Gửi thông báo cho người duyệt qua `INotificationService`.

### 2.4. Security Considerations
- **Authentication**: Xác thực qua JWT Bearer Token (chứa `UserId` và `Role`: `NVKD`, `KTT`, `GDKD`).
- **Authorization**:
  - `US-PAY-01`, `US-PAY-02`, `US-PAY-05`: Yêu cầu role `NVKD`.
  - `US-PAY-03`: Yêu cầu role `KTT` hoặc `GDKD` đúng theo ngưỡng 500 triệu đồng (`PAY-06` / `US-PAY-03`) **VÀ** kiểm tra `ApproverUserId != CreatedByUserId` (`PAY-07`).

### 2.5. Transaction Boundaries, Concurrency, Idempotency & Performance
- **Transaction Boundary**:
  - Mỗi thao tác (`Create`, `UpdateDraft`, `Submit`, `Approve`, `Reject`, `Cancel`) thực thi trong một Database Transaction bao gồm cả việc cập nhật `PaymentStatement` và chèn bản ghi `PaymentAuditLog` (`NFR-01`).
- **Chống thao tác trình duyệt trùng (`PAY-04`) & Concurrency**:
  - Hỗ trợ `Idempotency-Key` trên request trình duyệt kết hợp **Optimistic Concurrency Control (`RowVersion` / PostgreSQL `xmin`)** trên bảng `payment_statements`.
  - Khi hai request `Submit` (hoặc `Approve`) gửi tới cùng lúc, chỉ 1 request cập nhật thành công từ `Draft` $\rightarrow$ `PendingApproval`, request thứ 2 bị chặn (`409 Conflict` hoặc trả kết quả idempotent).
- **Đáp ứng Hiệu năng `< 3 giây` với `< 200 dòng` (`NFR-02`, Trang 3, Mục 6)**:
  - Khi lập bảng (`US-PAY-01`), đọc dữ liệu Sản lượng và Bảng giá theo **Batch Query** (1 lần truy vấn cho toàn bộ dịch vụ của hợp đồng trong kỳ, tuyệt đối không truy vấn N+1 theo từng dòng), tính toán trong bộ nhớ và Bulk Insert `< 200` dòng `PaymentLine` trong cùng 1 lệnh `SaveChangesAsync()`. Thời gian xử lý ước tính `< 150ms` (đạt yêu cầu `< 3000ms`).

