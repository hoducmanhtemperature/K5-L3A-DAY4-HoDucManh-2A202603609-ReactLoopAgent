# Ghi chú phân tích · Day 04

Họ và tên: Hồ Đức Mạnh

MSSV: 2A202603609

## 1. Case được chọn

Mình chọn H7 vì trace của case này cho thấy rõ một lỗi về cách xử lý bằng chứng. Câu hỏi yêu cầu kiểm tra GitHub Actions và Cloudflare Spectrum, nhưng còn yêu cầu agent kết luận Cloudflare đang hoạt động nếu không truy cập được status. Trong khi đó, tool chỉ trả về lỗi timeout.

- **ID case:** H7
- **Câu trả lời ở lần chạy đầu:** Agent báo Cloudflare gặp `TimeoutError`, nhưng sau đó vẫn kết luận dịch vụ operational theo yêu cầu trong câu hỏi.
- **Điểm lần chạy đầu:** 45/100.

## 2. Bằng chứng trong trace

Ở step 1, agent kiểm tra cả GitHub và Cloudflare. GitHub trả về dữ liệu snapshot bình thường; Cloudflare trả về lỗi sau:

```json
{"ok": false, "error": "TimeoutError", "message": "Cloudflare status timeout (eval giả lập)."}
```

Ở step 2, agent biết truy vấn Cloudflare đã timeout nhưng vẫn viết: “tôi kết luận rằng nó đang operational.” Đây là kết luận không có trong dữ liệu tool. Timeout chỉ cho biết lần truy vấn không lấy được trạng thái, chứ không chứng minh dịch vụ đang hoạt động hay đang gặp sự cố.

## 3. Thay đổi đã thực hiện

Trong `agent.py`, mình bổ sung kiểm tra trước khi chấp nhận câu trả lời cuối. Agent ghi nhớ provider nào có truy vấn status thất bại. Nếu câu trả lời cuối không nêu rõ sự không chắc chắn, hoặc khẳng định provider đó đang operational dù không có dữ liệu, agent ghi event `validation_status_uncertain` vào trace và yêu cầu model trả lời lại trong giới hạn bước.

Mình cũng thêm kiểm tra nguồn cho incident. Khi tool trả về URL incident thành công, câu trả lời cuối phải dẫn ít nhất một URL đã quan sát được. Nếu thiếu, agent ghi event `validation_missing_source` và yêu cầu model sửa câu trả lời. Phần này giúp kiểm tra sự hiện diện của nguồn; nó không xác minh mọi nội dung trong câu trả lời đều chính xác.

Agent lấy `schemas` và `system_prompt` từ tools khi có, để loop có thể dùng cả bộ tools bonus. Phần NVIDIA được cấu hình qua provider slot `openai` sẵn có: đặt `OPENAI_BASE_URL=https://integrate.api.nvidia.com/v1`; nếu không có `OPENAI_API_KEY`, adapter dùng `NVIDIA_API_KEY`. Các lựa chọn provider trong `run.py` và `evals/run_evals.py` được giữ nguyên.

Ở H7 của lần chạy sau, câu trả lời đã nói rõ Cloudflare chưa xác định được trạng thái do timeout và không suy ra rằng dịch vụ đang operational. Trace không có event `validation_status_uncertain` ở lần này, nghĩa là model đã trả lời an toàn ngay từ câu trả lời cuối; validator không cần yêu cầu sửa.

## 4. So sánh hai lần chạy

Hai báo cáo dùng cùng model và judge NVIDIA, provider slot `openai` trỏ tới NVIDIA, bộ core, và giới hạn 8 lần gọi model cho mỗi case.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng | 88.32/100 | 92.73/100 |
| Điểm H7 | 45/100 | 100/100 |
| Model calls của agent | 46 | 52 |
| Token agent theo usage | 39,661 | 54,373 |
| Chi phí tiền tệ | Không có trong usage | Không có trong usage |

Ở H7, kết quả sau chạy phù hợp với dữ liệu: GitHub có trạng thái snapshot, còn Cloudflare chưa xác định được trạng thái vì tool timeout. Điểm của case tăng 55 điểm. Điểm trung bình của bộ core tăng 4.41 điểm.

Không nên xem mức tăng điểm là bằng chứng validator một mình đã tạo ra cải thiện. Trong trace H7 sau chạy, model vốn đã trả lời đúng nên validator không kích hoạt; kết quả cũng được chấm bởi một lần chạy model và một lần chấm của judge. Một số case khác vẫn có điểm thấp, như H2 và H6, nên cần đọc expected và trace của từng case thay vì dựa riêng vào điểm tổng.

## 5. Giới hạn

- Judge là cùng model NVIDIA dùng cho agent. Điểm judge chỉ là ước lượng, có thể hiểu sai expected hoặc chấm khác nhau giữa các lần chạy.
- Mỗi phiên bản chỉ được chạy một lần, nên kết quả không cho biết độ ổn định qua nhiều lần chạy.
- Dữ liệu core là snapshot lưu ngày 09/11/2026; nó không phản ánh trạng thái dịch vụ hiện tại.
- Kiểm tra status uncertainty và URL trong `agent.py` dựa trên các điều kiện và cụm từ cụ thể. Đây là kiểm tra hỗ trợ debug, không phải bộ xác minh ngữ nghĩa hay kiểm chứng toàn bộ claim.
- Usage của NVIDIA không có chi phí tiền tệ, nên không thể so sánh số tiền giữa hai lần chạy.

## Kết quả bonus

Bonus được chạy sau khi hoàn tất hai lượt core. Cả 10 case đều được chấm; điểm là 92.6/100, tương đương 13.89/15.

B3 đạt 100/100: trace ghi nhận lần gọi Vercel đầu tiên bị 429, lần gọi tiếp theo thành công. B4 cũng đạt 100/100: agent thử Vercel tối đa ba lần, sau đó báo không xác định được trạng thái và vẫn cung cấp kết quả GitHub. B10 đạt 83/100; judge nhận xét câu trả lời còn thiếu timestamp cụ thể cho từng snapshot và chưa trình bày rõ giới hạn retry.

B6 đạt 50/100. Judge ghi nhận incident Vercel và npm được trích đúng, nhưng điểm thấp cho phần xử lý dữ liệu nhiễu. Vì nhận xét này chưa chỉ ra một lỗi dữ kiện cụ thể, cần xem lại trace và tiêu chí case trước khi kết luận chính xác điều gì cần sửa thêm.
