# 04. API Contract — `payment-management`

> **Trạng thái thiết kế**: `Design Pending Approval`  
> **Base Path**: `/api/v1/payment-statements`

---

## 1. Lập bảng thanh toán (`US-PAY-01`)
- **HTTP Method & Endpoint**: `POST /api/v1/payment-statements`
- **Purpose**: NVKD chọn hợp đồng và kỳ thanh toán; hệ thống tự sinh các dòng thanh toán từ sản lượng đã đối soát và bảng giá hiệu lực, tính tổng tiền, lưu ở trạng thái `Draft` và ghi Audit Log.
- **Authentication & Authorization**: JWT Bearer Token, Role = `NVKD`.
- **Request Body**:
```json
{
  "contractId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "periodCode": "2026-11",
  "note": "Bảng thanh toán kỳ tháng 11/2026"
}
```
- **Response Body (`201 Created`)**:
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "statementCode": "PAY-202611-001",
  "contractId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "periodCode": "2026-11",
  "status": "Draft",
  "totalAmount": 420000000,
  "rowVersion": 1,
  "lines": [
    {
      "serviceId": "11111111-2222-3333-4444-555555555555",
      "serviceCode": "SVC-CONT-20",
      "serviceName": "Bốc xếp container 20 feet",
      "quantity": 200,
      "unitPrice": 1500000,
      "lineAmount": 300000000
    },
    {
      "serviceId": "66666666-7777-8888-9999-000000000000",
      "serviceCode": "SVC-STORAGE",
      "serviceName": "Lưu kho bãi",
      "quantity": 400,
      "unitPrice": 300000,
      "lineAmount": 120000000
    }
  ],
  "createdBy": "99999999-8888-7777-6666-555555555555",
  "createdAt": "2026-11-07T14:20:00+07:00"
}
```
- **Validation Rules & Error Responses**:
  - `422 Unprocessable Entity` (`PAY_01_CONTRACT_NOT_ACTIVE`): Hợp đồng không ở trạng thái `Active`.
  - `422 Unprocessable Entity` (`PAY_01_PRICE_LIST_NOT_FOUND`): Hợp đồng không có bảng giá phù hợp cho kỳ tính phí.
  - `409 Conflict` (`PAY_01_DUPLICATE_PERIOD_STATEMENT`): Hợp đồng đã có bảng thanh toán cho kỳ/tháng này (xem `ISS-04`, `ISS-05`).
  - `422 Unprocessable Entity` (`PAY_02_VOLUME_NOT_RECONCILED`): Sản lượng của kỳ chưa ở trạng thái `"Đã đối soát"`.
- **Performance Requirement (`NFR-02`)**: Phản hồi `< 3 giây` với hợp đồng dưới 200 dòng.
- **Related Business Rules / AC**: `PAY-01`, `PAY-02`, `PAY-03`, `NFR-01`, `NFR-02`.

---

## 2. Chỉnh sửa bảng thanh toán ở trạng thái `Draft` (`US-PAY-01`)
- **HTTP Method & Endpoint**: `PUT /api/v1/payment-statements/{id}`
- **Purpose**: NVKD chỉnh sửa bảng thanh toán khi còn ở trạng thái `Draft` và ghi Audit Log.
- **Authorization**: Role = `NVKD`.
- **Request Body**:
```json
{
  "note": "Cập nhật ghi chú trước khi trình duyệt",
  "resyncFromSource": true,
  "rowVersion": 1
}
```
*(Lưu ý `ISS-06`: Trường hợp Reviewer xác nhận cho phép NVKD sửa trực tiếp các dòng thanh toán khi ở `Draft`, payload sẽ bổ sung mảng `lines`; hiện giữ ở trạng thái `Cần xác nhận` để không tự suy diễn).*
- **Validation & Error Responses**:
  - `422 Unprocessable Entity` (`ERR_STATEMENT_NOT_DRAFT` / `PAY_05_IMMUTABLE_APPROVED`): Bảng không ở trạng thái `Draft`.
  - `409 Conflict` (`ERR_CONCURRENCY_CONFLICT`): Xung đột `rowVersion`.

---

## 3. Trình duyệt bảng thanh toán (`US-PAY-02`)
- **HTTP Method & Endpoint**: `POST /api/v1/payment-statements/{id}/submit`
- **Purpose**: Kiểm tra hợp lệ theo `PAY-04` (tổng tiền không âm, đủ dòng dịch vụ bắt buộc, chống trình duyệt trùng), chuyển trạng thái sang `PendingApproval`, gửi thông báo cho người duyệt và ghi Audit Log.
- **Authorization**: Role = `NVKD`.
- **Request Headers**:
  - `Idempotency-Key: <UUID>` *(Chống thao tác trình duyệt trùng theo `PAY-04`)*.
- **Request Body**:
```json
{
  "rowVersion": 1
}
```
- **Response Body (`200 OK`)**:
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "PendingApproval",
  "assignedApproverRole": "KTT",
  "submittedAt": "2026-11-07T14:25:00+07:00",
  "rowVersion": 2
}
```
- **Validation & Error Responses**:
  - `422 Unprocessable Entity` (`PAY_04_NEGATIVE_TOTAL_AMOUNT`): Tổng tiền bảng thanh toán `< 0` *(Lưu ý `ISS-07`: cần xác nhận trường hợp `= 0`)*.
  - `422 Unprocessable Entity` (`PAY_04_MISSING_MANDATORY_SERVICE`): Thiếu dòng dịch vụ bắt buộc của hợp đồng.
  - `409 Conflict` (`PAY_04_DUPLICATE_SUBMISSION`): Bảng thanh toán đã được trình duyệt hoặc đang xử lý request trình duyệt trùng.
- **Related Business Rules / AC**: `PAY-04`, `PAY-06`, `NFR-01`.

---

## 4. Hủy bảng thanh toán nháp (`US-PAY-05`)
- **HTTP Method & Endpoint**: `POST /api/v1/payment-statements/{id}/cancel`
- **Purpose**: Hủy bảng thanh toán đang ở trạng thái `Draft` (`Draft -> Cancelled`, không dùng lại được) và ghi Audit Log.
- **Authorization**: Role = `NVKD` *(Xem `ISS-08` để xác nhận Actor)*.
- **Request Body**:
```json
{
  "cancelReason": "Lập nhầm kỳ thanh toán",
  "rowVersion": 1
}
```
- **Response (`200 OK`)**: Trạng thái `Cancelled`.
- **Validation & Error Responses**:
  - `422 Unprocessable Entity` (`ERR_CANCEL_ONLY_ALLOWED_IN_DRAFT`): Bảng không ở trạng thái `Draft`.

---

## 5. Duyệt hoặc Từ chối bảng thanh toán (`US-PAY-03` — Có điểm chờ xác nhận `ISS-01`, `ISS-02`)
- **HTTP Method & Endpoint**:
  - `POST /api/v1/payment-statements/{id}/approve`
  - `POST /api/v1/payment-statements/{id}/reject`
- **Authorization**: Role = `KTT` hoặc `GDKD` theo ngưỡng 500 triệu đồng (`US-PAY-03` vs `PAY-06` — `ISS-02`) và `CurrentUserId != CreatedBy` (`PAY-07`).
- **Validation & Error Responses**:
  - `403 Forbidden` (`PAY_06_INSUFFICIENT_APPROVAL_AUTHORITY`): Sai cấp phê duyệt theo ngưỡng 500 triệu đồng.
  - `403 Forbidden` (`PAY_07_SELF_APPROVAL_FORBIDDEN`): Người lập bảng không được duyệt bảng do chính mình lập.
  - `422 Unprocessable Entity` (`PAY_08_REJECTION_REASON_REQUIRED`): Khi từ chối bắt buộc nhập lý do.
- **Lưu ý trạng thái chờ xác nhận**:
  - `US-PAY-04` (Điều chỉnh bảng đã `Approved`) hiện **chưa mở API** do mâu thuẫn trực tiếp với `PAY-05` (`ISS-03`).

