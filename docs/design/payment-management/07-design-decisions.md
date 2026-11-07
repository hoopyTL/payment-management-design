# 07. Design Decisions (ADR) — `payment-management`

> **Trạng thái thiết kế**: `Design Pending Approval`

---

## ADR-01: Giao tiếp Chỉ đọc (Read-Only Ports) với Module Hợp đồng, Bảng giá và Sản lượng
- **Context**: Đặc tả (`BAITAP_TA_1-2.pdf`, Trang 1, Mục 1) quy định: *"Quản lý hợp đồng, bảng giá và sản lượng là module khác; module này chỉ đọc dữ liệu từ chúng."*
- **Problem**: Làm thế nào để thiết kế `payment-management` độc lập, dễ kiểm thử mà không bị phụ thuộc chặt vào cấu trúc bảng nội bộ của các module khác?
- **Options Considered**:
  1. Truy vấn trực tiếp bảng DB của các module Hợp đồng, Bảng giá, Sản lượng trong `PaymentStatementAppService`.
  2. Định nghĩa các Read-Only Interfaces (`IContractReadService`, `IPriceListReadService`, `IVolumeReadService`) trả về DTO chỉ đọc.
- **Decision**: Chọn **Phương án 2 (Read-Only Ports)**.
- **Rationale**: Tuân thủ đúng ranh giới module trong đặc tả, giúp viết Unit Test cho `US-PAY-01` và `US-PAY-02` dễ dàng bằng cách mock 3 interfaces này, đồng thời đảm bảo module Bảng thanh toán không bao giờ vô tình ghi/sửa dữ liệu của module khác.
- **Consequences & Risks**: Cần cài đặt Adapter thực tế khi tích hợp với các module còn lại.

---

## ADR-02: Giải pháp Chống thao tác trình duyệt trùng (`PAY-04`)
- **Context**: `PAY-04` (Trang 2, Mục 5) yêu cầu: *"Hệ thống chống thao tác trình duyệt trùng."*
- **Problem**: Khi NVKD bấm nút "Trình duyệt" nhiều lần liên tiếp (Double-click / Network retry), hệ thống có nguy cơ chuyển trạng thái nhiều lần, ghi nhiều dòng Audit Log trùng và gửi nhiều thông báo trùng cho KTT/GĐKD.
- **Options Considered**:
  1. Chỉ kiểm tra `if (statement.Status != Draft)` trong code ứng dụng (dễ bị Race Condition khi 2 request vào song song).
  2. Kết hợp kiểm tra trạng thái `Status == Draft` trong Domain + Optimistic Concurrency (`RowVersion` / PostgreSQL `xmin`) + `Idempotency-Key` trên API.
- **Decision**: Chọn **Phương án 2**.
- **Rationale**: Đảm bảo an toàn tuyệt đối cả khi gửi tuần tự lẫn khi 2 request đến đồng thời (PostgreSQL sẽ từ chối lệnh `UPDATE ... WHERE id = @id AND xmin = @rowVersion` thứ hai).

---

## ADR-03: Cô lập các Điểm Mâu thuẫn trong Đặc tả (`ISS-01`..`ISS-04`) không để AI tự suy diễn
- **Context**: Đặc tả `BAITAP_TA_1-2.pdf` chứa các mâu thuẫn có chủ đích để đánh giá khả năng phát hiện vấn đề (chiếm 30% tổng điểm bài tập):
  - `ISS-01`: Khi từ chối chuyển sang `Rejected` (Mục 3) hay `Draft` (`PAY-08`)?
  - `ISS-02`: Tổng tiền `= 500.000.000đ` do `GĐKD` duyệt (`US-PAY-03`: `>= 500tr`) hay `KTT` duyệt (`PAY-06`: `> 500tr`)?
  - `ISS-03`: Bảng `Approved` được KTT điều chỉnh khi có biên bản (`US-PAY-04`) hay cấm sửa dưới mọi hình thức (`PAY-05`)?
  - `ISS-04`: Kỳ thanh toán là tháng/quý (Mục 2) hay mỗi tháng 1 bảng (`PAY-01`)?
- **Decision**:
  - **Không tự ý chọn ngầm trong code/sơ đồ**.
  - Tách logic kiểm tra ngưỡng 500 triệu và trạng thái từ chối vào interface chính sách `IPaymentApprovalPolicy` để sau khi Reviewer xác nhận chỉ cần cấu hình 1 chỗ duy nhất.
  - Giữ trạng thái thiết kế của `US-PAY-04` là `Blocked / Pending Clarification (ISS-03)` cho đến khi có xác nhận chính thức từ người phụ trách.

