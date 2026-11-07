# 05. Data Design — `payment-management`

> **Trạng thái thiết kế**: `Approved by Reviewer`  
> **Hệ quản trị CSDL**: PostgreSQL

---

## 1. Danh sách Bảng Dữ liệu (Tables)

### 1.1. Bảng `payment_statements` (Bảng thanh toán gốc)

| Cột (Column) | Kiểu dữ liệu | Ràng buộc (Constraints) | Mô tả & Quy tắc liên quan |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Khóa chính bảng thanh toán |
| `statement_code` | `VARCHAR(50)` | `NOT NULL, UNIQUE` | Mã bảng thanh toán |
| `contract_id` | `UUID` | `NOT NULL` | Tham chiếu sang Hợp đồng (`PAY-01`) |
| `period_code` | `VARCHAR(20)` | `NOT NULL` | Mã kỳ thanh toán tháng (`2026-11`) hoặc quý (`2026-Q4`) (`ISS-04`) |
| `period_type` | `VARCHAR(15)` | `NOT NULL, CHECK (period_type IN ('Monthly', 'Quarterly'))` | Loại kỳ thanh toán theo cấu hình hợp đồng (`ISS-04`) |
| `period_start_date` | `DATE` | `NOT NULL` | Ngày bắt đầu kỳ |
| `period_end_date` | `DATE` | `NOT NULL, CHECK (period_end_date >= period_start_date)` | Ngày kết thúc kỳ |
| `total_amount` | `NUMERIC(18, 2)` | `NOT NULL` | Tổng tiền toàn bảng (`= SUM(line_amount)`) |
| `status` | `VARCHAR(25)` | `NOT NULL, CHECK (status IN ('Draft', 'PendingApproval', 'Approved', 'Rejected', 'Cancelled'))` | 5 trạng thái bảng thanh toán (Mục 3 & `ISS-01`) |
| `note` | `TEXT` | `NULL` | Ghi chú của NVKD (`ISS-06`) |
| `rejection_reason` | `TEXT` | `NULL` | Lý do từ chối (`PAY-08`) |
| `created_by` | `UUID` | `NOT NULL` | NVKD lập bảng (`PAY-07`, `ISS-08`) |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT NOW()` | Thời điểm lập bảng |
| `submitted_at` | `TIMESTAMPTZ` | `NULL` | Thời điểm trình duyệt |
| `approved_by` | `UUID` | `NULL, CHECK (approved_by IS NULL OR approved_by <> created_by)` | Người duyệt khác người lập (`PAY-07`) |
| `approved_at` | `TIMESTAMPTZ` | `NULL` | Thời điểm duyệt |
| `xmin` | `XID` | System Concurrency Token | Chống trình duyệt trùng & duyệt trùng (`PAY-04`) |

#### Unique Partial Index (`PAY-01`, `ISS-04`, `ISS-05`):
```sql
-- Mỗi hợp đồng có tối đa 1 bảng thanh toán cho mỗi kỳ (tháng hoặc quý theo cấu hình).
-- Cho phép lập bảng mới cùng kỳ nếu bảng cũ đã bị hủy (Cancelled).
CREATE UNIQUE INDEX ux_payment_statements_contract_period_active
ON payment_statements (contract_id, period_code)
WHERE status <> 'Cancelled';
```

---

### 1.2. Bảng `payment_lines` (Dòng thanh toán gốc)

| Cột (Column) | Kiểu dữ liệu | Ràng buộc (Constraints) | Mô tả & Quy tắc liên quan |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Khóa chính dòng thanh toán |
| `payment_statement_id` | `UUID` | `NOT NULL, REFERENCES payment_statements(id) ON DELETE CASCADE` | Khóa ngoại gắn với `payment_statements` |
| `service_id` | `UUID` | `NOT NULL` | Mã định danh dịch vụ (kiểm tra với `MandatoryServiceIds` - `ISS-07`) |
| `service_code` | `VARCHAR(50)` | `NOT NULL` | Mã dịch vụ |
| `service_name` | `VARCHAR(255)` | `NOT NULL` | Tên dịch vụ |
| `quantity` | `NUMERIC(18, 4)` | `NOT NULL, CHECK (quantity >= 0)` | Sản lượng đã đối soát (`PAY-02`) |
| `unit_price` | `NUMERIC(18, 2)` | `NOT NULL, CHECK (unit_price >= 0)` | Đơn giá tại thời điểm tính (`PAY-03`) |
| `line_amount` | `NUMERIC(18, 2)` | `NOT NULL, CHECK (line_amount = ROUND(quantity * unit_price, 2))` | Thành tiền = sản lượng $\times$ đơn giá (`PAY-03`) |

- `UNIQUE (payment_statement_id, service_id)`

---

### 1.3. Bảng `payment_adjustments` & `payment_adjustment_lines` (Bản ghi điều chỉnh riêng - `US-PAY-04`, `PAY-05`, `ISS-03`)

> Giải quyết triệt để mâu thuẫn giữa `US-PAY-04` và `PAY-05`: bảng `payment_statements` đã `Approved` không bao giờ bị sửa trực tiếp; mọi điều chỉnh của `KTT` được lưu thành bản ghi riêng trong `payment_adjustments`.

| Cột (`payment_adjustments`) | Kiểu dữ liệu | Ràng buộc (Constraints) | Mô tả & Quy tắc liên quan |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Khóa chính bản ghi điều chỉnh |
| `payment_statement_id` | `UUID` | `NOT NULL, REFERENCES payment_statements(id)` | Tham chiếu bảng thanh toán gốc đã `Approved` |
| `adjustment_code` | `VARCHAR(50)` | `NOT NULL, UNIQUE` | Mã bản ghi điều chỉnh |
| `confirmation_doc_url` | `TEXT` | `NOT NULL, CHECK (length(trim(confirmation_doc_url)) > 0)` | Biên bản xác nhận đính kèm bắt buộc (`US-PAY-04`) |
| `reason` | `TEXT` | `NOT NULL` | Nội dung / lý do điều chỉnh |
| `previous_total_amount` | `NUMERIC(18, 2)` | `NOT NULL` | Tổng tiền trước điều chỉnh |
| `adjusted_total_amount` | `NUMERIC(18, 2)` | `NOT NULL` | Tổng tiền sau điều chỉnh |
| `status` | `VARCHAR(30)` | `NOT NULL, CHECK (status IN ('PendingGdkdApproval', 'Approved', 'Rejected'))` | Nếu `adjusted_total_amount >= 500000000` thì `PendingGdkdApproval` chờ GĐKD duyệt lại (`ISS-03`) |
| `created_by_ktt_id` | `UUID` | `NOT NULL` | KTT thực hiện tạo bản điều chỉnh |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT NOW()` | Thời điểm tạo điều chỉnh |
| `approved_by_gdkd_id` | `UUID` | `NULL` | GĐKD duyệt lại (khi `>= 500 triệu` - `ISS-03`) |
| `approved_at` | `TIMESTAMPTZ` | `NULL` | Thời điểm GĐKD duyệt lại |

*(Bảng `payment_adjustment_lines` có cấu trúc dòng tương tự `payment_lines`, khóa ngoại `payment_adjustment_id REFERENCES payment_adjustments(id)`).*

---

### 1.4. Bảng `payment_audit_logs` (Nhật ký truy vết - `NFR-01`)

| Cột (Column) | Kiểu dữ liệu | Ràng buộc (Constraints) | Mô tả |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Khóa chính bản ghi log |
| `payment_statement_id` | `UUID` | `NOT NULL, REFERENCES payment_statements(id)` | Bảng thanh toán liên quan |
| `adjustment_id` | `UUID` | `NULL, REFERENCES payment_adjustments(id)` | Bản ghi điều chỉnh liên quan (nếu là log điều chỉnh) |
| `actor_id` | `UUID` | `NOT NULL` | Người thực hiện ("ai") |
| `occurred_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT NOW()` | Thời điểm thực hiện ("lúc nào") |
| `action` | `VARCHAR(50)` | `NOT NULL` | Hành động (`Created`, `UpdatedDraft`, `ResyncedDraft`, `Submitted`, `Approved`, `Rejected`, `RevisedToDraft`, `Cancelled`, `AdjustmentCreated`, `AdjustmentApproved`) |
| `old_status` | `VARCHAR(30)` | `NULL` | Trạng thái trước |
| `new_status` | `VARCHAR(30)` | `NOT NULL` | Trạng thái sau |
| `change_details_json` | `JSONB` | `NOT NULL` | Chi tiết thay đổi ("thay đổi gì") |
