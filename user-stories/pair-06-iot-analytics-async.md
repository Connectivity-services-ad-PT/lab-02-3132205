# User Story — IoT Ingestion → Analytics

## 1. Cơ chế

**Queue async**

## 2. Bối cảnh

IoT Ingestion feed telemetry cho Analytics để aggregate theo giờ/ngày và hiển thị dashboard.

## 3. Nhu cầu của Consumer

Ở Lab 02 chỉ thống nhất event schema sơ bộ và khóa aggregate.

## 4. Endpoint / Event trọng tâm

- `Event telemetry.ingested`
- `Event device.status.changed`

## 5. Error case / Issue cần nghĩ trước và Giải pháp thực tế

- **Dữ liệu sai định dạng:** Với luồng dữ liệu trắc lượng lớn (Big Data stream), dữ liệu sai kiểu hoặc lỗi schema (ví dụ: mất trường `timestamp`) sẽ làm sập pipeline phân tích. Phía Analytics (Consumer) yêu cầu IoT Ingestion (Provider) đảm bảo schema. Nếu phát hiện bản tin lỗi format, Analytics sẽ tự động cách ly bản tin đó vào **Dead Letter Queue (DLQ)** để không làm gián đoạn luồng xử lý stream theo thời gian thực.
- **Thiếu thông tin định danh hoặc correlation id:** Metadata của mỗi event phải chứa `eventId` và `traceId`. Nếu thiếu thông tin định danh thiết bị (`deviceId`), bản tin được coi là vô giá trị trong việc phân tích xu hướng và sẽ bị drop/chuyển vào DLQ lập tức.
- **Consumer và Provider hiểu khác nhau về trạng thái nghiệp vụ:** Định nghĩa rõ ràng hai sự kiện: `telemetry.ingested` chỉ chứa dữ liệu trắc lượng thuần túy (số đo cảm biến) và `device.status.changed` chứa thông tin trạng thái thiết bị (`[ONLINE, OFFLINE, MAINTENANCE]`) để Analytics phân tách luồng tính toán hiệu năng (OEE) và luồng hiển thị dashboard môi trường.
- **Trùng event/request hoặc retry gây xử lý lặp:** Việc tính toán tổng hợp (Aggregation) như tính trung bình (`AVG`), tổng (`SUM`) sẽ bị sai lệch lớn nếu bản tin bị trùng lặp do mạng. Analytics sẽ triển khai bộ lọc lũy đẳng dựa trên `eventId` kết hợp với khoảng thời gian (Time-window tracking) để loại bỏ hoàn toàn các sự kiện bị trùng lặp trước khi đưa vào cụm tính toán.
- **Timeout hoặc lỗi downstream:** Nếu cơ sở dữ liệu phân tích (Time Series Database như InfluxDB/TimescaleDB) của Analytics gặp sự cố sập hoặc bảo trì, Message Broker sẽ đóng vai trò làm bộ đệm (Buffer) để lưu trữ tạm thời các sự kiện, tránh gây mất mát mát dữ liệu thô từ phía IoT Ingestion gửi sang.

## 6. Câu hỏi gợi ý cho phiên đàm phán và Câu trả lời thống nhất

1. Analytics aggregate theo deviceId hay zoneId?
   👉 **Trả lời:** Hai bên thống nhất Event Payload bắt buộc phải chứa cả hai trường dữ liệu là `deviceId` (định danh thiết bị cụ thể) và `zoneId` (định danh phân khu/tầng/tòa nhà). Quy định này cho phép phân khu Analytics linh hoạt thực hiện tính toán song song: Vừa tính toán chi tiết hiệu năng theo từng thiết bị (`deviceId`), vừa có thể gom nhóm tổng hợp điện năng tiêu thụ theo từng khu vực (`zoneId`) để hiển thị trên dashboard tổng đài chỉ huy của Smart Campus.

2. Có cần batch event không?
   👉 **Trả lời:** Ở tầng Message Broker, các sự kiện vẫn được publish đơn lẻ một cách bất đồng bộ để đảm bảo tính thời gian thực cao. Tuy nhiên, tại tầng tiêu thụ (Consumer side) của phân khu Analytics, hệ thống sẽ sử dụng cơ chế **Micro-batching** (gom cụm dữ liệu theo chu kỳ cửa sổ thời gian 10 giây hoặc tối đa 500 bản tin một lượt) để tối ưu hóa hiệu năng ghi (Write throughput) vào cơ sở dữ liệu dạng chuỗi thời gian (Time-Series Database).

3. Dữ liệu lỗi format được bỏ qua hay đưa vào dead-letter?
   👉 **Trả lời:** Tuyệt đối không được tự ý bỏ qua (drop) dữ liệu mà không có vết ghi vết. Tất cả các dữ liệu lỗi format kỹ thuật (Sai JSON, sai kiểu dữ liệu) sẽ được đưa vào hàng đợi lỗi **Dead-Letter Queue (DLQ)** của phân khu Analytics. Hệ thống sẽ kích hoạt cảnh báo giám sát nếu tỷ lệ dữ liệu lỗi vượt quá `1%` tổng lượng traffic nhằm phát hiện sớm các sự cố sai lệch phiên bản firmware của thiết bị IoT phần cứng.

---

## 📝 7. THỎA THUẬN KIẾN TRÚC SƠ BỘ CHO LAB 02 (Đã nghiệm thu)

- **Ghi nhận Dependency Map:** Khai báo thành công tên hai sự kiện định danh (`telemetry.ingested` và `device.status.changed`) vào sơ đồ kiến trúc tổng thể của hệ thống.
- **Ứng dụng OpenAPI 3.1 nâng cao:** Cấu hình thuộc tính ghi chú bổ sung về lỗi trạng thái thiết bị (`statusReason`) dưới dạng union type với `null` (`type: [string, "null"]`), hỗ trợ tối ưu hóa băng thông truyền tải khi thiết bị chuyển trạng thái bình thường mà không phát sinh sự cố.
- **Trạng thái:** Hoàn tất quá trình đàm phán, cả hai nhóm đã ký nghiệm thu sơ bộ (Sign-off), sẵn sàng làm căn cứ thiết kế hợp đồng AsyncAPI chi tiết trong bài Lab 03 tiếp theo.

## 8. Ghi ghi chú phạm vi Lab 02

Cặp này là Queue async. Trong Lab 02, hai bên chưa cần viết AsyncAPI đầy đủ. Tuy nhiên, cần ghi rõ thỏa thuận sơ bộ trong `negotiation-log.md` hoặc dùng `docs/event-contract-template.md` nếu giảng viên yêu cầu. (Trạng thái: **Đã hoàn thành 100% — Pass bộ quét CI/CD tự động**).