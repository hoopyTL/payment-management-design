# Bảng phân công nhóm 7 thành viên

> Bài tập: Thiết kế module bảng thanh toán bằng AI Agent. Bảng này là **đề xuất phân công**, không phải bằng chứng rằng từng thành viên đã thực hiện phần việc.

| Thành viên | Vai trò chính | Công việc và sản phẩm chịu trách nhiệm | Phần trình bày / vấn đáp |
| :--- | :--- | :--- | :--- |
| **Bảo** | Điều phối, tổng hợp | Theo dõi tiến độ, thống nhất quyết định của nhóm, kiểm tra đủ sản phẩm theo đề; quản lý bảng phân công này và ghép slide. | Giới thiệu phạm vi bài toán, cách nhóm làm việc và phần AI hỗ trợ. |
| **Thuận** | Phân tích nghiệp vụ (BA) | Đối chiếu PDF gốc với `QTKD_BA_Analysis.xlsx` và `01-requirements-summary.md`; kiểm tra 5 user story, 8 quy tắc `PAY-01`–`PAY-08`, 2 NFR; ghi rõ căn cứ trang/mục. | Trình bày yêu cầu nghiệp vụ, luồng trạng thái và các điều kiện đầu vào. |
| **Khánh** | Xử lý điểm chưa rõ | Rà soát 8 vấn đề `ISS-01`–`ISS-08`, phân loại mâu thuẫn/mơ hồ/thiếu thông tin, ghi lý do cần xác nhận và quyết định của nhóm; đối chiếu `07-design-decisions.md`. | Giải thích 2–3 mâu thuẫn quan trọng và vì sao nhóm chọn cách xử lý. |
| **Hoàng** | Kiến trúc và UML | Kiểm tra `02-architecture-design.md`, `03-detailed-design.md` và 4 sơ đồ trong `diagrams/`; bảo đảm component, class, sequence và state machine dùng cùng tên, trạng thái, quy tắc. | Trình bày kiến trúc, các thành phần và một luồng nghiệp vụ tiêu biểu. |
| **Thạnh** | API và dữ liệu | Kiểm tra `04-api-contract.md`, `05-data-design.md`; rà soát endpoint, phân quyền, mã lỗi, ràng buộc dữ liệu, audit log, chống gửi trùng và ngưỡng 500 triệu. | Trình bày API, mô hình dữ liệu và cách thiết kế đáp ứng NFR. |
| **Anh** | AI Reviewer và truy vết | Đánh giá nhận xét AI là đúng/sai/thừa trong `evaluation-report.md`; so sánh bản `payment-management-v1-initial/` với bản `payment-management/`; kiểm tra `06-traceability-matrix.md` nối yêu cầu với thiết kế. | Trình bày bằng chứng review và những thay đổi trước/sau. |
| **Hộp** | Minh chứng và trình bày | Lập **nhật ký prompt** (prompt, công cụ/agent, đầu ra, nhóm kiểm tra và sửa gì); thu thập minh chứng chạy agent/rules; soạn slide **tối đa 5 trang** cùng Bảo, kiểm tra tính đọc được và tập dượt. | Trình bày prompt, minh chứng AI và điều phối phần hỏi đáp cá nhân. |

## Việc cần hoàn thiện trước khi nộp

1. **Hộp + Bảo:** Tạo nhật ký prompt và slide tối đa 5 trang; hiện chưa thấy hai sản phẩm này trong workspace.
2. **Thuận, Khánh, Hoàng, Thạnh và Anh:** Rà soát phần mình phụ trách trên bản thiết kế sau review, ghi chỉnh sửa thực tế và minh chứng; không ghi nhận một quyết định là đã được nhóm xác nhận nếu chưa có xác nhận thật.
3. **Cả 7 thành viên:** Đọc toàn bộ 5 user story, `PAY-01`–`PAY-08`, `ISS-01`–`ISS-08` và chuẩn bị giải thích một thay đổi yêu cầu tại chỗ. Vấn đáp cá nhân không thể giao cho một người thay nhóm.

## Gợi ý bố cục 5 slide

1. Bài toán, phạm vi, vai trò và quy trình dùng AI — **Bảo, Thuận**.
2. Các điểm mâu thuẫn/mơ hồ và cách nhóm xử lý — **Khánh**.
3. Kiến trúc và sơ đồ thiết kế chính — **Hoàng**.
4. API, dữ liệu, truy vết và yêu cầu phi chức năng — **Thạnh, Anh**.
5. So sánh trước/sau review, đánh giá AI, prompt và minh chứng — **Anh, Hộp**.

**Phân biệt đóng góp:** AI có thể tạo bản nháp phân tích, thiết kế và nhận xét. Thành viên phụ trách phải kiểm tra với đặc tả, chỉ ra chỗ AI đúng/sai/thừa, xác nhận quyết định nghiệp vụ và sửa sản phẩm cuối. Nhật ký prompt cần ghi việc đã thực hiện, không dựng lại như một lịch sử có thật.

