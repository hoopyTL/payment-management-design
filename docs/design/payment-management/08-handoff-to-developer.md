# 08. Handoff to Developer — `payment-management`

> **Trạng thái thiết kế**: `Approved by Reviewer` (Tất cả 8 điểm `ISS-01`..`ISS-08` đã được nhóm xác nhận)

---

## 1. Phạm vi Chức năng Cần Triển khai (Toàn bộ 5 User Stories)

1. **`US-PAY-01`**:
   - Lập bảng thanh toán từ dữ liệu đọc qua `IContractReadService`, `IVolumeReadService`, `IPriceListReadService`.
   - Kiểm tra hợp đồng `Active`, có bảng giá phù hợp, sản lượng kỳ `"Đã đối soát"`, và chưa có bảng `Status != Cancelled` cho cùng kỳ tháng/quý (`ISS-04`, `ISS-05`).
   - Khi ở `Draft`: chỉ cho phép sửa `Note` hoặc đồng bộ & tính lại từ dữ liệu nguồn (`ISS-06`).
2. **`US-PAY-02`**:
   - Trình duyệt bảng `Draft -> PendingApproval`.
   - Chặn khi `TotalAmount <= 0` hoặc thiếu dịch vụ trong `MandatoryServiceIds` (`ISS-07`).
   - Chống trình duyệt trùng bằng `Idempotency-Key` + `xmin` (`PAY-04`).
   - Thông báo cho `GĐKD` (nếu `>= 500tr`) hoặc `KTT` khác người lập (nếu `< 500tr`).
3. **`US-PAY-03`**:
   - Duyệt (`PendingApproval -> Approved`) hoặc Từ chối (`PendingApproval -> Rejected` kèm lý do bắt buộc).
   - Phân quyền: `>= 500.000.000` do `GĐKD` duyệt; `< 500.000.000` do `KTT` duyệt (`ISS-02`).
   - Kiểm tra `ApproverId != CreatedBy` (`PAY-07`; nếu người lập là KTT thì chuyển KTT khác duyệt - `ISS-08`).
   - Cho phép NVKD chọn chỉnh sửa lại để chuyển bảng từ `Rejected` $\rightarrow$ `Draft` (`ISS-01`).
4. **`US-PAY-04`**:
   - Giữ bảng `Approved` bất biến (`PAY-05`).
   - KTT tạo bản ghi điều chỉnh riêng (`PaymentAdjustment`) kèm `ConfirmationDocumentUrl`.
   - Nếu `AdjustedTotalAmount >= 500.000.000` thì trạng thái là `PendingGdkdApproval` chờ GĐKD duyệt lại; nếu `< 500.000.000` thì `Approved` ngay (`ISS-03`).
5. **`US-PAY-05`**:
   - Chỉ NVKD đã lập bảng (`ActorId == CreatedBy`) được hủy bảng ở trạng thái `Draft` (`Draft -> Cancelled`) (`ISS-08`).
   - Cho phép lập bảng mới cùng kỳ khi bảng cũ đã `Cancelled` (`ISS-05`).

---

## 2. Danh sách Các Giả định & Quyết định Đã Được Phê Duyệt
- Toàn bộ 8 vấn đề `ISS-01` đến `ISS-08` đã được nhóm phê duyệt chính thức (không còn câu hỏi mở tồn đọng).

---

## 3. Tiêu chí Hoàn thành (Definition of Done)
- Triển khai đầy đủ API, Domain Model, EF Core Persistence theo đúng `03-detailed-design.md`, `04-api-contract.md`, `05-data-design.md`.
- Pass 100% Unit Tests cho `PAY-01`..`PAY-08` và 8 kịch bản `ISS-01`..`ISS-08`.
- Thời gian lập bảng `< 3 giây` với hợp đồng `< 200 dòng` (`NFR-02`).
- Mọi thay đổi trạng thái và điều chỉnh đều ghi vào `payment_audit_logs` (`NFR-01`).
