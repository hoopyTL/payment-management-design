# BẢNG PHÂN CÔNG NHÓM

**Đề tài:** Thiết kế module bảng thanh toán bằng AI Agent

**Số thành viên:** 07

**Sản phẩm chung:** Tài liệu phân tích yêu cầu, hai phiên bản thiết kế (trước và sau review), báo cáo đánh giá AI, nhật ký prompt và bài trình bày tối đa 5 slide.

## 1. Phân công nhiệm vụ

| STT | MSSV | Họ và tên | Công việc cụ thể | Sản phẩm phụ trách |
| :---: | :---: | :--- | :--- | :--- |
| 1 | 23127328 | **Nguyễn Quốc Bảo** | Rà soát cấu hình BA Agent, Design Agent và rules để bảo đảm AI không tự quyết định chỗ đặc tả chưa rõ; kiểm tra danh mục sản phẩm theo đề; tổng hợp bảng phân công và tài liệu bàn giao. | `.cursor/agents/`, `.cursor/rules/`, `08-handoff-to-developer.md`, bảng phân công và danh mục nộp bài. |
| 2 | 23127490 | **Hồ Đức Thuận** | Đối chiếu PDF với 5 user story, 8 quy tắc và 2 yêu cầu phi chức năng; bổ sung căn cứ trang/mục; kiểm tra điều kiện lập, trình và duyệt bảng. | `QTKD_BA_Analysis.xlsx`, `01-requirements-summary.md` và phần yêu cầu trong ma trận truy vết. |
| 3 | 23127389 | **Tô Quốc Khánh** | Kiểm tra 8 vấn đề `ISS-01`–`ISS-08`: vị trí, loại, lý do cần xác nhận và cách xử lý; đối chiếu các quyết định với chuyển trạng thái của bảng. | Sheet `04_VanDe_CanXacNhan`, `07-design-decisions.md` và `diagrams/state-machine.puml`. |
| 4 | 23127370 | **Trần Huy Hoàng** | Rà soát kiến trúc, trách nhiệm các lớp và quan hệ giữa module thanh toán với các module nguồn; kiểm tra tính nhất quán của sơ đồ component và class. | `02-architecture-design.md`, `03-detailed-design.md`, `diagrams/component.puml` và `diagrams/class.puml`. |
| 5 | 23127479 | **Thái Chí Thạnh** | Rà soát nhóm API, quyền thao tác, mã lỗi và dữ liệu lưu trữ; kiểm tra chống trình duyệt trùng, audit log; đối chiếu luồng gọi API trong sequence diagram. | `04-api-contract.md`, `05-data-design.md` và `diagrams/sequence.puml`. |
| 6 | 23127320 | **Phạm Hoàng Anh** | Đánh giá từng nhận xét AI là đúng/sai/thừa và nêu căn cứ; so sánh bản đầu với bản sau review; kiểm tra yêu cầu được nối đến thiết kế và cách kiểm chứng. | `evaluation-report.md`, `06-traceability-matrix.md` và bảng đối chiếu hai phiên bản thiết kế. |
| 7 | 23127192 | **Nguyễn Văn Hộp** | Đối chiếu nhật ký với prompt và đầu ra thật; tập hợp minh chứng AI/nhóm sửa; biên tập slide 5 trang, kiểm tra chữ, hình và thời lượng nói của 7 người. | `docs/prompt-log.md`, minh chứng prompt và file slide trình bày. |

## 2. Phân chia phần thuyết trình

| Slide | Nội dung | Người trình bày |
| :---: | :--- | :--- |
| 1 | Giới thiệu nhóm, bài toán, phạm vi và quy trình dùng AI; tóm tắt yêu cầu nghiệp vụ. | **Nguyễn Quốc Bảo:** mở đầu và quy trình làm việc. **Hồ Đức Thuận:** phạm vi, chức năng và yêu cầu. |
| 2 | Các điểm chưa rõ trong đặc tả và phương án xử lý. | **Tô Quốc Khánh**. |
| 3 | Kiến trúc module và các trạng thái của bảng thanh toán. | **Trần Huy Hoàng**. |
| 4 | Nhóm API, bảng dữ liệu, các kiểm soát chính và ví dụ truy vết `PAY-07`. | **Thái Chí Thạnh:** API và dữ liệu. **Phạm Hoàng Anh:** ví dụ truy vết. |
| 5 | So sánh trước/sau review, đánh giá AI và minh chứng prompt. | **Phạm Hoàng Anh:** kết quả review. **Nguyễn Văn Hộp:** prompt và minh chứng. |

## 3. Trách nhiệm chung khi hoàn thiện và bảo vệ bài

- Mỗi thành viên kiểm tra phần được giao với PDF đặc tả và tài liệu trong workspace trước khi trình bày. Nội dung do AI tạo phải được con người đối chiếu; quyết định nghiệp vụ cần có căn cứ hoặc xác nhận của nhóm.
- Cả 7 thành viên cần nắm luồng `Draft → PendingApproval → Approved`, các nhánh `Rejected` và `Cancelled`, ngưỡng duyệt 500 triệu đồng, cách điều chỉnh bảng đã duyệt và lý do tồn tại hai phiên bản thiết kế.
- Khi vấn đáp cá nhân, mỗi thành viên tự giải thích được phần mình phụ trách và các nguyên tắc nghiệp vụ chung; phần thuyết trình được chia lượt nhưng kiến thức của đồ án là trách nhiệm của cả nhóm.


