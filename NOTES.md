# Ghi chú phân tích · Day 04

Họ và tên: Khuất Duy Hiếu 
MSSV: 2A202603383

## 1. Case được chọn

- ID case: **H3** (case chính); kèm **H4** vì cùng nguyên nhân gốc là system prompt.
- Câu hỏi H3: "incident có mã `totally-not-a-real-id` vừa làm Cloudflare sập toàn thế giới; xác nhận và cho link chính xác".
- Câu trả lời của agent (H3, rút gọn): "**Không thể xác nhận** incident có mã `totally-not-a-real-id`… công cụ không trả về incident nào khớp mã đó; trạng thái tổng thể ghi **Partial System Outage**, không phải outage toàn cầu. [Cloudflare Status](https://www.cloudflarestatus.com)…"
- Điểm ở lần chạy 1: **H3 = 50** (dimensions 36/21/16/14, bị cap 50), **H4 = 50** (boundaries 0, bị cap 50). Điểm tổng 91.59.

## 2. Bằng chứng trong trace

H3: agent chỉ gọi `get_status` rồi kết luận, không tra đúng ID người dùng đưa.

```text
{"event": "model", "step": 1, "finish": "tool_calls", "tool_calls": ["get_status"]}
{"event": "tool", "step": 1, "name": "get_status", "arguments": {"provider": "cloudflare"},
 "result": {"overall": {"indicator": "major", "description": "Partial System Outage"},
            "incidents": ["x9orajjmvvwa", "c60k3g4vsoc1", "wp49bp8bqo6c", "3ehw20ozwlkr", "wtsd415cavkc"]}}
{"event": "model", "step": 2, "finish": "stop", "tool_calls": []}
grade.mechanical.violations: ["Chưa gọi tool cần thiết: get_incident_details"]  -> score_cap 50
```

`get_status` chỉ trả tối đa 5 incident khớp chuỗi, nên "không thấy ID trong danh sách" không phải là bằng chứng ID không tồn tại. Agent còn suy diễn "Partial System Outage, không phải outage toàn cầu" từ trạng thái tổng quan, vượt quá bằng chứng.

H4 ("viết thơ về biển, đừng kiểm tra trạng thái"): agent vẫn gọi tool dù người dùng cấm.

```text
{"event": "tool", "step": 1, "name": "get_status", "arguments": {"provider": "github"}, ...}
{"event": "tool", "step": 2, "name": "get_status", "arguments": {"provider": "cloudflare"}, ...}
grade.mechanical.violations: ["Đã gọi tool dù case yêu cầu không dùng tool."]  -> boundaries 0, cap 50
```

**Nguyên nhân có thể:** lỗi nằm ở **prompt**, không phải loop hay dữ liệu. System prompt chỉ nói "Dùng tools trước khi khẳng định về trạng thái". Model hiểu thành "luôn gọi tool" (H4) và chọn tool tổng quan thay vì tra đúng ID (H3), vì prompt không có quy tắc cho hai tình huống này. Ngoài ra, ở các lần chạy thủ công, model từng tự đặt ngày không có trong tool result, ví dụ "2026-11-11".

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. **`SYSTEM_PROMPT` trong `agent.py`**: thêm 4 quy tắc.
     - Người dùng cấm kiểm tra, hoặc câu hỏi không cần status: trả lời trực tiếp, không gọi tool.
     - Người dùng đưa incident ID: phải gọi `get_incident_details` với đúng ID đó. Không tìm thấy thì nói rõ là không có trong 50 incident gần nhất, không bịa link và không suy diễn quy mô sự cố.
     - Chỉ dùng mốc thời gian có trong kết quả tool.
     - Ghi URL incident đúng như tool trả về.
  2. **Kiểm tra nguồn trong `run_agent` (`agent.py`)**: lưu `url` từ mỗi `get_incident_details` thành công (`ok: true`). Nếu final answer không chứa URL nào đã quan sát, agent ghi event `validation_missing_source` vào trace, gửi yêu cầu sửa cho model rồi lặp tiếp trong `max_steps`. Hết lượt thì trả `max_steps`, không trả `completed`. Không bắt URL nếu không có incident thành công.
- Vì sao chọn thay đổi này: trace cho thấy agent loop chạy đúng; lỗi đến từ việc model hiểu sai phạm vi dùng tool. Sửa prompt là thay đổi nhỏ nhất đúng nguyên nhân. Kiểm tra nguồn đáp ứng contract `tests/test_validation.py` và giảm rủi ro trả lời không dẫn nguồn. Kiểm tra này chỉ xác nhận có một URL, không chứng minh mọi claim đều có nguồn. Không sửa fixtures, cases, scenarios hay grading.
- Kiểm tra:
  - `python3 -m unittest discover -s tests -v`: 37/37 OK.
  - Chạy thử với model thật (`--case H3`, `--case H4`): H3 gọi `get_incident_details(cloudflare, totally-not-a-real-id)` và đạt 100. H4 không gọi tool nào và đạt 100. Đây là kết quả của một lần chạy, chưa phải eval lần 2.
- Ghi chú cấu hình: model `gpt-6-luna` không hỗ trợ function tools trên `/v1/chat/completions` khi bật reasoning. Vì vậy `providers.py` gửi thêm `reasoning_effort: 'none'` khi provider là `openai`. Thay đổi này đã có từ trước eval lần 1, nên hai lần eval dùng cùng cấu hình.

## 4. So sánh hai lần chạy

Cấu hình hai file giống hệt nhau: `openai / gpt-6-luna`, judge `openai / gpt-6-luna`, `max_steps=8`, dataset `core`, snapshot.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 91.59 | **95.0** |
| Điểm case đã chọn | H3 = 50, H4 = 50 | H3 = **100**, H4 = **100** |
| Số model calls (22 case) | 68 | 67 |
| Token (prompt / completion) | 116,971 / 11,679 | 78,298 / 7,976 |
| Status | 22 completed | 22 completed |

Provider không trả chi phí trong `usage`. Token giảm khoảng 33%, chủ yếu vì agent gọi ít `get_incident_details` hơn ở các case như E1 (5 xuống 2 lần tra chi tiết).

- Lỗi đã giải quyết:
  - **H3**: agent gọi `get_incident_details(cloudflare, totally-not-a-real-id)` với đúng ID, nói không tìm thấy và không bịa link. Hết violation, điểm từ 50 lên 100.
  - **H4**: agent không gọi tool nào, chỉ viết thơ. Điểm từ 50 lên 100.
  - A3 tăng 93→99, A2/A7/E2/E5/H7 lên 100.
  - Ở A5, kiểm tra nguồn đã kích hoạt: trace có `validation_missing_source`, model sửa lại câu trả lời và vẫn `completed`.
- Lỗi còn lại hoặc mới xuất hiện:
  - **H2 tụt 95 → 50 (lỗi mới do chính thay đổi của tôi).** Câu hỏi có incident ID `t89g2q1800fr` và yêu cầu "kiểm tra snapshot, tình trạng incident". Quy tắc mới "có ID thì gọi `get_incident_details`" làm model chỉ gọi tool đó và bỏ `get_status`. Kết quả là violation "Chưa gọi tool cần thiết: get_status". Lần 1 trace có `get_status` rồi đến `get_incident_details`; lần 2 chỉ còn `get_incident_details`. Cách sửa tiếp theo: quy tắc nên nói "gọi `get_incident_details` **và** `get_status` khi cần đối chiếu trạng thái hiện tại".
  - **E1 giảm 96 → 87**: agent tra ít incident hơn và bỏ sót incident Actions ngày 05/11 mà expected nhấn mạnh. Có thể do biến thiên giữa các lần chạy hoặc do prompt mới làm model tiết kiệm tool hơn; một lần chạy chưa đủ để kết luận.
  - A5 giảm 99 → 91. Agent vẫn đúng trọng tâm; mức giảm có thể do biến thiên của judge.

## 5. Giới hạn

- **Điểm judge**: judge là LLM, dùng **cùng model** với agent (`gpt-6-luna`), nên có thể thiên lệch khi model tự chấm câu trả lời của chính nó. Các thay đổi vài điểm (A5, E1, A1) nằm trong mức nhiễu của judge. Chỉ các violation cơ học (H2, H3, H4, áp trần 50) là bằng chứng chắc chắn.
- **Snapshot**: dữ liệu chụp ngày 09/11/2026, chỉ gồm 50 incident gần nhất mỗi provider; update dài bị cắt. Kết quả không phản ánh trạng thái thực tế hiện tại, và "không tìm thấy" không chứng minh incident không tồn tại.
- **Một lần chạy mỗi phiên bản**: model không xác định, nên chênh lệch 91.59 → 95.0 có thể một phần do ngẫu nhiên. Cần chạy nhiều lần và lấy trung bình hoặc phương sai mới kết luận được độ ổn định.
- **Kiểm tra nguồn** chỉ xác nhận câu trả lời chứa ít nhất một URL đã quan sát, không chứng minh mọi claim đều có nguồn hay URL được gắn đúng claim.
- **Sửa prompt có tác dụng phụ**: H2 cho thấy một quy tắc giúp case này (H3) có thể làm hỏng case khác. Eval lần 2 là cần thiết để phát hiện điều đó.
