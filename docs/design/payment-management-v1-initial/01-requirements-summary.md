# 01. Requirements Summary — Module Bảng thanh toán (`payment-management`)

> **Trạng thái thiết kế**: `Design Pending Approval`  
> **Tài liệu nguồn**: `docs/requirements/BAITAP_TA_1-2.pdf` (Trang 1–3)  
> **Bảng phân tích BA (Excel)**: `docs/requirements/QTKD_BA_Analysis.xlsx`

---

## 1. Lựa chọn Phạm vi Thiết kế (User Story có đủ thông tin vs. User Story có mâu thuẫn)

Theo quy định tại `.cursor/agents/design.md` (*"Nếu có mâu thuẫn giữa các tài liệu, phải nêu rõ và yêu cầu xác nhận trước khi quyết định thiết kế"*, *"Không đưa ra quyết định thay người dùng đối với các yêu cầu nghiệp vụ chưa rõ"*):

| Mã US | Tên User Story | Đánh giá mức độ rõ ràng của yêu cầu | Quyết định đưa vào Thiết kế tối thiểu |
| :--- | :--- | :--- | :--- |
| **`US-PAY-01`** | **Lập bảng thanh toán (NVKD)** | **Đủ thông tin cốt lõi** để thiết kế luồng tự động sinh dòng thanh toán từ sản lượng & bảng giá (`PAY-01`, `PAY-02`, `PAY-03`, `NFR-01`, `NFR-02`). Phần "chỉnh sửa khi ở Draft" và kỳ "tháng vs quý" được tách riêng để xác nhận. | **CÓ THIẾT KẾ** (Trọng tâm Class & Sequence Diagram) |
| **`US-PAY-02`** | **Trình duyệt (NVKD)** | **Đủ thông tin cốt lõi** (`Draft -> PendingApproval`, kiểm tra `PAY-04`: tổng tiền không âm, đủ dòng dịch vụ bắt buộc, chống trình duyệt trùng, gửi thông báo, ghi Audit Log). | **CÓ THIẾT KẾ** (Trọng tâm Class & Sequence Diagram) |
| **`US-PAY-05`** | **Hủy bảng nháp** | **Đủ thông tin cốt lõi** (`Draft -> Cancelled`, không dùng lại được, ghi Audit Log). Cần xác nhận Actor thực hiện và việc tạo bảng mới thay thế bảng đã hủy. | **CÓ THIẾT KẾ** |
| **`US-PAY-03`** | **Duyệt hoặc từ chối** | Luồng `Approve` (`PendingApproval -> Approved`) và rule `PAY-07` rõ ràng; nhưng có **2 mâu thuẫn trực tiếp**: (1) Ngưỡng đúng `500.000.000đ` giữa `Mục 4.3` (`>= 500tr`) và `PAY-06` (`> 500tr`); (2) Trạng thái sau từ chối giữa `Mục 3` (`-> Rejected`) và `PAY-08` (`-> Draft`). | **Thiết kế khung + Tách biến cấu hình/trạng thái chờ Reviewer chốt** (Không tự suy diễn) |
| **`US-PAY-04`** | **Điều chỉnh bảng đã duyệt (KTT)** | **Mâu thuẫn trực tiếp 100%** với `PAY-05` (*"Bảng thanh toán đã Approved không được sửa dưới bất kỳ hình thức nào"*). | **CHỜ XÁC NHẬN (`Blocked by ISS-03`)** — Không tự ý thiết kế sửa bảng `Approved` khi chưa có xác nhận của Reviewer. |

---

## 2. Actors và Vai trò (`BAITAP_TA_1-2.pdf`, Trang 1, Mục 2)

| Vai trò | Ý nghĩa / Trách nhiệm trong Module Bảng thanh toán | Căn cứ |
| :--- | :--- | :--- |
| **`NVKD`** | Nhân viên kinh doanh: Lập bảng thanh toán (`US-PAY-01`), chỉnh sửa khi ở `Draft`, trình duyệt bảng (`US-PAY-02`), hủy bảng nháp (`US-PAY-05`). | Trang 1 (Mục 1, 2), Trang 2 (Mục 4.1, 4.2, 4.5, `PAY-08`) |
| **`KTT`** | Kế toán trưởng: Duyệt hoặc từ chối bảng thanh toán dưới ngưỡng 500 triệu đồng (`US-PAY-03`, `PAY-06`); điều chỉnh bảng đã duyệt khi có biên bản xác nhận (`US-PAY-04` - đang mâu thuẫn với `PAY-05`). | Trang 1 (Mục 2), Trang 2 (Mục 4.3, 4.4, `PAY-05`, `PAY-06`) |
| **`GĐKD`** | Giám đốc kinh doanh: Duyệt hoặc từ chối bảng thanh toán từ/trên ngưỡng 500 triệu đồng (`US-PAY-03`, `PAY-06`). | Trang 1 (Mục 2), Trang 2 (Mục 4.3, `PAY-06`) |
| **External Modules (Read-only)** | Module Hợp đồng (`Contract`), Module Bảng giá (`PriceList`), Module Sản lượng (`ReconciledVolume`): Module Bảng thanh toán **chỉ đọc dữ liệu** từ các module này. | Trang 1 (Mục 1) |

---

## 3. Tổng hợp Business Rules & Yêu cầu Phi chức năng

| Mã Rule | Nội dung quy tắc | Căn cứ (`BAITAP_TA_1-2.pdf`) | Trạng thái |
| :--- | :--- | :--- | :--- |
| `PAY-01` | Chỉ lập bảng thanh toán cho hợp đồng đang hiệu lực (`Active`) và có bảng giá phù hợp. Mỗi hợp đồng chỉ có một bảng thanh toán cho mỗi tháng. | Trang 2, Mục 5 | Mâu thuẫn với Mục 2 (kỳ thanh toán là tháng hoặc quý) $\rightarrow$ Xem `ISS-04`, `ISS-05`. |
| `PAY-02` | Sản lượng của kỳ phải ở trạng thái `"Đã đối soát"` trước khi lập bảng. | Trang 2, Mục 5 | Rõ ràng. |
| `PAY-03` | Thành tiền mỗi dòng = sản lượng $\times$ đơn giá tại thời điểm tính. | Trang 2, Mục 5 | Rõ ràng (snapshot đơn giá). |
| `PAY-04` | Khi trình duyệt, hệ thống chặn nếu tổng tiền âm hoặc thiếu dòng dịch vụ bắt buộc. Hệ thống chống thao tác trình duyệt trùng. | Trang 2, Mục 5 | Cần xác nhận trường hợp `Tổng tiền = 0` và nguồn dòng dịch vụ bắt buộc (`ISS-07`). |
| `PAY-05` | Bảng thanh toán đã `Approved` không được sửa dưới bất kỳ hình thức nào. | Trang 2, Mục 5 | Mâu thuẫn trực tiếp với `US-PAY-04` (`ISS-03`). |
| `PAY-06` | Bảng có tổng tiền trên 500 triệu đồng phải do `GĐKD` duyệt; các bảng còn lại do `KTT` duyệt. | Trang 2, Mục 5 | Mâu thuẫn biên `= 500.000.000` với `US-PAY-03` (`ISS-02`). |
| `PAY-07` | Người lập bảng không được duyệt bảng do chính mình lập. | Trang 2, Mục 5 | Rõ ràng. Cần xác nhận người duyệt thay nếu trùng (`ISS-08`). |
| `PAY-08` | Khi từ chối phải nhập lý do; sau khi bị từ chối, bảng quay về trạng thái `Draft` để NVKD chỉnh sửa và trình lại. | Trang 2, Mục 5 | Mâu thuẫn trạng thái đích với `Mục 3` (`Rejected` vs `Draft` - `ISS-01`). |
| `NFR-01` | Mọi thay đổi trạng thái và mọi điều chỉnh phải được ghi audit log (ai, lúc nào, thay đổi gì). | Trang 3, Mục 6 | Rõ ràng. |
| `NFR-02` | Chức năng lập bảng phải phản hồi trong vòng 3 giây với hợp đồng có dưới 200 dòng. | Trang 3, Mục 6 | Rõ ràng. |

---

## 4. Danh sách Vấn đề Phát hiện trong Đặc tả (`Open Questions` — Cần xác nhận)

> [!IMPORTANT]
> Đây là bảng tổng hợp các điểm **Mâu thuẫn**, **Mơ hồ** và **Thiếu thông tin** trong `BAITAP_TA_1-2.pdf`. Thiết kế tuyệt đối không tự suy diễn mà tách rõ để Reviewer xác nhận.

| Mã vấn đề | Loại vấn đề | Vị trí trong đặc tả | Mô tả vấn đề & Lý do cần xác nhận |
| :--- | :--- | :--- | :--- |
| **`ISS-01`** | **Mâu thuẫn** | Trang 1, Mục 3 vs. Trang 2, Mục 5 (`PAY-08`) | **Mục 3** ghi: *"Khi bị từ chối, bảng chuyển sang `Rejected`"*. Nhưng **`PAY-08`** ghi: *"sau khi bị từ chối, bảng quay về trạng thái `Draft` để NVKD chỉnh sửa và trình lại"*. Nếu quay thẳng về `Draft` thì trạng thái `Rejected` không bao giờ được dùng; nếu dừng ở `Rejected` thì thiếu bước chuyển về `Draft`. |
| **`ISS-02`** | **Mâu thuẫn** | Trang 2, Mục 4.3 (`US-PAY-03`) vs. Trang 2, Mục 5 (`PAY-06`) | **`US-PAY-03`** ghi tổng tiền **từ 500 triệu đồng trở lên** (`>= 500.000.000`) do GĐKD duyệt. **`PAY-06`** ghi tổng tiền **trên 500 triệu đồng** (`> 500.000.000`) do GĐKD duyệt. Xung đột tại đúng mốc `500.000.000 đồng`. |
| **`ISS-03`** | **Mâu thuẫn** | Trang 2, Mục 4.4 (`US-PAY-04`) vs. Trang 2, Mục 5 (`PAY-05`) | **`US-PAY-04`** cho phép *"KTT có thể điều chỉnh bảng đã `Approved` nếu có biên bản xác nhận đính kèm"*, nhưng **`PAY-05`** quy định *"Bảng thanh toán đã `Approved` không được sửa dưới bất kỳ hình thức nào"*. Đồng thời chưa rõ sau khi điều chỉnh có cần duyệt lại không. |
| **`ISS-04`** | **Mâu thuẫn** | Trang 1, Mục 2 vs. Trang 2, Mục 5 (`PAY-01`) | **Mục 2** định nghĩa `Kỳ thanh toán` là **tháng hoặc quý** tùy cấu hình hợp đồng, nhưng **`PAY-01`** ghi *"Mỗi hợp đồng chỉ có một bảng thanh toán cho **mỗi tháng**"*. Hợp đồng theo Quý sẽ áp dụng ràng buộc duy nhất như thế nào? |
| **`ISS-05`** | **Thiếu thông tin** | Trang 2, Mục 4.5 (`US-PAY-05`) & `PAY-01` | Bảng ở `Draft` khi hủy chuyển sang `Cancelled` và *"không dùng lại được"* (`US-PAY-05`). Nếu `PAY-01` giới hạn 1 bảng/tháng (hoặc kỳ) mà tính cả bảng đã `Cancelled`, NVKD sẽ không thể lập lại bảng mới cho kỳ đó sau khi hủy bảng nháp. |
| **`ISS-06`** | **Mơ hồ** | Trang 2, Mục 4.1 (`US-PAY-01`) vs. Trang 1, Mục 1 & `PAY-02`, `PAY-03` | `US-PAY-01` cho phép *"NVKD được phép chỉnh sửa bảng khi còn ở Draft"*, nhưng Mục 1 nêu hợp đồng/bảng giá/sản lượng chỉ đọc từ module khác và sản lượng phải `"Đã đối soát"` (`PAY-02`). Chưa rõ NVKD được sửa trường nào ở `Draft` (chỉ sửa ghi chú/tính lại hay được sửa số lượng/đơn giá?). |
| **`ISS-07`** | **Mơ hồ & Thiếu TT** | Trang 2, Mục 5 (`PAY-04`) | 1) `PAY-04` chặn khi *"tổng tiền âm"*: vậy `Tổng tiền = 0` có được phép trình duyệt không?<br>2) Làm sao xác định danh sách *"dòng dịch vụ bắt buộc"* (thuộc tính của Hợp đồng hay Bảng giá)? |
| **`ISS-08`** | **Thiếu thông tin** | Trang 2, Mục 4.5 (`US-PAY-05`) & `PAY-07` | 1) `US-PAY-05` không ghi rõ Actor nào được phép hủy bảng `Draft` và có cần nhập lý do hủy không.<br>2) `PAY-07` cấm người lập tự duyệt bảng của mình: nếu người lập kiêm nhiệm KTT/GĐKD duy nhất thì ai duyệt thay? |

