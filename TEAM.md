# Team

## Thông tin nhóm

- Tên nhóm: DongDong
- Lớp: L3A
- Repo nhóm: `K5-L3A-DAY08-DongDong-FromWorkflowsToAgents` (xem [SUBMISSION.md](SUBMISSION.md))
- URL repo, nhánh nộp, commit chốt: https://github.com/dnphuc04/K5-L3A-DAY08-DongDong-FromWorkflowsToAgents

## Thành viên

| STT | Họ tên | MSSV | Email | GitHub | Vai trò |
| --: | ------ | ---- | ----- | ------ | ------- |
|   1 |    Đồng Ngọc Phúc    |  2A202603399    |    dnphuc04@gmail.com   |    dnphuc04    | Workflow + tích hợp + chốt bài nộp |
|   2 |   Nguyễn Trần Tiến     |   2A202603504   |    nguyentien2004abc@gmail.com   |    tiennguyen-ctrl    | Single agent + ReAct + budget |
|   3 |   Nguyễn Việt Hưng    |   2A202603411   | robinnguyen2004@gmail.com      | robinnguyen2004       | Orchestrator + handoff + trace an toàn |
|   4 |    Nguyễn Huy Hoàng    |      |       |        | Kiểm thử policy/durability + benchmark + đánh giá |

## Phân công thực hiện

Đây là kế hoạch phân công, chưa phải xác nhận công việc đã hoàn thành. Mỗi người tự bổ sung commit và bằng chứng thật sau khi làm xong; không dùng chung một commit để thay cho đóng góp cá nhân.

| Thành viên | Phần việc chính | Test cần thêm hoặc sửa | Sản phẩm và tiêu chí bàn giao |
| --- | --- | --- | --- |
| Đồng Ngọc Phúc | Viết `src/lab8/shapes/workflow.py`; tích hợp ba shape; giữ cổng host xanh; tổng hợp báo cáo nhóm và checklist nộp | `src/tests/test_team_workflow.py`: kiểm tra ticket mâu thuẫn dữ kiện đơn không làm đổi quote, và thiếu order không ghi refund | Workflow qua 62 test student; dùng quote đầy đủ, `finish()` và truyền `crash_at`; tổng hợp `group_report.md`, thông tin nhóm và commit chốt sau khi có kết quả |
| Nguyễn Trần Tiến | Viết `src/lab8/shapes/single.py` bằng `run_react`, tối đa 8 bước; chỉ cho model chọn tool đọc; tính đủ budget | `src/tests/test_team_single.py`: kiểm tra giới hạn vòng lặp và model không thể chọn tool ghi refund | Single qua 62 test student; tiền do code tính, host ghi; viết phần Single và ranh giới code/model trong mục 3 báo cáo nhóm |
| Nguyễn Việt Hưng | Viết `src/lab8/shapes/orchestrator.py`; dựng handoff bằng `make_handoff`; investigator chỉ đọc; trace theo allowlist | `src/tests/test_team_orchestrator.py`: kiểm tra handoff loại secret và tool ghi, trace không lộ dữ liệu nhạy cảm | Orchestrator qua 62 test student; viết phần Orchestrator, handoff và trace trong mục 3 báo cáo nhóm |
| Nguyễn Huy Hoàng | Kiểm thử tích hợp approval, trần theo đơn, idempotency, crash; chạy benchmark bốn shape và tự kiểm hidden khi có dữ liệu; phân tích số liệu | `src/tests/test_team_safety.py`: kiểm tra approval sai claim và crash/resume không tạo refund trùng trên ba shape | Sinh `src/reports/crash.json` đạt 20/20 và `benchmark.json` đủ bốn shape, năm seed; soạn `decision_adr.md`, `research_card.md`, mục 4 và 7 báo cáo nhóm từ số liệu thật |

### Thứ tự phối hợp

1. **CP1:** Phúc kiểm tra smoke và host test; cả bốn đọc business rules/contract, tự cài môi trường. Hoàng bổ sung MSSV; các thành viên bổ sung thông tin liên hệ còn thiếu.
2. **CP2:** Phúc, Tiến, Hưng triển khai ba shape trong ba file riêng. Hoàng chuẩn bị test an toàn và lệnh đánh giá; không sửa host, policy, dữ liệu hay config.
3. **CP3:** Mỗi chủ shape xử lý idempotency/crash của shape mình. Hưng kiểm tra handoff/trace; Hoàng kiểm thử chéo. Phúc tích hợp và chạy lại cổng host.
4. **CP4:** Hoàng chạy crash suite và kiểm approval/secret; lỗi thuộc shape nào giao chủ shape đó sửa. Chỉ bàn giao khi test liên quan xanh.
5. **CP5:** Hoàng chạy benchmark và soạn phân tích; cả nhóm cùng thống nhất shape từ bằng chứng. Phúc tổng hợp báo cáo, chạy `check_submission` và chốt bài nộp.

### Quy ước bàn giao

- Mọi lệnh Python chạy trong `src/`. Mỗi chủ shape chạy `python -m pytest -q -m student -o addopts= -k <shape>` trước khi bàn giao; khi tích hợp chạy toàn bộ student suite và `python -m pytest -q`.
- Test nhóm bổ sung phải kiểm tra hành vi hoặc ranh giới an toàn có ý nghĩa, không đổi nhãn/thước đo để test xanh. Tên file test trên là kế hoạch, chỉ ghi vào bảng đóng góp khi thực sự đã tạo và chạy.
- Không sửa `metrics.py`, `durability.py`, `policy.py`, `src/config/*.yaml`, `src/data/*`; không sửa tay JSON report. Báo lỗi host cho cả nhóm thay vì tự sửa thước đo.
- Mỗi người có commit kỹ thuật, ít nhất một test đã thêm hoặc sửa, và báo cáo cá nhân theo mẫu `report/team/MSSV_HoTen.md`. Không ghi trước kết quả, run id hoặc hash chưa có.
- Phúc điều phối bài nộp chung; từng thành viên tự nộp link repo lên LMS.

## Bảng đóng góp

`python -m lab8.check_submission` đọc bảng này bằng máy: giữ nguyên tiêu đề `## Bảng đóng góp` và các cột. Mỗi người một dòng: commit có nội dung và ít nhất một test đã thêm hoặc sửa.

| Người | Vai | Commit | Test đã thêm hoặc sửa |
| --- | --- | --- | --- |
| Đồng Ngọc Phúc | Workflow + tích hợp | [dd59b0e — Completed CP2](https://github.com/dnphuc04/K5-L3A-DAY08-DongDong-FromWorkflowsToAgents/commit/dd59b0eef515dfabec26b271f743df483b00eb79) | `src/tests/test_team_workflow.py`: `test_workflow_uses_order_facts_not_ticket` (4 ca), `test_workflow_does_not_infer_missing_order_from_ticket`, `test_workflow_unknown_claim_returns_failed_trace`; 6 ca đạt; student workflow 62/62 đạt |
| Nguyễn Trần Tiến | Single agent + ReAct | | |
| Nguyễn Tiến Hưng | Orchestrator + handoff + trace | | |
| Nguyễn Huy Hoàng | Kiểm thử an toàn + đánh giá | | |

## Cá nhân

Mỗi thành viên đồng thời tạo một báo cáo riêng trong `report/` theo dạng:

```text
report/team/MSSV_HoTen.md
```

Ví dụ: `report/team/20241234_NguyenVanAn.md`.

Sao chép mục dưới đây cho từng thành viên.

## MSSV_HoTen

### Phần việc phụ trách

-

### Đóng góp cụ thể

-

### Bằng chứng

-
