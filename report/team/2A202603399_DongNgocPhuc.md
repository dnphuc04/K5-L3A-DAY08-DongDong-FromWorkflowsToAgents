# Báo cáo cá nhân — 2A202603399_DongNgocPhuc

## 1. Thông tin thành viên

- Họ tên: Đồng Ngọc Phúc
- MSSV: 2A202603399
- Email: dnphuc04@gmail.com
- GitHub: dnphuc04
- Nhóm: DongDong — L3A

## 2. Phần việc phụ trách

Phụ trách shape A Workflow, test tương ứng, phối hợp tích hợp code và tổng hợp báo cáo/chốt bài nộp. Đã hoàn thành workflow, test riêng, phần Workflow trong báo cáo nhóm, bằng chứng đóng góp và kiểm tra tích hợp sau khi đủ ba shape. Đã bổ sung các mục tổng quan, khó khăn, tài liệu tham khảo và kết quả kiểm tra mới vào báo cáo nhóm. Việc push/merge cập nhật tài liệu cuối, xác minh môi trường Python theo yêu cầu và nộp LMS là các bước chốt còn lại; không coi là đã nộp bài.

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

### Kiểm tra tích hợp mới nhất

Tại commit `a8a9835`, Orchestrator đã hoàn thành và đủ báo cáo/test của bốn thành viên. Các lệnh thực tế chạy từ `src/`:

```powershell
../.venv/Scripts/python.exe -m lab8.smoke
../.venv/Scripts/python.exe -m pytest -q --tb=short --basetemp=.pytest_cache/final-host-03
../.venv/Scripts/python.exe -m pytest -q -m student -o addopts= --tb=short --basetemp=.pytest_cache/final-student-03
../.venv/Scripts/python.exe -m lab8.check_submission
```

Kết quả: smoke pass; host và test nhóm **181 passed**; student toàn bộ **248 passed** (62 ca mỗi shape); `check_submission` **0 FAIL, 0 WARN**. Mốc này cập nhật kết quả tích hợp mới nhất; các kết quả 60/79 và 186 pass/62 fail phía trên được giữ như lịch sử từng giai đoạn.

Đối chiếu `src/reports/benchmark.json` và ADR do Hoàng cung cấp ở commit `0f13675`: workflow đạt 70/70 trial public, pass rate và pass^5 đều 1.000, variance 0, cost/success mô phỏng 27.857142857 VND (làm tròn 27.86), không gọi model. Run id ví dụ: `R-workflow-C-001-s11`. `src/reports/crash.json` ghi 20/20. Đây là bằng chứng thực nghiệm của nhóm đã được kiểm tra cấu trúc bởi `check_submission`, không phải benchmark được cá nhân chạy lại trong lần kiểm tra tích hợp này. Chưa có kết quả hidden; không khẳng định đã đạt ngưỡng hidden.

Môi trường kiểm tra là Windows, Python 3.14.6. Lần chạy pytest ban đầu gặp `PermissionError` với thư mục tạm mặc định; dùng `--basetemp` riêng trong `.pytest_cache` đã chạy được. Cần xác minh lại trên Python 3.11–3.13 theo yêu cầu lab. Không có kết quả CI được xác minh trong lần kiểm tra local này.

## 5. Công cụ AI đã dùng

Đã dùng OpenAI Codex để đọc đề và contract, hỗ trợ triển khai workflow/test, chạy kiểm thử, kiểm tra lịch sử commit và soạn báo cáo. Các khẳng định về hành vi được đối chiếu bằng test tự động, ledger/audit trong bộ test và source code. Commit được lấy từ lịch sử Git; không tự đặt hash, run id hoặc số liệu benchmark. Thành viên cần tự đọc lại code, chạy lại các lệnh trên và giải thích được quote đầy đủ, ranh giới host và các field bị loại khỏi trace khi bảo vệ bài.

## 6. Tự đánh giá

Phần triển khai Workflow ở CP2 và kiểm tra retry/crash của Workflow ở CP3 đã hoàn thành theo ma trận public. Tại mốc tích hợp mới nhất, toàn bộ host/test nhóm và student đều xanh. Ưu điểm của workflow là luồng cố định, không phụ thuộc quyết định model và không lấy ticket làm nguồn quyết định số tiền. Benchmark public của nhóm cho thấy workflow có cùng pass rate với single/orchestrator và cost mô phỏng thấp hơn, phù hợp lựa chọn trong ADR. Kết luận chỉ áp dụng cho corpus và môi trường mô phỏng đã đo, chưa chứng minh pass rate hidden hoặc hiệu năng trên model thật.

Phần tích hợp và tổng hợp nội dung báo cáo ở CP5 đã được cập nhật theo bằng chứng hiện có. Báo cáo này không tự chấm điểm cá nhân và không xác nhận thay kết quả kiểm tra của giảng viên.

### Rà soát chốt CP5 sau commit 526a6ae

Đã xác minh lại bằng Python **3.12.14** trên Windows: toàn bộ **429 test pass** (181 host/test nhóm + 248 student), benchmark bốn shape tái lập các chỉ số đã báo cáo, crash suite **20/20**. Dùng Python runtime Codex, pytest 8.4.2 và PyYAML 6.0.3 từ `.venv/Lib/site-packages` qua `PYTHONPATH`; venv 3.14 của người dùng được giữ nguyên. Đây là kiểm tra trên phiên bản hợp lệ, chưa phải kiểm cài sạch hoặc nhiều hệ điều hành.

Lệnh test: `python -m pytest -q -o addopts= --tb=short --basetemp=.pytest_cache/cp5-py312-02`, chạy trong `src/` sau khi chọn Python 3.12 và dependencies. Benchmark/crash chạy lại vào `.pytest_cache/cp5_benchmark_py312.json` và `.pytest_cache/cp5_crash_py312.json`, không sửa report JSON bằng tay. Lệnh đầy đủ và giới hạn được ghi trong mục 5 báo cáo nhóm.

Đã rút gọn báo cáo nhóm theo năm phần trang A4 và thống nhất ADR/báo cáo nhóm ở commit chốt `526a6ae2df4653f6c59f62bbdf1d4a7cef6efb2c`; nguồn JSON gốc vẫn là `0f13675`. Cần hoàn tất commit/push bản rà soát, kiểm tra nộp lần cuối trên cây sạch và từng thành viên tự nộp link LMS. Hidden chưa được cấp, không báo đạt ngưỡng hidden.
