# 03. Detailed Design (Low-Level Design) — `payment-management`

> **Trạng thái thiết kế**: `Design Pending Approval`  
> **Sơ đồ UML đính kèm**:
> - Class Diagram: `diagrams/class.puml` *(Trạng thái: Đã kiểm tra kết xuất sơ đồ thành công)*
> - Sequence Diagram: `diagrams/sequence.puml` *(Trạng thái: Đã kiểm tra kết xuất sơ đồ thành công)*

---

## 1. Thiết kế Class và Interface (Class & Interface Design)

### 1.1. Enums & Value Objects
- **`PaymentStatus` (Enum - Mục 3, Trang 1)**:
  - `Draft = 1`
  - `PendingApproval = 2`
  - `Approved = 3`
  - `Rejected = 4` *(Lưu ý `ISS-01`: Mục 3 nêu chuyển sang `Rejected`, trong khi `PAY-08` nêu quay về `Draft` — xem cấu hình xử lý bên dưới)*
  - `Cancelled = 5`
- **`UserRole` (Enum - Mục 2, Trang 1)**:
  - `NVKD` (Nhân viên kinh doanh)
  - `KTT` (Kế toán trưởng)
  - `GDKD` (Giám đốc kinh doanh)
- **`BillingPeriod` (Value Object)**:
  - `PeriodCode: string` (Ví dụ: `"2026-11"` cho tháng hoặc `"2026-Q4"` cho quý — xem `ISS-04`)
  - `StartDate: DateOnly`
  - `EndDate: DateOnly`

### 1.2. Domain Entities (`PaymentManagement` Module)

#### `PaymentStatement` (Aggregate Root)
- **Attributes**:
  - `Id: Guid`
  - `StatementCode: string`
  - `ContractId: Guid`
  - `PeriodCode: string`
  - `PeriodStartDate: DateOnly`
  - `PeriodEndDate: DateOnly`
  - `TotalAmount: decimal`
  - `Status: PaymentStatus`
  - `Note: string?`
  - `RejectionReason: string?`
  - `CreatedBy: Guid`
  - `CreatedAt: DateTimeOffset`
  - `SubmittedAt: DateTimeOffset?`
  - `ApprovedBy: Guid?`
  - `ApprovedAt: DateTimeOffset?`
  - `RowVersion: uint` (Optimistic Concurrency Token)
  - `Lines: IReadOnlyCollection<PaymentLine>`
- **Operations (Methods)**:
  - `static CreateDraft(Guid contractId, BillingPeriod period, IEnumerable<PaymentLineInput> lineInputs, Guid createdBy, string? note)`:
    - Khởi tạo bảng ở trạng thái `Draft`, tạo các `PaymentLine` với `Amount = Quantity * UnitPrice` (`PAY-03`), tính `TotalAmount = sum(Lines.Amount)`.
  - `UpdateDraft(string? note, IEnumerable<PaymentLineInput>? updatedLines)`:
    - Kiểm tra `Status == PaymentStatus.Draft` (`US-PAY-01`, `PAY-05`). Nếu khác `Draft`, ném `DomainException("ERR_NOT_IN_DRAFT")`.
    - *(Lưu ý `ISS-06`: Việc cho phép sửa `updatedLines` hay chỉ sửa `note`/đồng bộ lại phụ thuộc xác nhận của Reviewer).*
  - `Submit(IReadOnlyCollection<Guid> mandatoryServiceIds)`:
    - Kiểm tra `Status == PaymentStatus.Draft` (`PAY-04` chống trình duyệt trùng ở tầng domain).
    - Kiểm tra `TotalAmount >= 0` (`PAY-04`: chặn nếu tổng tiền âm; xem thêm `ISS-07` về trường hợp `= 0`).
    - Kiểm tra mọi `serviceId` trong `mandatoryServiceIds` đều có mặt trong `Lines` (`PAY-04`: chặn nếu thiếu dòng dịch vụ bắt buộc).
    - Chuyển `Status = PaymentStatus.PendingApproval` và gán `SubmittedAt = UtcNow`.
  - `Cancel()`:
    - Kiểm tra `Status == PaymentStatus.Draft` (`US-PAY-05`).
    - Chuyển `Status = PaymentStatus.Cancelled` (sau khi hủy không dùng lại được).
  - `Approve(Guid approverId, UserRole approverRole, bool isInclusive500MThreshold)`:
    - Kiểm tra `Status == PaymentStatus.PendingApproval`.
    - Kiểm tra `approverId != CreatedBy` (`PAY-07`: người lập không được duyệt bảng do chính mình lập).
    - Kiểm tra thẩm quyền theo ngưỡng 500.000.000 VNĐ (`US-PAY-03` vs `PAY-06` - tách biến `isInclusive500MThreshold` để chờ chốt `ISS-02`).
    - Chuyển `Status = PaymentStatus.Approved`, gán `ApprovedBy = approverId`, `ApprovedAt = UtcNow`.
  - `Reject(Guid approverId, UserRole approverRole, string reason, bool transitionDirectlyToDraft)`:
    - Kiểm tra `Status == PaymentStatus.PendingApproval`, kiểm tra `approverId != CreatedBy` (`PAY-07`) và thẩm quyền duyệt.
    - Kiểm tra `!string.IsNullOrWhiteSpace(reason)` (`PAY-08`).
    - Lưu `RejectionReason = reason` và chuyển trạng thái theo cấu hình chốt `ISS-01` (`Rejected` theo Mục 3 hoặc `Draft` theo `PAY-08`).

#### `PaymentLine` (Entity thuộc `PaymentStatement`)
- **Attributes**:
  - `Id: Guid`
  - `PaymentStatementId: Guid`
  - `ServiceId: Guid`
  - `ServiceCode: string`
  - `ServiceName: string`
  - `Quantity: decimal` (Sản lượng thực tế đã đối soát)
  - `UnitPrice: decimal` (Snapshot đơn giá tại thời điểm tính - `PAY-03`)
  - `LineAmount: decimal` (`= Quantity * UnitPrice` - `PAY-03`)

#### `PaymentAuditLog` (Entity - Mục 6)
- **Attributes**:
  - `Id: Guid`
  - `PaymentStatementId: Guid`
  - `ActorId: Guid`
  - `OccurredAt: DateTimeOffset`
  - `Action: string` (`Created`, `UpdatedDraft`, `Submitted`, `Approved`, `Rejected`, `Cancelled`)
  - `OldStatus: PaymentStatus?`
  - `NewStatus: PaymentStatus`
  - `ChangeDetailsJson: string` (Lưu chi tiết thay đổi gì, lý do từ chối, snapshot tổng tiền/dòng thanh toán)

---

## 2. Read-Only Ports (Giao tiếp với các Module khác theo Mục 1)

Vì Mục 1 quy định *"Quản lý hợp đồng, bảng giá và sản lượng là module khác; module này chỉ đọc dữ liệu từ chúng"*, thiết kế định nghĩa 3 Read-Only Interfaces:

1. **`IContractReadService`**:
   - `GetContractSummaryAsync(Guid contractId, CancellationToken ct)` $\rightarrow$ trả về `ContractReadDto` (`ContractId`, `Status`, `BillingCycleType`, `MandatoryServiceIds`).
2. **`IPriceListReadService`**:
   - `GetApplicablePriceListAsync(Guid contractId, BillingPeriod period, CancellationToken ct)` $\rightarrow$ trả về `PriceListReadDto` (`PriceListId`, `UnitPricesByServiceId`).
3. **`IVolumeReadService`**:
   - `GetPeriodVolumeAsync(Guid contractId, BillingPeriod period, CancellationToken ct)` $\rightarrow$ trả về `VolumeReadDto` (`IsReconciled`, `Items: IReadOnlyList<VolumeItemDto>`).

---

## 3. Luồng tương tác chính (Sequence Flows)

Chi tiết được biểu diễn trong `diagrams/sequence.puml`, bao gồm:
1. **Luồng `US-PAY-01` (Lập bảng thanh toán)**:
   - `NVKD` gửi yêu cầu tạo bảng thanh toán với `contractId` và `periodCode`.
   - `PaymentStatementAppService` kiểm tra chưa tồn tại bảng thanh toán cho hợp đồng trong kỳ/tháng (`PAY-01`), đọc thông tin Hợp đồng (`Active` - `PAY-01`), đọc Sản lượng kỳ (`IsReconciled == true` - `PAY-02`), đọc Bảng giá hiệu lực (`PAY-01`).
   - Tự động tính `PaymentLine` (`LineAmount = Quantity * UnitPrice` - `PAY-03`) và `TotalAmount`, lưu trạng thái `Draft` và ghi `PaymentAuditLog` (`NFR-01`) trong `< 3 giây` (`NFR-02`).
2. **Luồng `US-PAY-02` (Trình duyệt bảng thanh toán)**:
   - `NVKD` gọi API trình duyệt kèm `Idempotency-Key`.
   - `PaymentStatementAppService` kiểm tra chống trình duyệt trùng (`PAY-04`), đọc danh sách `MandatoryServiceIds` từ `IContractReadService`, gọi `statement.Submit(mandatoryServiceIds)` để kiểm tra `TotalAmount >= 0` và đủ dịch vụ bắt buộc (`PAY-04`).
   - Cập nhật trạng thái sang `PendingApproval`, ghi `PaymentAuditLog`, xác định người duyệt (`KTT` hoặc `GĐKD`) và gửi thông báo qua `INotificationService`.

