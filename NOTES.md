# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Hoàng Ân
MSSV: 2A202604025

## 1. Case được chọn

- ID case: **H6** (scenario `noisy_status`). Câu hỏi: "Xem status GitHub Actions, tìm incident h02n9572mx9y và theo liên kết của nó. Incident gộp có nêu nguyên nhân hay chỉ mô tả ảnh hưởng? Nêu giới hạn dữ liệu và trích nguồn."
- Câu trả lời của agent (lần 1, trích): "Theo incident gộp đó (xr3myzj7ms51), sự cố mô tả ảnh hưởng đến Actions và Pages và đã được giải quyết, nhưng **không nêu rõ nguyên nhân cụ thể** mà chỉ tập trung mô tả ảnh hưởng và trạng thái xử lý." Nguồn được liệt kê: `githubstatus.com/incidents/h02n9572mx9y`, `githubstatus.com/incidents/xr3myzj7ms51`, `githubstatus.com`.
- Điểm ở lần chạy 1: **60** (judge chấm thô 90: factuality 37, completeness 20, evidence 18, boundaries 15; bị cap 60 bởi mechanical check "Cần tool calls ở ít nhất 2 model steps khác nhau").

## 2. Bằng chứng trong trace

Trace H6 trong `eval_before.json` (rút gọn 40 mục nhiễu):

```text
[model] step 1 · finish=tool_calls · 2 tool calls
[tool]  step 1 · get_status {"provider": "github", "component": "GitHub Actions"}
        -> overall: none / All Systems Operational · components: [] · incidents: t89g2q1800fr,
           xr3myzj7ms51 "Incident with Actions and Pages", ... · 40 distractors
[tool]  step 1 · get_incident_details {"provider": "github", "incident_id": "h02n9572mx9y"}
        -> ok · resolved · updates 3/4 · url https://stspg.io/7pwv28hvkmz8
           update mới nhất: "Please refer to the combined summary in this related incident:
           https://www.githubstatus.com/incidents/xr3myzj7ms51"
[model] step 2 · finish=stop · 0 tool calls -> câu trả lời ở mục 1
[grade] score 60 · raw 90 · mechanical: tool_calls 2, tool_steps 1
```

Lỗi: ở step 2 agent đã thấy link tới incident gộp `xr3myzj7ms51` nhưng không đọc mà trả lời luôn. Không có tool result nào chứa nội dung của xr3myzj7ms51, vậy mà câu trả lời khẳng định incident này "không nêu rõ nguyên nhân". Chi tiết "ảnh hưởng Actions và Pages" chỉ suy từ tiêu đề trong danh sách của `get_status`. Khẳng định này **sai**: update `resolved` của xr3myzj7ms51 trong snapshot ghi rõ "This incident was caused by a configuration change to an internal event-routing service ... depended on feature flags that were not set up as intended". `get_incident_details('github', 'xr3myzj7ms51')` vẫn trả câu này dù update bị cắt ở 900 ký tự (`truncated: true`). Ngoài ra, cả ba URL nguồn đều không phải URL do tool trả về.

Nguyên nhân có thể: loop không có cơ chế nào ràng buộc "khẳng định phải có observation". System prompt chỉ nói "Dẫn nguồn từ tool", nên model có thể dừng sớm và lấp khoảng trống bằng suy đoán nghe hợp lý. Lỗi nằm ở loop/prompt, không nằm ở dữ liệu, vì tool đọc được incident gộp.

Judge không phát hiện lỗi: factuality 37/40, phần reason còn khen agent "chỉ rõ nguyên nhân không được nêu". Lý do là judge chỉ thấy trace, mà trace không chứa nội dung xr3myzj7ms51 để đối chiếu.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: thêm kiểm tra nguồn cho final answer trong `agent.py`, gồm hàm `_check_sources` cùng phần ghi nhận trong `run_agent`.
  - Lưu `observed = {incident_id: url}` từ mỗi `get_incident_details` có `ok is True`. Lưu `linked`, là các incident ID được dẫn link (`/incidents/<id>`) trong update của incident đã đọc.
  - Khi model trả lời với `finish == 'stop'`, kiểm tra hai lớp:
    1. Nếu đã đọc được incident mà câu trả lời không chứa URL của nó (shortlink hoặc `/incidents/<id>`), ghi event `validation_missing_source`.
    2. Nếu còn incident trích trong câu trả lời hoặc nằm trong `linked` mà chưa đọc, ghi event `validation_unverified_source`.
  - Khi vi phạm, agent ghi event vào trace và thêm một message nhắc model đọc incident còn thiếu hoặc dẫn nguồn, rồi viết lại câu trả lời hoàn chỉnh cho câu hỏi ban đầu. Lần sửa vẫn tính vào `max_steps`, tối đa `MAX_REVISIONS = 2` lần. Quá giới hạn thì agent chấp nhận câu trả lời và ghi `validation_gave_up`.
  - Không sửa system prompt, tools, provider, dataset hay grading.
- Vì sao chọn thay đổi này: nó nhắm thẳng vào nguyên nhân gốc là "khẳng định về incident chưa được đọc", đồng thời hoàn thành contract kiểm tra nguồn trong `tests/test_validation.py`. Đây là kiểm tra bằng code, chạy ổn định giữa các lần chạy, nên tôi chọn nó thay vì chỉ thêm chỉ dẫn vào prompt. Tôi không sửa H7 bằng prompt cùng lúc, để điểm tăng chỉ quy cho một thay đổi.
- Quá trình thử bằng `--case` trước khi chạy eval lần 2:
  - Bản đầu chỉ bắt incident được nhắc trong câu trả lời. H6 lên 94, nhưng bản viết lại làm mất ý status Actions và có câu kiểu "Cập nhật trả lời...". Tôi bổ sung `REWRITE_HINT` yêu cầu viết lại câu trả lời hoàn chỉnh và không nhắc tới bước kiểm tra.
  - Ở lần chạy kế tiếp H6 rơi về 60: model chép sai ID thành `xr3myzjms51`, nên kiểm tra theo chuỗi không bắt được. Tôi đổi sang quy tắc "incident đã đọc dẫn link tới incident khác thì phải đọc incident đó". Trong snapshot chỉ có `h02n9572mx9y` dẫn link tới `xr3myzj7ms51`, và `qpfv5p86dmrl` dẫn link tới `djlmxz2zd0j7`. Incident h02n chỉ xuất hiện ở A6, H1, H6, cả ba đều yêu cầu theo link.
- Tests: `python3 -m unittest discover -s tests -v` pass 37/37.

## 4. So sánh hai lần chạy

Cùng cấu hình: agent `gpt-4.1-mini`, judge `gpt-4.1-mini`, `max_steps` 8, dataset core, snapshot 09/11/2026 (`config` hai file giống nhau).

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 90.59 | 96.50 |
| Điểm case đã chọn (H6) | 60 (raw 90) | 98 (raw 98) |
| Số model calls (agent, 22 case) | 47 | 55 |
| Token agent (input / output) | 26,188 / 5,660 | 44,656 / 8,680 |
| Token judge (input / output) | 35,373 / 3,617 | 43,523 / 3,610 |
| Chi phí | Provider không trả chi phí trong `usage` | như trên |

Điểm theo case thay đổi: H6 60→98, H1 60→100, H2 50→100, E5 93→100, H7 45→50, A1 100→96, A6 96→93, A5 94→92, A7 95→94. Các case còn lại giữ nguyên.

- Lỗi đã giải quyết:
  - **H6**: trace mới có `validation_missing_source` (step 2), rồi `validation_unverified_source` (step 3), rồi `get_incident_details(xr3myzj7ms51)` ở step 4. Câu trả lời cuối nêu đúng nguyên nhân (thay đổi cấu hình ở event-routing service, feature flags), dẫn đúng URL tool trả về (stspg.io/7pwv28hvkmz8, stspg.io/pjgptg3cfc6n), không còn mechanical violation. Ở step 2, model đã tự nói "không thể khẳng định" thay vì bịa, nhưng vẫn không đọc incident gộp cho tới khi bị nhắc.
  - **H1**: `validation_unverified_source`, rồi đọc xr3myzj7ms51 ở step 3, đạt 100 và hết cap 60.
  - Phần đóng góp của thay đổi: H1 và H6 cộng 78 điểm case, tương đương khoảng **+3.5** điểm tổng. Khoảng **+2.4** còn lại đến từ các case không có event validation nào (H2 +50, E5 +7, H7 +5, A1 −4, A6 −3, A5 −2, A7 −1), tức là dao động giữa hai lần chạy. Ví dụ H2: lần 2 model tự gọi thêm `get_status` nên hết vi phạm `requires_tools`. Thay đổi của tôi không tác động vào case này.
- Lỗi còn lại hoặc mới xuất hiện:
  - **H7 (50)**: Cloudflare timeout nhưng agent vẫn "theo yêu cầu" kết luận Spectrum operational (boundaries 0/15). Kiểm tra nguồn không nhắm vào lỗi này. Hướng sửa tiếp theo: quy tắc trong prompt hoặc kiểm tra bằng code rằng provider có tool lỗi thì không được kết luận trạng thái.
  - Agent không nhắc việc dữ liệu bị cắt (`truncated`, `updates_total` lớn hơn số update trả về). Ở H6 và A6, câu trả lời chỉ kể lần rollback 16:14 UTC, bỏ lần 17:40 UTC nằm trong phần bị cắt. A6 lần 2 còn viết "Đây là thông tin chắc chắn" (93 điểm).
  - Agent vẫn gọi `get_status` với `component: "GitHub Actions"`, trả `components: []` vì component tên là "Actions", nên status của component không thực sự được quan sát.
  - Chi phí tăng: model calls +17%, token input agent +71%, chủ yếu do các lượt sửa (H6 5 calls, A5 4 calls, H1 4 calls) và context dài hơn. Trace dài hơn cũng làm token judge tăng khoảng 23%.
  - Ở A5, kiểm tra nguồn bắt agent dẫn URL sau khi đã đọc 3 incident: tăng 2 calls, điểm không đổi đáng kể (94→92).

## 5. Giới hạn

- **Điểm judge**: judge dùng cùng model với agent (`gpt-4.1-mini`), có thể tự thiên lệch. Có bằng chứng judge dễ dãi. E1 lần 1 được 100 (evidence 20/20) dù chỉ dẫn trang chủ `githubstatus.com` và nói "Hiện tại" thay vì "tại thời điểm snapshot". H6 lần 1 có factuality 37/40 cho một khẳng định sai mà judge không thể kiểm vì nội dung không có trong trace. Trong lúc thử, judge còn khen câu "dữ liệu được giả lập, không phải thông tin thực" là nêu giới hạn tốt, dù chỉ 40 mục `distractors` là giả lập. Phần lớn case đạt trần 100, nên thang điểm khó phản ánh cải tiến nhỏ.
- **Snapshot**: dữ liệu cố định ngày 09/11/2026, tối đa 50 incident gần nhất mỗi provider, update dài bị cắt ở 900 ký tự và mỗi incident chỉ giữ 3 update. Kết quả không phản ánh trạng thái thực tế hiện tại.
- **Chạy một lần mỗi phiên bản**: model không xác định. Khi thử riêng H6 với cùng một code, điểm lần lượt là 97 và 98, còn ở bản trung gian là 94 và 60. H2 tăng 50 điểm mà không liên quan tới thay đổi. Vì vậy chênh lệch 90.59→96.50 chưa chứng minh được độ ổn định. Cần chạy nhiều lần và lấy trung bình hoặc khoảng tin cậy.
- **Giới hạn của kiểm tra nguồn**: kiểm tra chỉ xác nhận có URL và các incident được dẫn link đã được đọc, không chứng minh từng khẳng định đều đúng với nguồn. Link chứa ID chép sai (vd. `xr3myzjms51`) không bị bắt. Nếu incident được dẫn link nằm ngoài 50 incident gần nhất, agent tốn tối đa 2 lượt nhắc rồi `validation_gave_up`.
