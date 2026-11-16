# Báo cáo cá nhân — 2A202603399_DongNgocPhuc

## 1. Thông tin thành viên

- Họ tên: Đồng Ngọc Phúc
- MSSV: 2A202603399
- Email: dnphuc04@gmail.com
- GitHub: dnphuc04
- Nhóm: DongDong — L3A

## 2. Phần việc phụ trách

Phụ trách shape A Workflow, test tương ứng, phối hợp tích hợp code và tổng hợp báo cáo/chốt bài nộp. Phần đã hoàn thành gồm workflow, test riêng và kiểm tra sau merge. Phần tích hợp cuối và chốt bài nộp còn phụ thuộc việc hoàn thành Orchestrator, benchmark, ADR và các báo cáo của nhóm.

## 3. Đóng góp cụ thể

Commit kỹ thuật: [dd59b0e — Completed CP2](https://github.com/dnphuc04/K5-L3A-DAY08-DongDong-FromWorkflowsToAgents/commit/dd59b0eef515dfabec26b271f743df483b00eb79), gồm `src/lab8/shapes/workflow.py` và `src/tests/test_team_workflow.py`.

- Thay stub Workflow bằng luồng cố định: đọc claim, kiểm tra mã đơn và quyền xem đơn, tính quote, gọi host và trả kết quả theo contract.
- Dùng dữ kiện đơn và `compute_compensation` để quyết số tiền; không gọi model hoặc tin lệnh/số ngày ghi trong ticket.
- Truyền quote đầy đủ cho host. Không tự trừ số tiền đã hoàn vì host áp trần theo đơn; tự trừ trước có thể khiến phần chênh bị trừ hai lần.
- Giữ `proposal_amount` là quote ngay cả khi host cap số tiền thực ghi hoặc từ chối. Dùng `finish()` để lấy effect và audit từ ledger.
- Truyền `crash_at` và để `InjectedCrash` truyền ra ngoài; đóng kết nối bằng `finally`. Cơ chế chống refund trùng do host/ledger đảm nhiệm.
- Chỉ ghi trace tối thiểu: kết quả tìm claim, quyền xem đơn, quote và quyết định. Không ghi ticket hoặc dữ liệu thô chứa thông tin nhạy cảm.
- Bổ sung 6 ca test hồi quy và phần mô tả Workflow trong báo cáo nhóm; điền bằng chứng đóng góp trong `TEAM.md`.

## 4. Bằng chứng

Test đã thêm trong `src/tests/test_team_workflow.py`:

| Test | Hành vi được kiểm tra |
| --- | --- |
| `test_workflow_uses_order_facts_not_ticket` | 4 ca: ticket phóng đại/phủ nhận độ trễ tại seed 11 và 55; trạng thái, quote, effect và audit theo dữ kiện đơn; không gọi model |
| `test_workflow_does_not_infer_missing_order_from_ticket` | Thiếu mã đơn có cấu trúc vẫn trả `needs_info` dù ticket ghi mã đơn; tổng ledger không đổi; trace đúng run id và không chứa canary/ticket |
| `test_workflow_unknown_claim_returns_failed_trace` | Claim không tồn tại trả `failed`, không có effect, có trace đúng run id |

Kiểm tra đã thực hiện với hỗ trợ Codex, mọi lệnh chạy từ `src/`:

```powershell
python -m pytest -q -m student -o addopts= -k workflow --tb=short --basetemp=.pytest_cache/phuc-workflow-01
python -m pytest -q --tb=short --basetemp=.pytest_cache/phuc-host-01
```

Kết quả khi hoàn thành workflow: student workflow **62 passed**; host và 6 ca mới **60 passed**. Ma trận workflow có kiểm tra T11 gọi lại và T12 crash/resume.

Sau khi merge Single vào `main`, tại commit `3108b05590af2c851467a93a781f73ec78fb3612`, chạy smoke, host và toàn bộ student suite:

```powershell
python -m lab8.smoke
python -m pytest -q --tb=short --basetemp=.pytest_cache/merged-host-02
python -m pytest -q -m student -o addopts= --tb=no -rN --basetemp=.pytest_cache/merged-student-02
```

Smoke pass; host và test nhóm **79 passed**; student **186 passed, 62 failed**. Workflow, Single và Swarm đạt; Orchestrator còn stub. Đây là mốc kiểm tra sau merge, không phải xác nhận toàn bộ bài đã hoàn thành. Các lệnh thực tế dùng trực tiếp executable `../.venv/Scripts/python.exe`, tương đương `python` khi kích hoạt đúng môi trường.

Môi trường kiểm tra là Windows, Python 3.14.6. Lần chạy pytest ban đầu gặp `PermissionError` với thư mục tạm mặc định; dùng `--basetemp` riêng trong `.pytest_cache` đã chạy được. Cần xác minh lại trên Python 3.11–3.13 theo yêu cầu lab. Chưa có số liệu benchmark/hidden hoặc run id benchmark để báo cáo trong phần cá nhân này.

## 5. Công cụ AI đã dùng

Đã dùng OpenAI Codex để đọc đề và contract, hỗ trợ triển khai workflow/test, chạy kiểm thử, kiểm tra lịch sử commit và soạn báo cáo. Các khẳng định về hành vi được đối chiếu bằng test tự động, ledger/audit trong bộ test và source code. Commit được lấy từ lịch sử Git; không tự đặt hash, run id hoặc số liệu benchmark. Thành viên cần tự đọc lại code, chạy lại các lệnh trên và giải thích được quote đầy đủ, ranh giới host và các field bị loại khỏi trace khi bảo vệ bài.

## 6. Tự đánh giá

Phần triển khai Workflow ở CP2 và kiểm tra retry/crash của Workflow ở CP3 đã hoàn thành theo ma trận public. Ưu điểm là luồng cố định, không phụ thuộc quyết định model và không lấy ticket làm nguồn quyết định số tiền. Kiểm thử public chưa chứng minh pass rate hidden hoặc ưu thế cost so với các shape khác; cần benchmark cùng corpus và seed trước khi chọn kiến trúc.

Trách nhiệm tích hợp cuối, tổng hợp báo cáo và chốt bài ở CP5 chưa hoàn tất tại mốc kiểm tra nêu trên. Báo cáo này không tự chấm điểm cá nhân và không xác nhận thay kết quả kiểm tra của giảng viên.
