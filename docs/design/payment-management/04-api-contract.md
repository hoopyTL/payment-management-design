# 04. API Contract — `payment-management`

> **Trạng thái thiết kế**: `Approved by Reviewer`  
> **Base Path**: `/api/v1/payment-statements`

---

## 1. Lập bảng thanh toán (`US-PAY-01`)
- **Endpoint**: `POST /api/v1/payment-statements`
- **Authorization**: Role `NVKD`.
- **Request Body**:
```json
{
  "contractId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "periodCode": "2026-11",
  "note": "Bảng thanh toán kỳ tháng 11/2026"
}
```
*(Hỗ trợ cả mã kỳ theo tháng `"2026-11"` hoặc theo quý `"2026-Q4"` tùy cấu hình của hợp đồng — `ISS-04`)*
- **Response (`201 Created`)**: Đối tượng `PaymentStatementResponse` ở trạng thái `Draft` kèm danh sách `lines` (`quantity`, `unitPrice`, `lineAmount`).
- **Validation & Error Codes**:
  - `422 PAY_01_CONTRACT_NOT_ACTIVE`: Hợp đồng không ở trạng thái `Active`.
  - `422 PAY_01_INVALID_BILLING_PERIOD`: Kỳ truyền vào không khớp cấu hình kỳ (tháng/quý) của hợp đồng (`ISS-04`).
  - `409 PAY_01_DUPLICATE_ACTIVE_PERIOD_STATEMENT`: Hợp đồng đã có bảng thanh toán chưa bị hủy (`Status != Cancelled`) trong kỳ này (`ISS-04`, `ISS-05`).
  - `422 PAY_01_PRICE_LIST_NOT_FOUND`: Không có bảng giá phù hợp.
  - `422 PAY_02_VOLUME_NOT_RECONCILED`: Sản lượng của kỳ chưa ở trạng thái `"Đã đối soát"`.

---

## 2. Chỉnh sửa ghi chú hoặc Đồng bộ & Tính lại bảng nháp (`US-PAY-01` & `ISS-06`)
- **Endpoint**: `PUT /api/v1/payment-statements/{id}/draft`
- **Authorization**: Role `NVKD`.
- **Request Body**:
```json
{
  "note": "Đồng bộ lại dữ liệu sản lượng và bảng giá mới nhất",
  "resyncFromSource": true,
  "rowVersion": 1
}
```
- **Validation & Error Codes**:
  - Theo `ISS-06`: API **không nhận** danh sách sửa số lượng/đơn giá/thêm bớt dòng thủ công; nếu `resyncFromSource = true`, hệ thống đọc lại từ `IVolumeReadService` và `IPriceListReadService` để tính lại.
  - `422 ERR_STATEMENT_NOT_DRAFT`: Bảng không ở trạng thái `Draft`.

---

## 3. Trình duyệt bảng thanh toán (`US-PAY-02` & `ISS-07`)
- **Endpoint**: `POST /api/v1/payment-statements/{id}/submit`
- **Authorization**: Role `NVKD`.
- **Headers**: `Idempotency-Key: <UUID>` (Bắt buộc chống trình duyệt trùng - `PAY-04`).
- **Request Body**: `{ "rowVersion": 1 }`
- **Response (`200 OK`)**: Trạng thái `PendingApproval`, `requiredApproverRole` (`GDKD` nếu `>= 500.000.000`, `KTT` nếu `< 500.000.000` - `ISS-02`).
- **Validation & Error Codes**:
  - `422 PAY_04_NON_POSITIVE_TOTAL_AMOUNT`: Chặn trình duyệt khi **`TotalAmount <= 0`** (`ISS-07`).
  - `422 PAY_04_MISSING_MANDATORY_SERVICE`: Thiếu dòng dịch vụ bắt buộc nằm trong `MandatoryServiceIds` (`ISS-07`).
  - `409 PAY_04_DUPLICATE_SUBMISSION`: Request trình duyệt trùng.

---

## 4. Duyệt hoặc Từ chối bảng thanh toán (`US-PAY-03`, `ISS-01`, `ISS-02`, `ISS-08`)
- **Endpoints**:
  - `POST /api/v1/payment-statements/{id}/approve` — Body: `{ "rowVersion": 2 }`
  - `POST /api/v1/payment-statements/{id}/reject` — Body: `{ "reason": "Sai kỳ đối soát", "rowVersion": 2 }`
- **Authorization**:
  - `TotalAmount >= 500_000_000` $\rightarrow$ bắt buộc Role `GDKD` (`ISS-02`).
  - `TotalAmount < 500_000_000` $\rightarrow$ bắt buộc Role `KTT` (`ISS-02`).
  - Bắt buộc `CurrentUserId != statement.CreatedBy` (`PAY-07`; nếu người lập kiêm KTT thì KTT khác phải thực hiện duyệt - `ISS-08`).
- **Response (`200 OK`)**:
  - Khi `approve`: `status = "Approved"`.
  - Khi `reject`: `status = "Rejected"` (`ISS-01`).
- **Error Codes**:
  - `403 PAY_06_INVALID_APPROVER_ROLE`: Sai thẩm quyền duyệt theo mốc `500.000.000`.
  - `403 PAY_07_SELF_APPROVAL_FORBIDDEN`: Người lập bảng không được duyệt bảng do chính mình lập.
  - `422 PAY_08_REJECTION_REASON_REQUIRED`: Từ chối nhưng không nhập lý do.

---

## 5. Chuyển bảng bị Từ chối (`Rejected`) về `Draft` để chỉnh sửa lại (`ISS-01` & `PAY-08`)
- **Endpoint**: `POST /api/v1/payment-statements/{id}/revise`
- **Authorization**: Role `NVKD`.
- **Request Body**: `{ "rowVersion": 3 }`
- **Response (`200 OK`)**: Chuyển trạng thái từ `Rejected` $\rightarrow$ `Draft` để NVKD sửa ghi chú/đồng bộ lại và trình duyệt lại.

---

## 6. Tạo & Duyệt Bản ghi Điều chỉnh cho Bảng đã `Approved` (`US-PAY-04`, `PAY-05`, `ISS-03`)
- **Tạo bản ghi điều chỉnh**:
  - **Endpoint**: `POST /api/v1/payment-statements/{id}/adjustments`
  - **Authorization**: Role `KTT`.
  - **Request Body**:
  ```json
  {
    "confirmationDocumentUrl": "https://storage.abc.local/confirmations/BB_Xac_Nhan_01.pdf",
    "reason": "Điều chỉnh sản lượng container theo biên bản đối soát bổ sung",
    "resyncFromUpdatedSource": true
  }
  ```
  - **Behavior (`ISS-03`)**:
    - Giữ nguyên không sửa bảng gốc `PaymentStatement` (`PAY-05`).
    - Tạo bản ghi `PaymentAdjustment` riêng. Nếu `adjustedTotalAmount >= 500000000` thì `status = "PendingGdkdApproval"`, ngược lại `status = "Approved"`.
- **GĐKD duyệt bản ghi điều chỉnh (`>= 500 triệu`)**:
  - **Endpoint**: `POST /api/v1/payment-statements/{id}/adjustments/{adjustmentId}/approve`
  - **Authorization**: Role `GDKD`.

---

## 7. Hủy bảng thanh toán nháp (`US-PAY-05` & `ISS-08`)
- **Endpoint**: `POST /api/v1/payment-statements/{id}/cancel`
- **Authorization**: Role `NVKD` **VÀ** `CurrentUserId == statement.CreatedBy` (`ISS-08`: chỉ NVKD đã lập bảng mới được hủy).
- **Request Body**: `{ "rowVersion": 1 }`
- **Response (`200 OK`)**: `status = "Cancelled"`.
- **Error Codes**:
  - `403 ERR_ONLY_CREATOR_CAN_CANCEL`: Người dùng không phải là NVKD đã lập bảng nháp này (`ISS-08`).
  - `422 ERR_CANCEL_ONLY_ALLOWED_IN_DRAFT`: Bảng không ở trạng thái `Draft`.
