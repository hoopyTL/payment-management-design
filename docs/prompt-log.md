# Nhật Ký Prompt & Minh Chứng Làm Việc Với AI Agent

> **Đề tài**: Thiết kế module bảng thanh toán bằng AI Agent (`payment-management`)  
> **Công cụ sử dụng**: Antigravity / Cursor AI Agent (kết hợp `.cursor/agents/` và `.cursor/rules/`)  
> **Tài liệu đặc tả nguồn**: `docs/requirements/BAITAP_TA_1-2.pdf`

---

## 1. Chi Tiết Nhật Ký Prompt Theo Trình Tự Thời Gian

### Prompt #1 — Đọc cấu hình Agent & Rules
- **Thời điểm**: `2026-11-07 14:00`
- **Prompt**:
  > *"đọc .cursor nhé !"*
- **Kết quả AI thực hiện**:
  - Đọc toàn bộ cấu hình trong `.cursor/rules/qtkd-context.mdc`, `.cursor/agents/ba.md`, `.cursor/agents/design.md`.
  - Xác nhận nguyên tắc: đọc tài liệu trong `docs/requirements/`, ghi rõ mã rule và trang/mục tham chiếu, phân biệt yêu cầu tài liệu với đề xuất kỹ thuật, ghi `Cần xác nhận` nếu thiếu thông tin, không tự ý sửa code khi chỉ phân tích/thiết kế.

---

### Prompt #2 — Thực hiện nhiệm vụ BA
- **Thời điểm**: `2026-11-07 14:02`
- **Prompt**:
  > *"Thực hiện nhiệm vụ BA trong `@.cursor/agents/ba.md`"*
- **Kết quả AI thực hiện**:
  - Phân tích tài liệu cũ đang có sẵn trong `docs/requirements/` và xuất file Excel phân tích BA lần đầu.

---

### Prompt #3 — Thực hiện thiết kế tối thiểu (`v1-initial`)
- **Thời điểm**: `2026-11-07 14:12` – `14:15`
- **Prompt**:
  > *"Hãy thực hiện nhiệm vụ design tối thiểu dựa trên `@.cursor/agents/design.md` để trong folder docs/design, hãy lựa chọn User Story có đủ thông tin yêu câu, chỉ cần trước class diagram và sequence diagram thôi."*
- **Kết quả AI thực hiện**:
  - Đọc đúng tài liệu `docs/requirements/BAITAP_TA_1-2.pdf` (3 trang), xóa nháp thiết kế từ file cũ.
  - Cập nhật `docs/requirements/QTKD_BA_Analysis.xlsx` với 4 sheet (`01_TongQuan_Actors`, `02_User_Stories`, `03_Business_Rules_NFR`, `04_VanDe_CanXacNhan`), phát hiện **8 vấn đề (`ISS-01` $\rightarrow$ `ISS-08`)** gồm 4 mâu thuẫn, 2 mơ hồ, 2 thiếu thông tin.
  - Tạo bản thiết kế tối thiểu lần đầu (`docs/design/payment-management/` — sau này lưu snapshot tại `docs/design/payment-management-v1-initial/`): chọn thiết kế `US-PAY-01`, `US-PAY-02`, `US-PAY-05`; cô lập `US-PAY-03` qua `IPaymentApprovalPolicy` và tạm khóa `US-PAY-04` (`Blocked by ISS-03`) để không tự suy diễn.
  - Tạo 2 sơ đồ UML ban đầu: `diagrams/class.puml` và `diagrams/sequence.puml`.


---

### Prompt #4 — Cung cấp kết quả review 8 vấn đề (`ISS-01..ISS-08`) và cập nhật thiết kế hoàn chỉnh (`v2`)
- **Thời điểm**: `2026-11-07 16:13`
- **Prompt**:
  > *"Đây là quyết định của nhóm mình nhé !"* *(kèm ảnh bảng kết quả review `04_VanDe_CanXacNhan` đã điền cột `Cách nhóm xử lý` và `Nhận xét: Đúng` cho toàn bộ `ISS-01` $\rightarrow$ `ISS-08`)*
- **Kết quả AI thực hiện**:
  - Lưu trữ bản thiết kế lần đầu vào `docs/design/payment-management-v1-initial/`.
  - Cập nhật `docs/requirements/QTKD_BA_Analysis.xlsx` chuyển toàn bộ 8 vấn đề sang trạng thái `Đã xác nhận`.
  - Cập nhật bộ tài liệu `docs/design/payment-management/` sang trạng thái `Approved by Reviewer`, bổ sung thiết kế cho `US-PAY-04` (`PaymentAdjustment`, `PaymentAdjustmentLine`) và các quyết định đã chốt.
  - Hoàn thiện và kiểm tra render thành công cả 4 sơ đồ PlantUML: `component.puml`, `class.puml`, `sequence.puml`, `state-machine.puml`.
  - Tạo báo cáo đánh giá & so sánh trước/sau tại `docs/design/evaluation-report.md`.


---

## 2. Minh Chứng Sản Phẩm Đầu Ra Trong Workspace

| Hạng mục | Đường dẫn trong repo | Trạng thái |
| :--- | :--- | :--- |
| Cấu hình Agent & Rules | `.cursor/agents/ba.md`, `.cursor/agents/design.md`, `.cursor/rules/qtkd-context.mdc` | Đã sử dụng |
| Phân tích BA & Bảng 8 vấn đề | `docs/requirements/QTKD_BA_Analysis.xlsx` | Đã chốt đủ 8 vấn đề |
| Bản thiết kế lần đầu (`v1`) | `docs/design/payment-management-v1-initial/` | Đã lưu đối chiếu |
| Bản thiết kế sau review (`v2`) | `docs/design/payment-management/` (8 file `.md` + 4 file `.puml`) | Đã kiểm tra render SVG không lỗi |
| Báo cáo đánh giá & So sánh | `docs/design/evaluation-report.md` | Hoàn thành |
| Nhật ký prompt | `docs/prompt-log.md` | Hoàn thành |
