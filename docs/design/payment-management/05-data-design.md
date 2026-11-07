# 05. Data Design — `payment-management`

> **Trạng thái thiết kế**: `Design Pending Approval`  
> **Hệ quản trị CSDL**: PostgreSQL

---

## 1. Danh sách Bảng (Tables) thuộc Module Bảng thanh toán

> **Lưu ý phạm vi dữ liệu (`BAITAP_TA_1-2.pdf`, Trang 1, Mục 1)**:  
> Quản lý hợp đồng, bảng giá và sản lượng thuộc các module khác; module `payment-management` chỉ đọc dữ liệu từ chúng và sở hữu 3 bảng dữ liệu dưới đây.

### 1.1. Bảng `payment_statements` (Bảng thanh toán)

| Cột (Column) | Kiểu dữ liệu (PostgreSQL) | Ràng buộc (Constraints) | Mô tả & Quy tắc liên quan |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Khóa chính bảng thanh toán |
| `statement_code` | `VARCHAR(50)` | `NOT NULL, UNIQUE` | Mã bảng thanh toán (VD: `PAY-202611-001`) |
| `contract_id` | `UUID` | `NOT NULL` | Tham chiếu sang Hợp đồng (`PAY-01`) |
| `period_code` | `VARCHAR(20)` | `NOT NULL` | Mã kỳ thanh toán (VD: `2026-11` hoặc `2026-Q4` — xem `ISS-04`) |
| `period_start_date` | `DATE` | `NOT NULL` | Ngày bắt đầu kỳ thanh toán |
| `period_end_date` | `DATE` | `NOT NULL, CHECK (period_end_date >= period_start_date)` | Ngày kết thúc kỳ thanh toán |
| `total_amount` | `NUMERIC(18, 2)` | `NOT NULL` | Tổng tiền toàn bảng (`= SUM(line_amount)`) |
| `status` | `VARCHAR(25)` | `NOT NULL, CHECK (status IN ('Draft', 'PendingApproval', 'Approved', 'Rejected', 'Cancelled'))` | Trạng thái bảng thanh toán (Mục 3) |
| `note` | `TEXT` | `NULL` | Ghi chú của NVKD |
| `rejection_reason` | `TEXT` | `NULL` | Lý do từ chối (`PAY-08`) |
| `created_by` | `UUID` | `NOT NULL` | ID của NVKD lập bảng (`PAY-07`) |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT NOW()` | Thời điểm lập bảng |
| `submitted_at` | `TIMESTAMPTZ` | `NULL` | Thời điểm trình duyệt (`US-PAY-02`) |
| `approved_by` | `UUID` | `NULL, CHECK (approved_by IS NULL OR approved_by <> created_by)` | Người duyệt (`KTT`/`GĐKD`), ràng buộc `PAY-07` |
| `approved_at` | `TIMESTAMPTZ` | `NULL` | Thời điểm duyệt |
| `xmin` | `XID` (System Column) | Concurrency Token | Phục vụ Optimistic Locking chống trình duyệt/duyệt trùng (`PAY-04`) |

#### Unique Constraint & Indexes trên `payment_statements`:
- **Unique Index chống trùng bảng trong kỳ (`PAY-01` & `ISS-05`)**:
  ```sql
  CREATE UNIQUE INDEX ux_payment_statements_contract_period_active
  ON payment_statements (contract_id, period_code)
  WHERE status <> 'Cancelled';
  ```
  *(Lưu ý: Điều kiện `WHERE status <> 'Cancelled'` giải quyết `ISS-05` để khi bảng nháp bị hủy thì NVKD có thể tạo bảng mới cho kỳ đó — đang chờ Reviewer xác nhận).*
- **Index truy vấn nhanh**: `ix_payment_statements_status_total_amount (status, total_amount)` phục vụ màn hình danh sách chờ duyệt của `KTT` và `GĐKD`.

---

### 1.2. Bảng `payment_lines` (Dòng thanh toán)

| Cột (Column) | Kiểu dữ liệu (PostgreSQL) | Ràng buộc (Constraints) | Mô tả & Quy tắc liên quan |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Khóa chính dòng thanh toán |
| `payment_statement_id` | `UUID` | `NOT NULL, REFERENCES payment_statements(id) ON DELETE CASCADE` | Khóa ngoại gắn với `payment_statements` |
| `service_id` | `UUID` | `NOT NULL` | Mã định danh dịch vụ |
| `service_code` | `VARCHAR(50)` | `NOT NULL` | Mã dịch vụ |
| `service_name` | `VARCHAR(255)` | `NOT NULL` | Tên dịch vụ tại thời điểm lập bảng |
| `quantity` | `NUMERIC(18, 4)` | `NOT NULL, CHECK (quantity >= 0)` | Sản lượng thực tế đã đối soát (`PAY-02`) |
| `unit_price` | `NUMERIC(18, 2)` | `NOT NULL, CHECK (unit_price >= 0)` | Đơn giá snapshot tại thời điểm tính (`PAY-03`) |
| `line_amount` | `NUMERIC(18, 2)` | `NOT NULL, CHECK (line_amount = ROUND(quantity * unit_price, 2))` | Thành tiền mỗi dòng = sản lượng $\times$ đơn giá (`PAY-03`) |

#### Unique Constraint & Indexes trên `payment_lines`:
- `UNIQUE (payment_statement_id, service_id)`: Mỗi dịch vụ xuất hiện tối đa 1 dòng trong 1 bảng thanh toán.
- `INDEX ix_payment_lines_statement_id (payment_statement_id)`: Tối ưu tải `< 200 dòng` trong `< 3 giây` (`NFR-02`).

---

### 1.3. Bảng `payment_audit_logs` (Nhật ký truy vết - `NFR-01`, Trang 3, Mục 6)

| Cột (Column) | Kiểu dữ liệu (PostgreSQL) | Ràng buộc (Constraints) | Mô tả |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | Khóa chính bản ghi log |
| `payment_statement_id` | `UUID` | `NOT NULL, REFERENCES payment_statements(id)` | Bảng thanh toán liên quan |
| `actor_id` | `UUID` | `NOT NULL` | Người thực hiện thao tác ("ai") |
| `occurred_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT NOW()` | Thời điểm thực hiện ("lúc nào") |
| `action` | `VARCHAR(50)` | `NOT NULL` | Hành động (`Created`, `UpdatedDraft`, `Submitted`, `Approved`, `Rejected`, `Cancelled`) |
| `old_status` | `VARCHAR(25)` | `NULL` | Trạng thái trước |
| `new_status` | `VARCHAR(25)` | `NOT NULL` | Trạng thái sau |
| `change_details_json` | `JSONB` | `NOT NULL` | Chi tiết thay đổi ("thay đổi gì": snapshot dữ liệu, lý do từ chối...) |

- **Index**: `ix_payment_audit_logs_statement_time (payment_statement_id, occurred_at DESC)`.
- **Quy tắc bảo vệ**: Bảng `payment_audit_logs` là append-only (chỉ `INSERT`, không `UPDATE`/`DELETE`).

