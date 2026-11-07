# 02. Architecture Design (AS-IS & TO-BE) — `payment-management`

> **Trạng thái thiết kế**: `Approved by Reviewer`  
> **Sơ đồ kiến trúc thành phần**: `diagrams/component.puml`

---

## 1. Khảo sát Kiến trúc Hiện tại (AS-IS Analysis)

Theo quy định tại `.cursor/agents/design.md` (Step 2):
- **Mã nguồn `src/Qtkd.Api/`**: Chưa tồn tại trong workspace hiện tại (dự án khởi tạo mới — Greenfield).
- **Phân loại thành phần**:
  - **Thành phần đã tồn tại**: Không có.
  - **Thành phần cần thay đổi**: Không có.
  - **Thành phần cần bổ sung mới**: Module `PaymentManagement` trong `src/Qtkd.Api`.

---

## 2. Kiến trúc Giải pháp Đề xuất (TO-BE High-Level Design)

### 2.1. Architecture Overview & Architectural Style
- **Kiến trúc lựa chọn**: **Modular Monolith kết hợp Clean / Layered Architecture** trên nền tảng **ASP.NET Core Web API (C#)** và **PostgreSQL**.
- **Lý do lựa chọn**:
  - Đáp ứng đúng yêu cầu phạm vi tại `BAITAP_TA_1-2.pdf` (Trang 1, Mục 1): *"Quản lý hợp đồng, bảng giá và sản lượng là module khác; module này chỉ đọc dữ liệu từ chúng."*
  - Sử dụng các **Read-Only Ports** (`IContractReadService`, `IPriceListReadService`, `IVolumeReadService`) để tách biệt hoàn toàn module Bảng thanh toán với các module nguồn.
  - Đơn giản, dễ kiểm thử, không lạm dụng Microservices/CQRS phức tạp theo đúng nguyên tắc tại `.cursor/agents/design.md` (Mục 8).

### 2.2. Component Responsibilities (Trách nhiệm các thành phần)

| Thành phần | Layer | Trách nhiệm chính |
| :--- | :--- | :--- |
| **`PaymentStatementsController`** | Presentation (API) | Tiếp nhận HTTP Requests cho lập bảng (`US-PAY-01`), cập nhật/đồng bộ bảng nháp (`US-PAY-01`), trình duyệt (`US-PAY-02`), duyệt/từ chối/chỉnh sửa lại (`US-PAY-03`), tạo & duyệt bản ghi điều chỉnh (`US-PAY-04`), hủy bảng nháp (`US-PAY-05`). |
| **`PaymentStatementAppService`** | Application | Điều phối use case, gọi các Read Ports đọc Hợp đồng/Bảng giá/Sản lượng, quản lý Transaction, ghi `PaymentAuditLog` và gửi thông báo qua `INotificationService`. |
| **`PaymentStatement`, `PaymentLine`, `PaymentAdjustment`** | Domain | Đóng gói dữ liệu và toàn bộ quy tắc nghiệp vụ `PAY-01`..`PAY-08` cùng 8 quyết định chốt `ISS-01`..`ISS-08` (bất biến bảng `Approved`, tạo bản ghi điều chỉnh riêng, kiểm tra ngưỡng `>= 500.000.000`, kiểm tra `TotalAmount > 0`, `ApproverId != CreatedBy`). |
| **`IContractReadService`, `IPriceListReadService`, `IVolumeReadService`** | Application Ports (Read-Only) | Chỉ đọc dữ liệu Hợp đồng (`Active`, loại kỳ thanh toán tháng/quý, `MandatoryServiceIds`), Bảng giá hiệu lực, và Sản lượng kỳ (`Đã đối soát`). |
| **`IPaymentStatementRepository`, `IPaymentAdjustmentRepository`, `IPaymentAuditLogRepository`** | Infrastructure (Persistence) | Lưu trữ vào PostgreSQL, đảm bảo Unique Partial Index theo kỳ (`status <> 'Cancelled'`) và Optimistic Locking (`xmin`). |
| **`INotificationService`** | Application Port | Gửi thông báo cho `GĐKD` (khi `>= 500tr`) hoặc `KTT` khác người lập (khi `< 500tr`). |

### 2.3. Security & Contextual Authorization
1. **Lập bảng, Sửa/Đồng bộ nháp, Trình duyệt (`US-PAY-01`, `US-PAY-02`)**: Yêu cầu role `NVKD`.
2. **Hủy bảng nháp (`US-PAY-05` & `ISS-08`)**: Yêu cầu role `NVKD` **VÀ** `CurrentUserId == statement.CreatedBy` (chỉ NVKD đã lập bảng mới được hủy).
3. **Duyệt / Từ chối (`US-PAY-03`, `PAY-06`, `PAY-07`, `ISS-02`, `ISS-08`)**:
   - Bắt buộc `CurrentUserId != statement.CreatedBy` (`PAY-07`). Nếu người lập cũng có role `KTT`, hệ thống chặn tự duyệt và yêu cầu một `KTT` khác duyệt (`ISS-08`).
   - Nếu `TotalAmount >= 500_000_000`: bắt buộc role `GDKD` (`ISS-02`).
   - Nếu `TotalAmount < 500_000_000`: bắt buộc role `KTT` (`ISS-02`).
4. **Điều chỉnh bảng đã Approved (`US-PAY-04`, `PAY-05`, `ISS-03`)**:
   - Chỉ `KTT` được tạo bản ghi `PaymentAdjustment` kèm biên bản xác nhận.
   - Nếu `AdjustedTotalAmount >= 500_000_000`, bắt buộc `GDKD` phê duyệt bản ghi điều chỉnh.

### 2.4. Transaction Boundaries, Concurrency, Idempotency & Performance
- **Transaction Boundary**: Mọi thao tác làm thay đổi trạng thái hoặc tạo bản ghi điều chỉnh đều thực thi trong **1 Database Transaction** cùng với việc ghi `PaymentAuditLog` (`NFR-01`).
- **Idempotency & Concurrency (`PAY-04`)**:
  - Sử dụng `Idempotency-Key` kết hợp PostgreSQL `xmin` (Optimistic Concurrency) để chống thao tác trình duyệt trùng hoặc duyệt trùng.
  - Sử dụng Unique Partial Index `ON payment_statements(contract_id, period_code) WHERE status <> 'Cancelled'` để đảm bảo mỗi hợp đồng chỉ có tối đa 1 bảng thanh toán đang hoạt động cho mỗi kỳ tháng/quý (`ISS-04`, `ISS-05`).
- **Performance (`NFR-02`)**:
  - Truy vấn batch 1 lần cho toàn bộ sản lượng và đơn giá trong kỳ, tính toán trên bộ nhớ và bulk insert `< 200 dòng` trong 1 round-trip DB, đảm bảo thời gian phản hồi `< 3 giây`.
