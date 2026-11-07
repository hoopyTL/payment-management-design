# 01. Requirements Summary — Module Bảng thanh toán (`payment-management`)

> **Trạng thái thiết kế**: `Approved by Reviewer` (Đã cập nhật theo toàn bộ quyết định chốt `ISS-01`..`ISS-08` của nhóm)  
> **Tài liệu nguồn**: `docs/requirements/BAITAP_TA_1-2.pdf` (Trang 1–3)  
> **Bảng phân tích BA (Excel)**: `docs/requirements/QTKD_BA_Analysis.xlsx`  
> **Bản thiết kế lần chạy đầu (trước khi chốt câu hỏi)**: `docs/design/payment-management-v1-initial/`

---

## 1. Phạm vi Chức năng (Toàn bộ 5 User Stories sau khi Nhóm xác nhận)

| Mã US | Tên User Story | Actor | Nội dung sau khi Nhóm xác nhận các điểm mâu thuẫn / mơ hồ |
| :--- | :--- | :--- | :--- |
| **`US-PAY-01`** | **Lập bảng thanh toán & Chỉnh sửa bảng nháp** | `NVKD` | - Chọn Hợp đồng (`Active`, có bảng giá phù hợp) và Kỳ thanh toán (tháng hoặc quý theo cấu hình hợp đồng - `ISS-04`).<br>- Kiểm tra chưa có bảng thanh toán `!= Cancelled` trong cùng kỳ (`ISS-04`, `ISS-05`) và sản lượng kỳ ở trạng thái `"Đã đối soát"` (`PAY-02`).<br>- Tự động sinh các dòng thanh toán với `Thành tiền = Sản lượng × Đơn giá tại thời điểm tính` (`PAY-03`), lưu trạng thái `Draft` trong `< 3 giây` với `< 200 dòng` (`NFR-02`).<br>- Khi ở `Draft`, NVKD **chỉ được sửa ghi chú/mô tả hoặc chọn đồng bộ và tính lại từ dữ liệu nguồn**; không sửa trực tiếp số lượng, đơn giá hay thêm/bớt dòng (`ISS-06`). |
| **`US-PAY-02`** | **Trình duyệt bảng thanh toán** | `NVKD` | - Trình duyệt bảng đang ở trạng thái `Draft` (`Draft -> PendingApproval`).<br>- Chặn trình duyệt khi **`Tổng tiền <= 0`** và dùng **`MandatoryServiceIds`** để kiểm tra đủ các dòng dịch vụ bắt buộc (`PAY-04`, `ISS-07`).<br>- Chống thao tác trình duyệt trùng (`PAY-04`) và gửi thông báo cho đúng cấp duyệt (`>= 500tr` gửi `GĐKD`; `< 500tr` gửi `KTT` khác người lập - `ISS-02`, `ISS-08`). |
| **`US-PAY-03`** | **Duyệt hoặc từ chối bảng thanh toán** | `KTT`, `GĐKD`, `NVKD` | - Bảng có tổng tiền **từ 500 triệu đồng trở lên (`>= 500.000.000`)** do `GĐKD` duyệt; **dưới 500 triệu đồng (`< 500.000.000`)** do `KTT` duyệt (`ISS-02`).<br>- Người lập bảng không được duyệt bảng do chính mình lập (`PAY-07`); nếu người lập cũng là `KTT` thì chuyển sang `KTT` khác duyệt (`ISS-08`).<br>- Khi duyệt: chuyển sang `Approved`.<br>- Khi từ chối: bắt buộc nhập lý do (`PAY-08`), bảng chuyển sang **`Rejected`**; khi `NVKD` chọn **Chỉnh sửa lại**, bảng chuyển từ **`Rejected` về `Draft`** để đồng bộ/sửa ghi chú và trình lại (`ISS-01`). |
| **`US-PAY-04`** | **Điều chỉnh bảng đã duyệt** | `KTT`, `GĐKD` | - Bảng thanh toán gốc đã `Approved` là bất biến, không được sửa trực tiếp (`PAY-05`).<br>- Khi phát hiện sai sót, `KTT` **tạo bản ghi điều chỉnh riêng (`PaymentAdjustment`)** kèm biên bản xác nhận đính kèm (`ISS-03`).<br>- Nếu tổng tiền sau điều chỉnh **từ 500 triệu đồng trở lên (`>= 500.000.000`)** thì chuyển trạng thái chờ **`GĐKD` duyệt lại** (`PendingGdkdApproval -> Approved`); nếu `< 500.000.000` thì có hiệu lực ngay (`Approved`) (`ISS-03`).<br>- Mọi điều chỉnh được ghi Audit Log (`NFR-01`). |
| **`US-PAY-05`** | **Hủy bảng nháp** | `NVKD` đã lập bảng | - **Chỉ `NVKD` đã lập bảng** (`ActorId == CreatedBy`) mới được hủy bản nháp đang ở trạng thái `Draft` (`ISS-08`).<br>- Sau khi hủy, bảng chuyển sang `Cancelled` và không dùng lại được (`US-PAY-05`).<br>- Cho phép lập bảng thanh toán mới cùng kỳ nếu bảng cũ đã bị hủy (`ISS-05`). |

---

## 2. Bảng Tổng hợp Quyết định Xử lý 8 Vấn đề trong Đặc tả (`ISS-01` → `ISS-08`)

| Mã vấn đề | Loại vấn đề | Vị trí trong đặc tả (`BAITAP_TA_1-2.pdf`) | Quyết định của Nhóm (Đã xác nhận) | Đánh giá AI |
| :--- | :--- | :--- | :--- | :--- |
| **`ISS-01`** | Mâu thuẫn | Trang 1, Mục 3 vs. Trang 2, Mục 5 (`PAY-08`) | Chọn phương án 1: Khi bị từ chối, bảng sang `Rejected`. Khi NVKD chọn chỉnh sửa lại, bảng về `Draft`. | Đúng |
| **`ISS-02`** | Mâu thuẫn | Trang 2, Mục 4.3 (`US-PAY-03`) vs. Trang 2, Mục 5 (`PAY-06`) | Từ 500 triệu đồng trở lên (`>= 500.000.000`) do `GĐKD` duyệt; dưới 500 triệu đồng (`< 500.000.000`) do `KTT` duyệt. | Đúng |
| **`ISS-03`** | Mâu thuẫn | Trang 2, Mục 4.4 (`US-PAY-04`) vs. Trang 2, Mục 5 (`PAY-05`) | Tạo bản ghi điều chỉnh riêng (`PaymentAdjustment`). Nếu tổng tiền sau điều chỉnh từ 500 triệu đồng trở lên (`>= 500.000.000`) thì `GĐKD` duyệt lại. | Đúng |
| **`ISS-04`** | Mâu thuẫn | Trang 1, Mục 2 vs. Trang 2, Mục 5 (`PAY-01`) | Mỗi hợp đồng có tối đa một bảng thanh toán cho mỗi kỳ tháng hoặc quý theo cấu hình. | Đúng |
| **`ISS-05`** | Thiếu thông tin | Trang 2, Mục 4.5 (`US-PAY-05`) & `PAY-01` | Cho lập bảng mới cùng kỳ nếu bảng cũ đã bị hủy (`Cancelled`). | Đúng |
| **`ISS-06`** | Mơ hồ | Trang 2, Mục 4.1 (`US-PAY-01`) vs. Mục 1, `PAY-02`, `PAY-03` | Khi bảng ở `Draft`, NVKD chỉ được sửa ghi chú/mô tả hoặc chọn đồng bộ và tính lại từ dữ liệu nguồn. Không sửa trực tiếp số lượng, đơn giá hay thêm/bớt dòng. | Đúng |
| **`ISS-07`** | Mơ hồ & Thiếu TT | Trang 2, Mục 5 (`PAY-04`) | 1. Chặn trình duyệt khi tổng tiền `<= 0`.<br>2. Dùng `MandatoryServiceIds` để kiểm tra các dòng dịch vụ bắt buộc. | Đúng |
| **`ISS-08`** | Thiếu thông tin | Trang 2, Mục 4.5 (`US-PAY-05`) & `PAY-07` | 1. Chỉ NVKD đã lập bảng được hủy bản nháp.<br>2. Nếu người lập cũng là KTT thì chuyển sang KTT khác duyệt. | Đúng |

---

## 3. Yêu cầu Phi chức năng (`BAITAP_TA_1-2.pdf`, Trang 3, Mục 6)
- **`NFR-01` (Audit Log)**: Mọi thay đổi trạng thái và mọi điều chỉnh phải được ghi audit log (ai, lúc nào, thay đổi gì).
- **`NFR-02` (Hiệu năng)**: Chức năng lập bảng phải phản hồi trong vòng `3 giây` với hợp đồng có dưới `200 dòng`.
