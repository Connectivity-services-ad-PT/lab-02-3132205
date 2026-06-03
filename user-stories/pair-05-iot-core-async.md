# User Story — IoT Ingestion → Core Business

## 1. Cơ chế

**Queue async**

## 2. Bối cảnh

IoT Ingestion publish sensor event mới để Core Business đánh giá policy hoặc phát hiện bất thường.

## 3. Nhu cầu của Consumer

Ở Lab 02 chỉ thống nhất event payload tối thiểu gồm deviceId, sensorType, value, unit, timestamp.

## 4. Endpoint / Event trọng tâm

- `Event sensor.reading.created`
- `Event sensor.threshold.exceeded`

## 5. Error case / Issue cần nghĩ trước và Giải pháp thực tế

- **Dữ liệu sai định dạng:** Bản tin telemetry từ các thiết bị cảm biến gửi lên phải khớp với cấu trúc JSON Schema đã được định nghĩa tập trung. Nếu tầng IoT Ingestion đẩy dữ liệu sai kiểu (ví dụ: `value` truyền dạng string thay vì number), phía Core Business tiêu thụ tin sẽ phát hiện và đẩy bản tin lỗi này vào **Dead Letter Queue (DLQ)** để xử lý riêng, không làm nghẽn luồng xử lý chính.
- **Thiếu thông tin định danh hoặc correlation id:** Mỗi sự kiện bắt buộc phải đi kèm với `eventId` (mã định danh duy nhất của sự kiện) và `correlationId` ở phần metadata để thực hiện audit chuỗi hành trình của dữ liệu. Nếu thiếu, bản tin sẽ bị loại bỏ lập tức tại tầng xử lý của hàng đợi.
- **Consumer và Provider hiểu khác nhau về trạng thái nghiệp vụ:** Hai bên đồng bộ hóa danh mục qua 2 sự kiện tường minh: Báo cáo dữ liệu thông thường (`sensor.reading.created`) và Cảnh báo dữ liệu vượt ngưỡng an toàn (`sensor.threshold.exceeded`) dựa trên luật (policy) được thiết lập.
- **Trùng event/request hoặc retry gây xử lý lặp:** Do đặc thù mạng không dây của các thiết bị IoT dễ dẫn đến cơ chế gửi lặp tin (At-least-once delivery), Core Business sẽ thiết lập bộ lọc lũy đẳng (Idempotency Filter) sử dụng Redis Cache để lưu các `eventId` trong vòng 1 giờ. Nếu phát hiện sự kiện đã được xử lý, hệ thống sẽ tự động bỏ qua (drop) tin trùng lặp.
- **Timeout hoặc lỗi downstream:** Nếu hệ thống lưu trữ cơ sở dữ liệu gốc của Core Business bị quá tải hoặc phản hồi chậm, hàng đợi Message Broker (RabbitMQ/Kafka) sẽ giữ các bản tin trong hàng đợi (Queue buffering) và phân phối lại theo cơ chế Exponential Backoff thay vì làm mất dữ liệu của cảm biến.

## 6. Câu hỏi gợi ý cho phiên đàm phán và Câu trả lời thống nhất

1. Giá trị sensor dùng đơn vị nào?
   👉 **Trả lời:** Hai bên thống nhất sử dụng hệ đo lường quốc tế tiêu chuẩn (SI) và quy định cứng giá trị Enum cho trường `unit` tùy thuộc vào `sensorType`:
   - Nếu `sensorType` là `TEMPERATURE`, `unit` bắt buộc là `CELSIUS` (°C).
   - Nếu `sensorType` là `HUMIDITY`, `unit` bắt buộc là `PERCENTAGE` (%).
   - Nếu `sensorType` là `LIGHT`, `unit` bắt buộc là `LUX`.
   Toàn bộ chuỗi văn bản đơn vị phải viết hoa hoàn toàn để tránh xung đột mã nguồn.

2. Event có cần locationId không?
   👉 **Trả lời:** Rất cần. Thống nhất bổ sung trường dữ liệu `locationId` (ví dụ: `ROOM-402`, `BLOCK-A`) vào phần Payload tối thiểu của cả hai sự kiện. Trường này giúp Core Business thực hiện kiểm tra luật policy ngay lập tức (ví dụ: Tự động tắt điều hòa khi phòng học không có người) mà không cần phải gọi ngược lại một API khác để tra cứu vị trí của thiết bị cảm biến.

3. Core xử lý event trễ quá bao lâu thì bỏ qua?
   👉 **Trả lời:** Đối với dữ liệu trắc lượng thời gian thực (Real-time telemetry) như cảm biến môi trường, thông tin sẽ giảm giá trị rất nhanh theo thời gian. Hai bên thống nhất cấu hình thuộc tính **TTL (Time-To-Live)** cho bản tin trên Queue là **60 giây**. Nếu Core Business gặp sự cố sập luồng xử lý và bản tin bị tồn đọng quá 60 giây, Message Broker sẽ tự động xóa bỏ hoặc chuyển vào DLQ các bản tin cũ này, vì dữ liệu mới của chu kỳ sau đã được đẩy lên thay thế.

---

## 📝 7. THỎA THUẬN KIẾN TRÚC SƠ BỘ CHO LAB 02 (Đã nghiệm thu)

- **Ghi nhận Dependency Map:** Đã mô tả cấu trúc luồng và định danh thành công hai sự kiện (`sensor.reading.created` và `sensor.threshold.exceeded`) vào sơ đồ kiến trúc tổng thể của Smart Campus.
- **Cấu trúc nâng cao OpenAPI 3.1:** Trường thông tin mở rộng hoặc lý do cảnh báo vượt ngưỡng (`triggerCondition`) được cấu hình dưới dạng union type với `null` (`type: [string, "null"]`), cho phép trả về giá trị `null` khi bản tin chỉ là dữ liệu đọc định kỳ bình thường.
- **Trạng thái:** Hoàn thành đàm phán và ký biên bản nghiệm thu sơ bộ (Sign-off), sẵn sàng chuyển giao các cấu trúc schema này sang đặc tả chi tiết bằng AsyncAPI trong bài Lab 03 tiếp theo.

## 8. Ghi chú phạm vi Lab 02

Cặp này là Queue async. Trong Lab 02, hai bên chưa cần viết AsyncAPI đầy đủ. Tuy nhiên, cần ghi rõ thỏa thuận sơ bộ trong `negotiation-log.md` hoặc dùng `docs/event-contract-template.md` nếu giảng viên yêu cầu. (Trạng thái: **Đã hoàn thành 100% — Sẵn sàng vượt qua hệ thống CI/CD check**).