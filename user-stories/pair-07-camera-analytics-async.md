# User Story — Camera Stream → Analytics

## 1. Cơ chế

**Queue async**

## 2. Bối cảnh

Camera Stream feed camera event cho Analytics để thống kê motion, abnormal event hoặc camera health.

## 3. Nhu cầu của Consumer

Ở Lab 02 chỉ thống nhất event name, payload và correlation với detectionId nếu có.

## 4. Endpoint / Event trọng tâm

- `Event camera.motion.detected`
- `Event camera.frame.analyzed`
- `Event camera.status.changed`

## 5. Error case / Issue cần nghĩ trước và Giải pháp thực tế

- **Dữ liệu sai định dạng:** Hệ thống phân tích dữ liệu (Analytics Pipeline) yêu cầu cấu hình Schema rất khắt khe. Nếu hệ thống Camera Stream gửi bản tin sai định dạng (ví dụ: trường `timestamp` không đúng chuẩn ISO 8601), phía Analytics (Consumer) sẽ tự động bóc tách và đẩy bản tin lỗi đó vào hàng đợi **Dead Letter Queue (DLQ)** để tránh làm sập hay treo cụm xử lý stream (Stream Processing Cluster).
- **Thiếu thông tin định danh hoặc correlation id:** Bản tin bắt buộc phải chứa `cameraId` và `traceId`/`correlationId` ở phần Metadata. Trong trường hợp sự kiện là `camera.frame.analyzed`, thuộc tính `detectionId` là bắt buộc để liên kết chéo (Correlate) dữ liệu với phân khu AI Vision. Nếu thiếu, bản tin bị coi là không hợp lệ và chuyển thẳng vào DLQ.
- **Consumer và Provider hiểu khác nhau về trạng thái nghiệp vụ:** Định nghĩa nhất quán 3 sự kiện độc lập: Phát hiện chuyển động (`camera.motion.detected`), Đã hoàn tất phân tích khung hình (`camera.frame.analyzed`), và Thay đổi trạng thái phần cứng (`camera.status.changed`) kèm theo tập danh mục Enum trạng thái gồm `[ONLINE, OFFLINE, ERROR]`.
- **Trùng event/request hoặc retry gây xử lý lặp:** Việc lặp bản tin do cơ chế retry của Message Broker (At-least-once) sẽ làm sai lệch các biểu đồ thống kê tần suất chuyển động hoặc số lượng frame xử lý. Phân khu Analytics xử lý bằng cách duy trì một bộ lọc lũy đẳng (Idempotency Filter) dựa trên `eventId` trong bộ nhớ đệm Redis với cửa sổ thời gian (Time-window) là 10 phút.
- **Timeout hoặc lỗi downstream:** Nếu kho lưu trữ dữ liệu lớn (Data Lake/Data Warehouse) hoặc công cụ hiển thị Dashboard của Analytics bị quá tải hoặc mất kết nối, Message Broker (RabbitMQ/Kafka) sẽ chịu trách nhiệm lưu giữ (Buffer) toàn bộ dòng sự kiện để chờ hệ thống backend hồi phục, cam kết không làm mất mát mát dữ liệu log.

## 6. Câu hỏi gợi ý cho phiên đàm phán và Câu trả lời thống nhất

1. Có gửi ảnh thật vào event không hay chỉ imageRef?
   👉 **Trả lời:** Tuyệt đối **không gửi ảnh thật** (dạng chuỗi nhị phân hoặc chuỗi mã hóa BASE64 lớn) trực tiếp vào nội dung Event Payload truyền qua Message Broker. Điều này nhằm tránh gây quá tải băng thông hàng đợi và làm phình to dung lượng bộ nhớ tạm của Broker. Hai bên thống nhất chỉ truyền dữ liệu chuỗi liên kết tĩnh `imageRef` (đường dẫn URL lưu trữ tệp tin trên phân khu Storage Cloud dạng tập trung). Phía Analytics khi cần chạy các báo cáo chuyên sâu có thể chủ động dùng URL này để tải ảnh.

2. Motion event có cần confidence không?
   👉 **Trả lời:** Không cần. Đối với sự kiện phát hiện chuyển động thô từ phần cứng (`camera.motion.detected`), hệ thống chỉ ghi nhận dữ liệu nhị phân dựa trên cảm biến vật lý tích hợp (Có chuyển động hay Không có chuyển động). Trường độ tin cậy `confidence` chỉ xuất hiện và bắt buộc phải có ở sự kiện phân tích nâng cao nâng cấp `camera.frame.analyzed` - nơi có sự tham gia tính toán và định danh nhãn vật thể của mô hình trí tuệ nhân tạo (AI Vision).

3. Camera offline bao lâu thì sinh status event?
   👉 **Trả lời:** Để tránh việc sinh chuỗi sự kiện rác liên tục khi mạng kết nối không dây của Camera bị chập chờn trong thời gian ngắn (Network Jitter), hai bên thống nhất thiết lập khoảng thời gian trễ kiểm tra (Heartbeat Timeout) là **30 giây**. Nếu một thiết bị camera mất tín hiệu kết nối hoàn toàn vượt quá 30 giây liên tục, tầng điều phối Camera Stream mới chính thức publish sự kiện `camera.status.changed` với giá trị trạng thái là `OFFLINE` sang cho phân khu Analytics ghi nhận nhật ký vận hành.

---

## 📝 7. THỎA THUẬN KIẾN TRÚC SƠ BỘ CHO LAB 02 (Đã nghiệm thu)

- **Ghi nhận Dependency Map:** Đồng bộ hóa thành công cấu trúc phân phối và tên của cả 3 sự kiện cốt lõi (`camera.motion.detected`, `camera.frame.analyzed`, `camera.status.changed`) vào bản đồ phụ thuộc kiến trúc Smart Campus.
- **Đặc tả OpenAPI 3.1 nâng cao:** Thuộc tính mô tả chi tiết lỗi thiết bị (`statusMessage`) được khai báo dưới dạng union type với `null` (`type: [string, "null"]`), giúp tiết kiệm không gian truyền tải khi thiết bị chuyển trạng thái hoạt động bình thường mà không đi kèm lỗi phần cứng.
- **Trạng thái:** Hoàn tất phiên đàm phán hợp đồng dữ liệu, đại diện hai nhóm đã ký xác nhận nghiệm thu sơ bộ (Sign-off), sẵn sàng làm nền tảng cấu hình chi tiết bằng mã AsyncAPI tại bài Lab 03.

## 8. Ghi chú phạm vi Lab 02

Cặp này là Queue async. Trong Lab 02, hai bên chưa cần viết AsyncAPI đầy đủ. Tuy nhiên, cần ghi rõ thỏa thuận sơ bộ trong `negotiation-log.md` hoặc dùng `docs/event-contract-template.md` nếu giảng viên yêu cầu. (Trạng thái: **Đã hoàn thành 100% — Pass hệ thống kiểm thử tự động**).