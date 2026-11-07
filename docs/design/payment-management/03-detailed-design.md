# 03. Detailed Design (Low-Level Design) — `payment-management`

> **Trạng thái thiết kế**: `Approved by Reviewer`  
> **Các sơ đồ UML đính kèm**:
> - Component Diagram: `diagrams/component.puml` *(Đã kiểm tra kết xuất thành công)*
> - Class Diagram: `diagrams/class.puml` *(Đã kiểm tra kết xuất thành công)*
> - Sequence Diagram: `diagrams/sequence.puml` *(Đã kiểm tra kết xuất thành công)*
> - State Machine Diagram: `diagrams/state-machine.puml` *(Đã kiểm tra kết xuất thành công)*

---

## 1. Thiết kế Class và Interface (Class & Interface Design)

### 1.1. Enums & Value Objects
- **`PaymentStatus` (Enum - Mục 3 & `ISS-01`)**:
  - `Draft = 1`
  - `PendingApproval = 2`
  - `Approved = 3`
  - `Rejected = 4`
  - `Cancelled = 5`
- **`AdjustmentStatus` (Enum - `US-PAY-04` & `ISS-03`)**:
  - `PendingGdkdApproval = 1` *(Khi `AdjustedTotalAmount >= 500_000_000`)*
  - `Approved = 2` *(Khi `AdjustedTotalAmount < 500_000_000` hoặc sau khi GĐKD duyệt)*
  - `Rejected = 3`
- **`PeriodType` (Enum - Mục 2 & `ISS-04`)**:
  - `Monthly = 1`
  - `Quarterly = 2`
- **`UserRole` (Enum - Mục 2)**:
  - `NVKD = 1`, `KTT = 2`, `GDKD = 3`
- **`BillingPeriod` (Value Object)**:
  - `PeriodCode: string` (VD: `"2026-11"` hoặc `"2026-Q4"`), `PeriodType: PeriodType`, `StartDate: DateOnly`, `EndDate: DateOnly`.

---

### 1.2. Domain Entities

#### `PaymentStatement` (Aggregate Root)
- **Attributes**:
  - `Id: Guid`, `StatementCode: string`, `ContractId: Guid`, `Period: BillingPeriod`, `TotalAmount: decimal`, `Status: PaymentStatus`, `Note: string?`, `RejectionReason: string?`, `CreatedBy: Guid`, `CreatedAt: DateTimeOffset`, `SubmittedAt: DateTimeOffset?`, `ApprovedBy: Guid?`, `ApprovedAt: DateTimeOffset?`, `RowVersion: uint`.
  - `Lines: IReadOnlyCollection<PaymentLine>`
  - `Adjustments: IReadOnlyCollection<PaymentAdjustment>`
- **Constants**:
  - `GdkdApprovalThreshold = 500_000_000m` (`ISS-02`: từ 500 triệu đồng trở lên do GĐKD duyệt).
- **Methods**:
  1. `static CreateDraft(Guid contractId, BillingPeriod period, IReadOnlyList<PaymentLineInput> lineInputs, Guid nvkdUserId, string? note)`:
     - Sinh các `PaymentLine` với `LineAmount = Math.Round(Quantity * UnitPrice, 2)` (`PAY-03`), tính `TotalAmount`, gán `Status = PaymentStatus.Draft`.
  2. `UpdateDraft(Guid actorId, string? note, IReadOnlyList<PaymentLineInput>? resyncedLineInputs)`:
     - Kiểm tra `Status == PaymentStatus.Draft` (`US-PAY-01`, `PAY-05`).
     - Theo `ISS-06`: Chỉ cập nhật `Note` và (nếu `resyncedLineInputs != null`) thay thế toàn bộ `Lines` bằng dữ liệu vừa đồng bộ lại từ module Sản lượng & Bảng giá, tính lại `TotalAmount`. Tuyệt đối không có phương thức sửa lẻ số lượng/đơn giá thủ công.
  3. `Submit(Guid actorId, IReadOnlyCollection<Guid> mandatoryServiceIds)`:
     - Kiểm tra `Status == PaymentStatus.Draft` (`PAY-04` chống trình duyệt trùng).
     - Kiểm tra `TotalAmount > 0` (`PAY-04` & `ISS-07`: chặn khi `TotalAmount <= 0`).
     - Kiểm tra mọi phần tử trong `mandatoryServiceIds` đều tồn tại trong `Lines.Select(x => x.ServiceId)` (`PAY-04` & `ISS-07`).
     - Chuyển `Status = PaymentStatus.PendingApproval`, `SubmittedAt = UtcNow`.
  4. `GetRequiredApproverRole()`:
     - Trả về `UserRole.GDKD` nếu `TotalAmount >= 500_000_000m`, ngược lại trả về `UserRole.KTT` (`ISS-02`).
  5. `Approve(Guid approverId, UserRole approverRole)`:
     - Kiểm tra `Status == PaymentStatus.PendingApproval`.
     - Kiểm tra `approverId != CreatedBy` (`PAY-07` & `ISS-08`: người lập không được tự duyệt; nếu người lập là KTT thì KTT khác phải duyệt).
     - Kiểm tra `approverRole == GetRequiredApproverRole()` (`PAY-06` & `ISS-02`).
     - Chuyển `Status = PaymentStatus.Approved`, `ApprovedBy = approverId`, `ApprovedAt = UtcNow`.
  6. `Reject(Guid approverId, UserRole approverRole, string reason)`:
     - Kiểm tra `Status == PaymentStatus.PendingApproval`, `approverId != CreatedBy` (`PAY-07`), `approverRole == GetRequiredApproverRole()` (`ISS-02`).
     - Kiểm tra `!string.IsNullOrWhiteSpace(reason)` (`PAY-08`).
     - Gán `RejectionReason = reason`, chuyển `Status = PaymentStatus.Rejected` (`ISS-01`).
  7. `ReviseToDraft(Guid nvkdActorId)`:
     - Kiểm tra `Status == PaymentStatus.Rejected` (`ISS-01` & `PAY-08`).
     - Chuyển `Status = PaymentStatus.Draft` để NVKD sửa/đồng bộ lại và trình lại.
  8. `Cancel(Guid actorId)`:
     - Kiểm tra `Status == PaymentStatus.Draft` (`US-PAY-05`).
     - Kiểm tra `actorId == CreatedBy` (`ISS-08`: chỉ NVKD đã lập bảng mới được hủy bản nháp).
     - Chuyển `Status = PaymentStatus.Cancelled`.
  9. `CreateAdjustment(Guid kttActorId, string confirmationDocUrl, string reason, IReadOnlyList<PaymentAdjustmentLineInput> adjustedLines)`:
     - Kiểm tra bảng gốc `Status == PaymentStatus.Approved` (`US-PAY-04`).
     - Bản ghi `PaymentStatement` gốc và `Lines` gốc **giữ nguyên tuyệt đối không sửa** (`PAY-05` & `ISS-03`).
     - Kiểm tra `!string.IsNullOrWhiteSpace(confirmationDocUrl)` (`US-PAY-04`).
     - Khởi tạo đối tượng `PaymentAdjustment`: nếu `AdjustedTotalAmount >= 500_000_000m` thì `AdjustmentStatus = PendingGdkdApproval`, ngược lại `AdjustmentStatus = Approved` (`ISS-03`).

#### `PaymentAdjustment` & `PaymentAdjustmentLine` (Entities - `US-PAY-04` & `ISS-03`)
- `Id: Guid`, `PaymentStatementId: Guid`, `AdjustmentCode: string`, `ConfirmationDocumentUrl: string`, `Reason: string`, `PreviousTotalAmount: decimal`, `AdjustedTotalAmount: decimal`, `Status: AdjustmentStatus`, `CreatedByKttId: Guid`, `CreatedAt: DateTimeOffset`, `ApprovedByGdkdId: Guid?`, `ApprovedAt: DateTimeOffset?`, `Lines: IReadOnlyCollection<PaymentAdjustmentLine>`.

#### `PaymentAuditLog` (Entity - `NFR-01`)
- Ghi nhận mọi sự kiện: `Created`, `UpdatedDraft`, `ResyncedDraft`, `Submitted`, `Approved`, `Rejected`, `RevisedToDraft`, `Cancelled`, `AdjustmentCreated`, `AdjustmentApproved`.

---

## 2. Bảng Chuyển đổi Trạng thái Đầy đủ (`State Transitions`)

| Đối tượng | Trạng thái đầu | Hành động | Điều kiện (Guard) | Trạng thái đích | Rule / Quyết định |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `PaymentStatement` | `[*]` | `create()` | `Contract.Active`, có bảng giá, `Volume.IsReconciled`, chưa có bảng `!= Cancelled` cùng kỳ | `Draft` | `US-PAY-01`, `PAY-01..03`, `ISS-04`, `ISS-05` |
| `PaymentStatement` | `Draft` | `updateNote()` / `resync()` | `Actor == NVKD` (không sửa trực tiếp số lượng/đơn giá) | `Draft` | `US-PAY-01`, `ISS-06` |
| `PaymentStatement` | `Draft` | `submit()` | `TotalAmount > 0` & đủ `MandatoryServiceIds` | `PendingApproval` | `US-PAY-02`, `PAY-04`, `ISS-07` |
| `PaymentStatement` | `Draft` | `cancel()` | `ActorId == CreatedBy` (NVKD lập bảng) | `Cancelled` | `US-PAY-05`, `ISS-08` |
| `PaymentStatement` | `PendingApproval` | `approve()` | `ApproverId != CreatedBy` & (`>= 500tr`: `GDKD`; `< 500tr`: `KTT` khác) | `Approved` | `US-PAY-03`, `PAY-06`, `PAY-07`, `ISS-02`, `ISS-08` |
| `PaymentStatement` | `PendingApproval` | `reject(reason)` | `ApproverId != CreatedBy`, đúng cấp duyệt, `reason != empty` | `Rejected` | `US-PAY-03`, `PAY-08`, `ISS-01` |
| `PaymentStatement` | `Rejected` | `revise()` | `Actor == NVKD` | `Draft` | `PAY-08`, `ISS-01` |
| `PaymentAdjustment` | `[*]` (trên bảng `Approved`) | `createAdjustment()` | `Actor == KTT`, có `ConfirmationDocumentUrl`, `AdjustedTotalAmount < 500tr` | `Approved` | `US-PAY-04`, `PAY-05`, `ISS-03` |
| `PaymentAdjustment` | `[*]` (trên bảng `Approved`) | `createAdjustment()` | `Actor == KTT`, có `ConfirmationDocumentUrl`, `AdjustedTotalAmount >= 500tr` | `PendingGdkdApproval` | `US-PAY-04`, `PAY-05`, `ISS-03` |
| `PaymentAdjustment` | `PendingGdkdApproval` | `approveAdjustment()` | `Actor == GDKD` & `ApproverId != CreatedByKttId` | `Approved` | `US-PAY-04`, `ISS-03` |
