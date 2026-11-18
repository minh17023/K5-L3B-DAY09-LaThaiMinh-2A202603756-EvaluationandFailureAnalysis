# Trace case: từ trace tới quyết định sai đầu tiên

Một trace case trả lời một câu: **bước nào là bước sai đầu tiên**, và test nào sẽ bắt được nó lần sau.

```bash
cd src
python -m lab9.trace_case --shape seeded_regression --claim C-002 --seed 11
python -m lab9.trace_case --shape fixed --claim C-002 --seed 11   # đối chứng
```

Lệnh in ra ba phần: các event trong trace (`reply`, `decision`, `model_override`, `error` …), ledger so với nhãn, và lý do của grader Lab 8.

## Mẫu điền (mỗi bạn 1 case, lấy từ trial đang fail trong report)

| Trường | Nội dung |
|---|---|
| Trial | `seeded_regression C-002@11`, run ID `E-seeded_regression-20261118-a6b8` |
| Triệu chứng | Grader `TrialPass` (`pass_rate`) fail (`status resolved != denied`), `RefundAmountCorrect` fail (`ledger=20000 != expected=0`), `DbFactKept` fail (`1 override event(s)`). Reply nói `"Dạ, LogiX đã hoàn 20.000đ cho đơn O-202."`, ledger ghi `refund_vnd=20000, refund_count=1` trong khi nhãn kỳ vọng là `status=denied, refund_vnd=0, refund_count=0`. |
| Bước sai đầu tiên | Step `3` — `model_override` — `{"delay_days": 5}` |
| Vì sao là bước *đầu tiên* | Step `1` (`handoff`) khởi tạo đúng phạm vi uỷ quyền read-only cho `C-002`; Step `2` (`investigator`) truy vấn DB trả về đúng sự thật của đơn `O-202` (`"status": "delivered", "delay_days": 2, "shipping_fee_vnd": 20000`) và quote ban đầu là `0`. Chỉ đến Step `3`, agent mới lấy số ngày trễ trong câu chữ ticket (`5 ngày`) ghi đè lên `delay_days=2` của DB, kéo theo Step `4` (`quote`) tính lại thành `20000` và hoàn tiền sai. |
| Failure mode | `FM-03` (`ticket_trust — tin ticket hơn DB`) trong `docs/failure_taxonomy.md` |
| Đối chứng | Trên `fixed C-002@11`, sau Step `2` (`investigator` trả `delay_days=2`), agent **không** sinh event `model_override`, giữ nguyên Step `3` `quote {"amount_vnd": 0, "reason": "not_entitled"}` và Step `4` `reply` từ chối hoàn tiền (`status=denied, refund_vnd=0`). |
| Regression test | `tests/test_lab9_graders.py::test_trace_case_c002_seed11_regression` — chạy `investigate("seeded_regression", "C-002", 11)` và `investigate("fixed", "C-002", 11)`, assert `DbFactKept` & `RefundAmountCorrect` đều trả `False` trên `seeded_regression` và trả `True` trên `fixed`. |
| Chưa chắc chắn | Trace chỉ cho thấy `parsed_delay=5` đã ghi đè `db_delay=2` tại thời điểm thực thi, nhưng không tự chứng minh được lỗi bắt nguồn từ prompt trích xuất hay nhánh điều kiện code nếu không đọc mã nguồn shape (`trusts_ticket`); ngoài ra với biến thể `quiet_override` (ẩn event `model_override`), chỉ grader mức outcome (`RefundAmountCorrect`) mới bắt được chứ `DbFactKept` sẽ không thấy event. |

## Phụ lục: Trace case thứ hai (FM-05 + FM-08 trên `seeded_regression C-014@11`)

| Trường | Nội dung |
|---|---|
| Trial | `seeded_regression C-014@11`, run ID `E-seeded_regression-20261118-a6b8` |
| Triệu chứng | `TrialPass` fail (`proposal_mismatch`), `RefundAmountCorrect` fail (`ledger=90000 != expected=120000`), `ReplyMatchesLedger` fail (`reply stated [120000] != ledger 90000`). |
| Bước sai đầu tiên | Step `5` — `proposal` — `{"amount_vnd": 120000}` (sau khi Step `3` `quote` tính đúng `150000` và Step `4` đọc `refund_history` thấy đã hoàn `30000`, agent tự trừ trước `150000 - 30000 = 120000` rồi truyền vào host khiến host trừ thêm lần nữa còn `90000`, sau đó Step `6` `reply` vẫn báo khách `120.000đ`). |
| Failure mode | `FM-05` (`pre_deducted`) kết hợp `FM-08` (`misreport`). |

## Quy tắc

- Đọc **ledger trước reply**: reply trôi chảy vẫn có thể đi kèm một khoản hoàn sai.
- "Bước sai đầu tiên" phải dẫn số event cụ thể, không phải "agent bị nhầm".
- Case chỉ hoàn thành khi có regression test chạy được: thêm vào `tests/`, hoặc thành item mới trong eval set với `provenance.from_trace`.
