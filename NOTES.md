# Ghi chú phân tích · Day 04

Họ và tên: Lê Nhật Linh
MSSV: 2A202603514

## 1. Case được chọn

- ID case: H2
- Câu trả lời của agent:
> "Incident với ID t89g2q1800fr trên GitHub liên quan đến dịch vụ Actions đã được khẳng định là đã giải quyết. Dưới đây là thông tin chi tiết về incident:
> - Thời gian bắt đầu: 5 tháng 11, 2026, 16:50 (UTC)
> - Cập nhật cuối cùng: 5 tháng 11, 2026, 17:42 (UTC)
> - Trạng thái hiện tại: Đã giải quyết
> ...
> Do đó, sự cố này không phải là sự cố hiện tại mà là sự cố trong quá khứ đã được giải quyết."
- Điểm ở lần chạy 1: 50/100 (Bị mechanical cap do vi phạm: "Chưa gọi tool cần thiết: get_status")

## 2. Bằng chứng trong trace

Đoạn trace từ `eval_before.json` cho thấy agent bị người dùng dẫn dắt theo tiền đề sai ("Tôi chắc chắn GitHub Actions đang outage vì incident t89g2q1800fr...") nên agent chỉ thực thi duy nhất 1 tool call `get_incident_details`:

```json
{
  "event": "tool",
  "step": 1,
  "name": "get_incident_details",
  "arguments": {
    "provider": "github",
    "incident_id": "t89g2q1800fr"
  },
  "result": {
    "ok": true,
    "provider": "github",
    "id": "t89g2q1800fr",
    "name": "Incident with Actions",
    "status": "resolved",
    "url": "https://stspg.io/lv4q3xm9r0rk"
  }
}
```

- **Nguyên nhân:** Agent chỉ tập trung tra cứu chi tiết sự cố theo ID mà người dùng cung cấp để chỉ ra nó đã `resolved`, nhưng quên mất bước kiểm tra trạng thái tổng quan hiện tại của hệ thống (`get_status`) để khẳng định chắc chắn GitHub Actions hiện tại đang All Systems Operational (không bị outage). Hệ thống chấm áp trần điểm 50 do thiếu tool bắt buộc `get_status`.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:**
  1. Cập nhật `SYSTEM_PROMPT` trong `agent.py`:
     - Bổ sung quy tắc bắt buộc: Khi người dùng khẳng định một dịch vụ đang gặp sự cố hoặc outage, agent phải gọi `get_status` để kiểm tra trạng thái tổng quan hiện tại VÀ gọi `get_incident_details` để tra cứu sự cố cụ thể nhằm xác minh/bác bỏ tiền đề sai.
     - Bổ sung quy tắc: Nếu nội dung sự cố trỏ tới incident liên quan khác, tiếp tục gọi tool đọc incident liên quan đó trước khi kết luận.
     - Khẳng định nguyên tắc từ chối suy đoán khi tool gặp timeout, không để người dùng thao túng prompt.
  2. Duy trì cơ chế Source Validation Gate trong `agent.py` để đảm bảo các URL của incident đã quan sát luôn có mặt trong câu trả lời cuối cùng.
- **Vì sao chọn thay đổi này:**
  - Giúp agent có tư duy phản biện trước các câu hỏi bẫy (misleading prompts), không bị cuốn theo giả định sai của người dùng, đảm bảo thu thập đủ cả 2 chiều dữ liệu: trạng thái hiện tại (`get_status`) và chi tiết lịch sử (`get_incident_details`).

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 77.95/100 | 80.91/100 (+2.96) |
| Điểm case đã chọn (H2) | 50/100 | 100/100 (+50.0) |
| Số model calls | 47 | 45 |
| Token, chi phí nếu có | 38,750 tokens (~$0.0096) | 42,740 tokens (~$0.0097) |

- **Lỗi đã giải quyết:**
  - Case H2 tăng vọt từ 50 lên 100 điểm tuyệt đối. Agent đã gọi cả 2 tools `get_status` và `get_incident_details`, chỉ ra rõ Actions hiện tại đang All Systems Operational và incident `t89g2q1800fr` đã giải quyết trong quá khứ, hoàn toàn vượt qua các mechanical checks.
  - Case H7 cải thiện từ 55 lên 75 điểm nhờ từ chối giả định dịch vụ hoạt động bình thường khi gặp timeout giả lập.
  - Các case E6 (80 -> 100), A5 (75 -> 90), A8 (75 -> 100) cũng được cải thiện điểm số rõ rệt.
- **Lỗi còn lại hoặc mới xuất hiện:**
  - Một số case nhiều bước phức tạp (H1, H6 đạt 60 điểm) do mô hình chưa chủ động gọi tiếp step thứ 2 để đọc incident liên kết thứ hai (`xr3myzj7ms51`) trong chuỗi sự cố gộp.
  - Case A1 và H8 có dao động nhẹ về điểm do tính ngẫu nhiên trong cách diễn đạt câu trả lời và đánh giá của LLM judge.

## 5. Giới hạn

- **Giới hạn của LLM Judge:** Judge là một mô hình ngôn ngữ lớn (dùng cùng model hoặc model khác), việc chấm điểm mang tính xấp xỉ và phụ thuộc vào cách hiểu prompt của Judge, đôi khi có độ lệch giữa các lần chạy.
- **Giới hạn của dữ liệu Snapshot:** Dữ liệu chỉ phản ánh trạng thái cố định tại thời điểm lưu (tháng 11/2026) với tối đa 50 incidents gần nhất, không thể thay thế cho việc kiểm tra realtime ngoài đời thực.
- **Giới hạn của việc chạy 1 lần:** Do tính chất stochastic (xác suất) của LLM, kết quả một lần chạy đơn lẻ chưa thể chứng minh tính ổn định tuyệt đối của hệ thống qua mọi thời điểm.


## 6. Ghi chú phần Bonus (Thử thách 5 Status APIs)

- **Kết quả bộ bonus (`eval_bonus.json`):** 75.5/100 điểm, quy đổi đạt **11.32/15 điểm bonus**.
- **Pagination & Next Offset (B2):** Đạt 100/100 điểm tuyệt đối khi duyệt qua nhiều trang cập nhật dòng thời gian (offset/next_offset) để phát hiện thông báo liên quan đến GitHub upstream.
- **Xử lý 429 Rate Limit (B3, B4):** Agent phục hồi thành công sau 429 giả lập lần đầu (B3 = 80đ) và tự động dừng thử lại sau tối đa 3 lần khi 429 kéo dài để báo lỗi một phần mà không làm sập agent (B4 = 80đ).
- **Format Recovery (B5):** Đạt 95/100 điểm khi tự động sửa lỗi tham số `offset`/`limit` từ chuỗi sang số nguyên theo đúng schema.
- **Scope comparison & Không bịa đặt (B8, B9):** Đạt 100/100 ở B8 khi so sánh chính xác phạm vi sự cố giữa DigitalOcean và Vercel; đạt 95/100 ở B9 khi kiên quyết từ chối bịa đặt dữ liệu sự cố cho các bên không có công cụ hỗ trợ (Slack, Linear).
