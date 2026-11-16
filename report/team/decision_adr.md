# Decision record

Điền sau dấu `:` trên cùng dòng, không xóa nhãn. Giảng viên và `python -m lab8.check_submission` (chạy trong `src/`) đọc các dòng này bằng máy. Không đổi tên hay dời file này khỏi `report/team/`.
`Shape chọn` phải là đúng một trong: `workflow`, `single`, `orchestrator`, `swarm`. Đây là shape được chấm cổng hidden.

- Shape chọn: workflow
- Commit hash: 0f13675 (code, test và `src/reports/*.json` được sinh ở commit này)
- Run id dẫn chứng: `R-workflow-C-001-s11` … `R-workflow-C-014-s55` (70 trial trong `src/reports/benchmark.json`); đối chứng lỗi swarm `R-swarm-C-002-s55`
- Pass rate / pass^5 / seed min-max / variance: 1.000 / 1.000 / 1.000–1.000 / 0.000000 (14 claim × 5 seed, `src/reports/benchmark.json`)
- Cost per success (mô phỏng, VND): 27.86 (single 29.97, swarm 49.57, orchestrator 56.65)
- Crash safety: 20/20 (`src/reports/crash.json`); thêm 30 ca crash/resume trên ba shape trong `src/tests/test_team_safety.py`
- Secret leak / approval bypass: 0 / 0 (benchmark 70/70 trial workflow pass, không có `forbidden_substring`/`leak_suspect`; test approval sai claim, thiếu, không tồn tại, hết hạn đều không có refund)

Ví dụ một dòng đã điền: `- Shape chọn: workflow`, `- Crash safety: 20/20 (src/reports/crash.json)`.

## Vì sao shape này

Trên cùng corpus public (14 claim), cùng 5 seed `11,22,33,44,55`, cùng bảng giá mô phỏng và mỗi trial một SQLite riêng, ba shape của nhóm đều đạt pass rate 1.000, pass^5 1.000 và variance 0, còn swarm tham chiếu đạt 0.986 / 0.929 với seed 55 chỉ 0.929 (C-002 bị `resolved` thay vì `denied` vì tin số ngày trễ trong ticket). Khi chất lượng ngang nhau, chi phí và độ tự chủ quyết định: workflow rẻ nhất (27.86 VND/success, 0 model call, 2.79 tool call, 3.79 bước trung bình), single đắt hơn 7.6% vì 3.79 model call mỗi trial, orchestrator đắt gấp 2.03 lần vì đọc lại claim và chạy investigator (5.57 tool call). Bài toán claim là tuần tự và quyết định hoàn toàn bằng luật (dữ kiện đơn + policy + approval), nên không có bước nào cần model suy luận để ra quyết định. Workflow là kiến trúc ít tự chủ nhất mà vẫn làm đủ việc, và cũng có ít bề mặt nhất cho lỗi tin ticket hay rò dữ liệu qua handoff.

## Có đáng đổi lấy multi-agent không

Không. Trên bộ public, orchestrator không tăng pass rate nào (1.000 = 1.000) nhưng tăng cost/success từ 27.86 lên 56.65 VND; swarm còn tệ hơn workflow ở cả pass^5 (0.929) lẫn variance (0.000816). Kể cả nếu một shape multi-agent nhỉnh hơn vài điểm pass, nhóm vẫn giữ workflow khi chênh lệch nằm trong mức dao động seed và cost cao hơn. Quyết định sẽ đảo lại khi: (1) bộ hidden có loại claim mà luật cố định không bao được (ví dụ phải đọc chứng từ tự do để xác minh hư hỏng) và workflow rơi dưới 0.75 trong khi shape khác vượt; (2) có bước điều tra độc lập, song song được, khiến investigator giảm latency thật (hiện latency `unavailable`); hoặc (3) shape agent đạt pass^5 cao hơn rõ rệt với variance không tăng. Khi đó nhóm cân nhắc single (cost chỉ +7.6%) trước orchestrator.

## Trace và bằng chứng

Trace của workflow ghi các event `claim` (chỉ `found`), `order` (chỉ `visible`, `reason`), `quote` (kết quả `compute_compensation`) và `decision` (status, số tiền đề xuất, lý do host). Trace cố ý **không** ghi ticket (dữ liệu khách tự viết, có thể chứa canary hoặc lệnh chèn), claim/order row thô, `parent_secret`, `raw_headers`, SĐT, địa chỉ, ghi chú nội bộ và tên hàng: grader và đội vận hành đọc trace, nên mọi field không cần cho việc kiểm toán quyết định đều là rủi ro rò rỉ. Ranh giới này được kiểm bằng benchmark (`forbid_substrings` trên 70 trial) và bằng test nhóm so với giá trị nhạy cảm thật lấy từ dữ liệu. Nếu trace nằm ngoài chỗ scanner quét được, đội security không dựa vào việc "quét không thấy" mà chứng minh bằng thiết kế: (1) shape chỉ đọc claim qua `get_claim_view`, không bao giờ cầm `parent_secret`/header trong bộ nhớ; (2) mọi payload vào trace là field allowlist chọn tay trong code, không dump object; (3) test âm tính cố ý ghi claim row vào trace phải làm test fail (đã thử với orchestrator: 29/47 ca fail); (4) ghi bằng chứng ledger/audit trong SQLite thay vì log tự do.

## Giới hạn

MockLLM không phải model sản xuất. Latency để `unavailable`. Không sửa JSON cho đủ 0.75. Nhóm không nhận được `student_drop/` nên chưa chạy tự kiểm hidden; kết luận chỉ dựa trên 14 claim public, nơi ba shape của nhóm đều bão hòa ở 1.000, nên benchmark public không phân biệt được chất lượng giữa workflow, single và orchestrator — chỉ phân biệt được cost và độ tự chủ.
