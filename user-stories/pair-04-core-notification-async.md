# User Story — Core Business → Notification

## 1. Cơ chế

**Queue async**

## 2. Bối cảnh

Core Business phát event alert để Notification gửi thông báo đa kênh như Telegram, email hoặc app message.

## 3. Nhu cầu của Consumer

Ở Lab 02 chỉ thống nhất event name, payload tối thiểu và retry assumption; chi tiết AsyncAPI để Lab 03.

## 4. Endpoint / Event trọng tâm

- `Event alert.created`
- `Event alert.escalated`
- `Event alert.resolved`

## 5. Error case / Issue cần nghĩ trước và Giải pháp thực tế

- **Dữ liệu sai định dạng:** Dù truyền qua Queue bất đồng bộ, Event Payload vẫn phải tuân thủ cấu trúc JSON Schema được thỏa thuận trước. Nếu Notification phát hiện bản tin sai định dạng cấu trúc, nó sẽ đẩy bản tin đó vào một hàng đợi lỗi riêng biệt gọi là **DLQ (Dead Letter Queue)** để tránh làm treo luồng xử lý chính.
- **Thiếu thông tin định danh hoặc correlation id:** Bản tin sự kiện bắt buộc phải có thuộc tính `eventId` và `correlationId` ở phần Metadata. Nếu thiếu, Notification sẽ không thể trace luồng gửi thông báo chéo kênh (Audit Trail), bản tin sẽ bị coi là không hợp lệ và chuyển vào DLQ.
- **Consumer và Provider hiểu khác nhau về trạng thái nghiệp vụ:** Hai bên thống nhất đồng bộ hóa danh mục trạng thái qua 3 Event cụ thể: Khởi tạo cảnh báo (`created`), Leo thang mức độ nguy hiểm (`escalated`), và Đã xử lý/Đóng cảnh báo (`resolved`).
- **Trùng event/request hoặc retry gây xử lý lặp:** Hệ thống Message Broker có thể phân phối lặp bản tin (At-least-once delivery). Phân khu Notification (Provider) sẽ hiện thực tính lũy đẳng (Idempotency) bằng cách lưu lại các `eventId` đã xử lý thành công vào Redis Cache trong vòng 24 giờ. Nếu gặp lại `eventId` cũ, hệ thống sẽ bỏ qua không gửi thông báo trùng lặp cho người dùng.
- **Timeout hoặc lỗi downstream:** Nếu một kênh trung gian (như Telegram API hoặc dịch vụ GMail) bị sập/timeout, Notification Service sẽ tạm lưu bản tin vào hàng đợi retry nội bộ kèm theo cơ chế cấu hình giãn cách thời gian (Exponential Backoff).

## 6. Câu hỏi gợi ý cho phiên đàm phán và Câu trả lời thống nhất

1. Event alert.created cần field severity không?
    **Trả lời:** Rất cần. Hai bên thống nhất bổ sung trường `severity` vào Payload của sự kiện `alert.created` với các giá trị Enum nghiêm ngặt: `[INFO, WARNING, CRITICAL]`. Thuộc tính này giúp hệ thống Notification nhận biết mức độ khẩn cấp để ưu tiên đẩy lên các kênh có tốc độ phản hồi tức thời (như SMS/Telegram thay vì Email).

2. Notification có cần nhận danh sách channel hay tự định tuyến?