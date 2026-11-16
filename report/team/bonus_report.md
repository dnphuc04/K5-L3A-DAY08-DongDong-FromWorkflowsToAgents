# Bonus report: B3

- Topic + nguồn: Verifier/aggregator chống khuếch đại lỗi; Google Research (2026), [Towards a science of scaling agent systems](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/), slide 17, 22, 42.
- CLAIM: kiểm chứng tập trung có thể hạn chế lỗi lan truyền giữa agent. Lab kiểm một cơ chế cụ thể, không tái lập hệ số khuếch đại lỗi của nghiên cứu.
- BOUNDARY: MockLLM và Swarm tham chiếu có lỗi cố ý theo seed; dữ liệu giả lập, không có model thật hoặc đo latency.
- HYPOTHESIS: thêm verifier bằng code sẽ sửa lỗi dùng độ trễ trong ticket, đưa pass rate/Pass^5 gần Workflow; chi phí vẫn cao hơn Workflow vì giữ các peer/model call.
- Biến cố định: corpus, năm seed 11/22/33/44/55, policy, giá, host, evaluator và luồng Swarm. Biến thêm: một bước kiểm dữ kiện trước quyết định cuối, gồm đọc lại order và tối đa một lần tính lại quote.

## Triển khai và tái lập

`src/bonus/swarm_verified.py` giữ luồng Swarm, gồm lỗi override theo ticket. Verifier đọc lại order theo principal, đối chiếu `order_id`, `delay_days`, `status`, `damage_verified`, phí ship và tiền đã trả. Nếu lệch, tính lại quote một lần, ghi `verifier_override` chỉ chứa tên field và vòng sửa. Kiểm cả quote bằng 0 để sửa được từ chối nhầm. Không sửa Swarm gốc, host hoặc thước đo.

`bonus/cases.jsonl` và `world.json` có 14 claim tự viết: ticket sai độ trễ/damage, nhiều số, injection, hủy đơn, approval thiếu/thấp, đã hoàn tiền, thiếu mã đơn, sai principal. Nhãn viết tay trong `bonus/build_cases.py`, không gọi quote hoặc đọc pack instructor để sinh nhãn.

Chạy trong `src/`, với Python 3.11–3.13 và dependencies của lab:

```powershell
python -m pytest -q tests/test_bonus_swarm_verified.py
python -m bonus.evaluate --extra-shape swarm_verified=bonus.swarm_verified:run_claim
```

Script dùng extra-shape và evaluator nguyên bản, mỗi trial một SQLite mới. Số liệu do script sinh trong `src/reports/bonus_b3_public.json`, `bonus_b3_adversarial.json`, `bonus_b3_safety.json`. Lệnh CLI độc lập và cách tạo lại fixture ở `src/bonus/README.md`.

## Kết quả

Mỗi dòng dưới đây có 70 trial (14 claim × 5 seed). Chi phí là VND mô phỏng trên mỗi trial thành công, bao gồm chi phí của trial fail.

| Corpus / shape | Pass rate | Pass^5 | Variance | Cost/success |
| --- | --- | --- | --- | --- |
| Public / Workflow | 1.000000 | 1.000000 | 0.000000 | 27.86 |
| Public / Swarm | 0.985714 | 0.928571 | 0.000816 | 49.57 |
| Public / Verified | 1.000000 | 1.000000 | 0.000000 | 57.57 |
| Đối kháng / Workflow | 1.000000 | 1.000000 | 0.000000 | 27.86 |
| Đối kháng / Swarm | 0.914286 | 0.571429 | 0.029388 | 54.94 |
| Đối kháng / Verified | 1.000000 | 1.000000 | 0.000000 | 60.37 |

<!-- pagebreak -->

## Can thiệp, trace và an toàn

Verifier kiểm 60 trial/corpus (10 trial dừng sớm vì thiếu order hoặc sai scope). Public có 1 lần override; đối kháng có 11. Tất cả can thiệp quan sát được là `delay_days`, đúng loại lỗi mà Swarm tham chiếu tạo ra. 11 override chỉ sửa 6 trial fail ở baseline; 5 can thiệp còn lại không đổi kết quả vì các điều kiện khác đã chặn/định số tiền.

Trace minh họa `R-swarm_verified-C-002-s55`: `model_override(delay_days=5)` → `verifier_check(changed_fields=[delay_days])` → `verifier_override(repair_round=1)` → `quote(amount_vnd=0)` → `denied`. C-002 có DB trễ 2 ngày. Trích đoạn event đã lọc được lưu trong safety report; trace đầy đủ ở `src/traces/` không commit.

Trên mỗi corpus và mỗi shape, 70 trial được quan sát: **approval bypass = 0, secret leak = 0, PII leak = 0, invalid trace = 0, unexpected ledger = 0**. Với Verified, **multiple repair = 0**. Các số đếm dùng nhãn approval và marker nhạy cảm của fixture, đối chiếu trace/ledger; không phải chứng minh mọi đầu ra đều an toàn.

Kiểm tra trên Windows/Python 3.12.14: **161 test B3 pass**, gồm 70 public, 70 đối kháng, kiểm riêng từng field, không sửa khi dữ kiện khớp, scope, approval và crash/resume. Bộ test bơm cả status/damage sai trực tiếp vào verifier; không tuyên bố MockLLM đã tự tạo hai loại lỗi này. Ba điểm crash × ba claim có refund, sau đó resume/retry bằng seed khác đều giữ một effect và một audit committed. Toàn bộ suite cùng student: **590 passed** (429 cũ + 161 mới).

## Kết luận

Kết quả ủng hộ giả thuyết trong phạm vi đo: Verified đạt độ đúng bằng Workflow trên hai corpus và sửa được cả hoàn thừa lẫn từ chối nhầm của baseline. Tuy nhiên cost/success Verified cao hơn Workflow khoảng **2,07 lần trên public**, **2,17 lần trên đối kháng**. Việc kiểm lại tạo thêm một tool read cho claim có order hợp lệ và một quote call khi sửa; peer/model call vẫn được giữ.

Với task có luật và dữ kiện có cấu trúc này, swarm chưa thể hiện giá trị bổ sung về độ đúng sau khi code verifier đưa quyết định về nguồn dữ liệu chuẩn. Nhóm giữ **Workflow** làm shape chính. Cần đánh giá lại nếu có bước đọc chứng từ phi cấu trúc hoặc điều tra độc lập mà peer giúp cải thiện chất lượng đủ bù chi phí; không suy ra multi-agent luôn kém hiệu quả.

## Giới hạn và điều kiện tính bonus

MockLLM, lỗi theo seed cố ý và 14 case do nhóm thiết kế làm kết quả thiên về lỗi đã biết. Giá mô phỏng không phải hóa đơn API; latency là `unavailable`. Hai corpus không chứng minh hidden pass. Verifier chỉ kiểm dữ kiện, không phải oracle độc lập cho lỗi policy/quote khi input khớp; đọc lại cũng chưa bảo đảm snapshot trước mọi chỉnh sửa đồng thời. Test crash không thay suite concurrency đầy đủ.

Bonus chỉ được tính khi shape chính qua ba must-pass, hidden pass rate **≥ 0.80** và host test xanh. Tự kiểm student_drop không nhãn chưa xác nhận gate hidden; báo cáo không tự nhận đủ +10 điểm.
