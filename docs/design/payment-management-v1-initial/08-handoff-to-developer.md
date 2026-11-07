# 08. Handoff to Developer — `payment-management`

> **Trạng thái thiết kế**: `Design Pending Approval`

---

## 1. Phạm vi Chức năng Bàn giao Triển khai Ngay
Developer Agent có đầy đủ thông tin thiết kế để triển khai ngay các chức năng không mâu thuẫn:
1. **`US-PAY-01` (Lập bảng thanh toán)**: Tạo bảng thanh toán ở trạng thái `Draft` từ dữ liệu đọc qua `IContractReadService`, `IPriceListReadService`, `IVolumeReadService`, tự tính `LineAmount = Quantity * UnitPrice` (`PAY-03`) và `TotalAmount`, đảm bảo phản hồi `< 3 giây` với `< 200 dòng` (`NFR-02`).
2. **`US-PAY-02` (Trình duyệt bảng thanh toán)**: Kiểm tra `PAY-04` (tổng tiền không âm, đủ dòng dịch vụ bắt buộc, chống trình duyệt trùng bằng `xmin` + `Idempotency-Key`), chuyển trạng thái `Draft -> PendingApproval` và gửi thông báo.
3. **`US-PAY-05` (Hủy bảng nháp)**: Chuyển trạng thái `Draft -> Cancelled` và khóa mọi thao tác tiếp theo trên bảng đã hủy.
4. **Hệ thống `PaymentAuditLog` (`NFR-01`)**: Ghi log mọi thay đổi trạng thái trong cùng Database Transaction.

---

## 2. Danh sách Thành phần Cần Tạo Mới (`src/Qtkd.Api/Modules/PaymentManagement/`)
- `Domain/`:
  - `PaymentStatus.cs`, `UserRole.cs`, `BillingPeriod.cs`
  - `PaymentStatement.cs`, `PaymentLine.cs`, `PaymentAuditLog.cs`
  - `IPaymentApprovalPolicy.cs`
- `Application/`:
  - `Ports/`: `IContractReadService.cs`, `IPriceListReadService.cs`, `IVolumeReadService.cs`, `INotificationService.cs`, `IPaymentStatementRepository.cs`, `IPaymentAuditLogRepository.cs`
  - `Services/`: `PaymentStatementAppService.cs`
  - `Dtos/`: `CreatePaymentStatementRequest.cs`, `SubmitPaymentStatementRequest.cs`, `PaymentStatementResponse.cs`
- `Infrastructure/`:
  - `Persistence/`: EF Core Configurations cho `payment_statements`, `payment_lines`, `payment_audit_logs`.
- `Presentation/`:
  - `PaymentStatementsController.cs`

---

## 3. Những Vấn đề Còn Tồn tại Cần Xác nhận Trước Khi Chốt Toàn Bộ Module (`Open Issues`)
Developer **KHÔNG** tự suy diễn các điểm sau mà phải chờ Reviewer xác nhận:
1. **`ISS-01` (`Mục 3` vs `PAY-08`)**: Khi từ chối chuyển sang `Rejected` hay về thẳng `Draft`?
2. **`ISS-02` (`US-PAY-03` vs `PAY-06`)**: Tổng tiền đúng bằng `500.000.000 VNĐ` do `GĐKD` (`>= 500tr`) hay `KTT` (`> 500tr`) duyệt?
3. **`ISS-03` (`US-PAY-04` vs `PAY-05`)**: Xử lý mâu thuẫn giữa việc cấm sửa bảng `Approved` dưới mọi hình thức (`PAY-05`) và cho phép KTT điều chỉnh khi có biên bản (`US-PAY-04`).
4. **`ISS-04` & `ISS-05` (`Mục 2`, `US-PAY-05` vs `PAY-01`)**: Ràng buộc 1 bảng/hợp đồng tính theo Kỳ thanh toán (Tháng/Quý) và loại trừ các bảng đã `Cancelled`.
5. **`ISS-06` (`US-PAY-01`)**: Phạm vi các trường NVKD được phép sửa khi bảng ở trạng thái `Draft`.
6. **`ISS-07` (`PAY-04`)**: Bảng có `Tổng tiền = 0` có được trình duyệt không?

---

## 4. Tiêu chí Hoàn thành (Definition of Done)
- Tất cả Unit Tests cho `PAY-01`, `PAY-02`, `PAY-03`, `PAY-04`, `PAY-05`, `PAY-07` đều Pass.
- Concurrency Test cho `Submit` (`PAY-04`) đảm bảo không trình duyệt trùng.
- Performance Test lập bảng 200 dòng hoàn thành `< 3 giây` (`NFR-02`).
- Mọi thao tác chuyển trạng thái đều sinh bản ghi trong `payment_audit_logs` (`NFR-01`).
- Các điểm `ISS-01`..`ISS-08` được Reviewer phê duyệt phương án xử lý.

