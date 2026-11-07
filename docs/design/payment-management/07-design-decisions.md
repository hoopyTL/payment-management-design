# 07. Design Decisions (ADR) — `payment-management`

> **Trạng thái thiết kế**: `Approved by Reviewer`

---

## ADR-01: Giải quyết Mâu thuẫn giữa `US-PAY-04` (Điều chỉnh bảng đã duyệt) và `PAY-05` (Không được sửa bảng `Approved`) — `ISS-03`
- **Context**: `US-PAY-04` (Trang 2, Mục 4.4) cho phép KTT điều chỉnh bảng đã `Approved` nếu có biên bản xác nhận đính kèm, nhưng `PAY-05` (Trang 2, Mục 5) quy định *"Bảng thanh toán đã Approved không được sửa dưới bất kỳ hình thức nào"*.
- **Decision (Nhóm đã chốt)**:
  - Giữ bảng thanh toán gốc (`PaymentStatement`) đã `Approved` ở trạng thái **bất biến (Immutable)** để tuân thủ nghiêm ngặt `PAY-05`.
  - Khi KTT thực hiện điều chỉnh (kèm biên bản xác nhận đính kèm), hệ thống **tạo bản ghi điều chỉnh riêng (`PaymentAdjustment`)** tham chiếu tới bảng gốc.
  - Nếu tổng tiền sau điều chỉnh **từ 500 triệu đồng trở lên (`>= 500.000.000`)**, bản ghi điều chỉnh chuyển sang trạng thái `PendingGdkdApproval` để **GĐKD duyệt lại**; nếu `< 500.000.000` thì chuyển thẳng sang `Approved`.
- **Consequences**: Đáp ứng đồng thời cả `PAY-05`, `US-PAY-04`, `PAY-06` và giữ nguyên lịch sử kế toán minh bạch.

---

## ADR-02: Giải quyết Mâu thuẫn Luồng Trạng thái khi Từ chối giữa `Mục 3` và `PAY-08` — `ISS-01`
- **Context**: `Mục 3` (Trang 1) ghi khi bị từ chối bảng chuyển sang `Rejected`, còn `PAY-08` (Trang 2) ghi sau khi bị từ chối bảng quay về `Draft` để NVKD chỉnh sửa và trình lại.
- **Decision (Nhóm đã chốt)**:
  - Khi KTT/GĐKD bấm Từ chối (kèm lý do bắt buộc), bảng chuyển từ `PendingApproval` $\rightarrow$ `Rejected`.
  - Khi NVKD mở bảng `Rejected` và chọn **"Chỉnh sửa lại"** (`revise`), bảng chuyển từ `Rejected` $\rightarrow$ `Draft` để NVKD đồng bộ/sửa ghi chú và trình duyệt lại.
- **Consequences**: Bảo toàn đầy đủ 5 trạng thái của `Mục 3` (`Draft`, `PendingApproval`, `Approved`, `Rejected`, `Cancelled`) và đáp ứng trọn vẹn quy trình nghiệp vụ của `PAY-08`.

---

## ADR-03: Chốt Điều kiện Biên `500.000.000 VNĐ` (`ISS-02`) và Chống Tự duyệt khi Người lập kiêm KTT (`ISS-08`)
- **Context**: `US-PAY-03` ghi *"từ 500 triệu đồng trở lên"* (`>= 500tr`) do GĐKD duyệt, trong khi `PAY-06` ghi *"trên 500 triệu đồng"* (`> 500tr`). Ngoài ra `PAY-07` cấm người lập tự duyệt bảng của mình.
- **Decision (Nhóm đã chốt)**:
  - Chốt điều kiện biên: **`TotalAmount >= 500.000.000 VNĐ` do GĐKD duyệt**; `< 500.000.000 VNĐ` do KTT duyệt.
  - Nếu bảng `< 500.000.000 VNĐ` mà người lập bảng đồng thời là KTT, hệ thống chặn người đó tự duyệt (`PAY-07`) và định tuyến sang **KTT khác** duyệt.

---

## ADR-04: Ràng buộc Duy nhất theo Kỳ Tháng/Quý (`ISS-04`, `ISS-05`) & Phạm vi Chỉnh sửa ở `Draft` (`ISS-06`, `ISS-07`)
- **Decision (Nhóm đã chốt)**:
  1. **Kỳ thanh toán & Bảng đã hủy (`ISS-04`, `ISS-05`)**: Mỗi hợp đồng có tối đa 1 bảng thanh toán chưa bị hủy (`Status != Cancelled`) cho mỗi kỳ tháng hoặc quý theo cấu hình hợp đồng (cài đặt bằng PostgreSQL Partial Unique Index `WHERE status <> 'Cancelled'`).
  2. **Phạm vi sửa ở `Draft` (`ISS-06`)**: NVKD chỉ được sửa ghi chú/mô tả hoặc bấm đồng bộ & tính lại từ dữ liệu nguồn (không sửa trực tiếp số lượng, đơn giá hay thêm/bớt dòng).
  3. **Điều kiện trình duyệt (`ISS-07`)**: Chặn trình duyệt khi `TotalAmount <= 0` và kiểm tra đủ các dịch vụ trong `MandatoryServiceIds`.
  4. **Quyền hủy bảng nháp (`ISS-08`)**: Chỉ NVKD đã lập bảng (`ActorId == CreatedBy`) mới được hủy bản nháp đó.
