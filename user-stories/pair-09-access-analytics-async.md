# User Story — Access Gate → Analytics

## 1. Cơ chế

**Queue async**

## 2. Bối cảnh

Access Gate feed log ra/vào cho Analytics để thống kê lưu lượng, giờ cao điểm và tỷ lệ deny.

## 3. Nhu cầu của Consumer

Ở Lab 02 chỉ thống nhất payload log và idempotency.

## 4. Endpoint / Event trọng tâm

- `Event access.log.created`
- `Event access.denied`

## 5. Error case / Issue cần nghĩ trước và Giải pháp thực tế

- **Dữ liệu sai định dạng:** Để tránh làm gián đoạn luồng xử lý stream thời gian thực hoặc làm sai lệch biểu đồ thống kê lưu lượng Smart Campus, toàn bộ Event Payload truyền qua Message Broker phải tuân thủ nghiêm ngặt cấu trúc JSON Schema đã được thỏa thuận trước. Nếu phân khu Analytics (Consumer) phát hiện bản tin sai định dạng, bản tin đó sẽ tự động được đẩy vào **Dead Letter Queue (DLQ)** để xử lý riêng.
- **Thiếu thông tin định danh hoặc correlation id:** Bản tin sự kiện bắt buộc phải đi kèm với `eventId`/`logId` và `traceId`/`correlationId` ở phần Metadata của thông điệp để phục vụ chuỗi theo vết hành trình dữ liệu (Data Lineage). Bản tin thiếu các trường định danh thiết bị (`gateId`) hoặc thẻ (`cardId`) sẽ bị coi là vô giá trị trong việc phân tích và bị chuyển thẳng vào DLQ.
- **Consumer và Provider hiểu khác nhau về trạng thái nghiệp vụ:** Hai bên đồng bộ hóa danh mục qua 2 sự kiện tường minh: Bản tin quẹt thẻ thành công (`access.log.created`) và Bản tin bị từ chối truy cập (`access.denied`) kèm theo tập danh mục lý do lỗi từ chối thống nhất (như `EXPIRED_CARD`, `INVALID_ZONE`).
- **Trùng event/request hoặc retry gây xử lý lặp:** Cơ chế phân phối bản tin của Message Broker (At-least-once delivery) hoặc việc thiết bị phần cứng retry gửi tin khi mạng chập chờn sẽ tạo ra các bản tin log trùng lặp, gây sai lệch nghiêm trọng cho thống kê giờ cao điểm. Analytics xử lý bằng cách duy trì một bộ lọc lũy đẳng (Idempotency Filter) dựa trên `eventId`/`logId` duy nhất trong Redis Cache với cửa sổ thời gian (Time-window) là 1 giờ để tự động loại bỏ bản tin trùng.
- **Timeout hoặc lỗi downstream:** Khi hệ thống cơ sở dữ liệu phân tích luồng hoặc công cụ hiển thị Dashboard của Analytics gặp sự cố sập hoặc bảo trì, hàng đợi Message Broker (RabbitMQ/Kafka) sẽ đóng vai trò làm bộ đệm (Buffer) để lưu giữ an toàn mọi log quẹt thẻ từ Access Gate gửi sang cho đến khi hệ thống hoạt động bình thường trở lại.

## 6. Câu hỏi gợi ý cho phiên đàm phán và Câu trả lời thống nhất

1. direction dùng IN/OUT hay ENTER/EXIT?
   👉 **Trả lời:** Để đồng bộ hóa nhất quán với dữ liệu thiết kế OpenAPI của phân khu Access Gate ở các cặp REST sync trước, hai bên thống nhất sử dụng cặp giá trị **`IN` / `OUT`** dưới dạng chuỗi viết hoa (String Enum) cho thuộc tính `direction`. Giá trị `IN` đại diện cho hành động đi vào phân khu/tòa nhà, và `OUT` đại diện cho hành động đi ra.

2. Có cần hash cardId để bảo vệ dữ liệu cá nhân không?
   👉 **Trả lời:** Nhằm tuân thủ nghiêm ngặt các quy định về bảo mật thông tin cá nhân và an toàn dữ liệu trong Smart Campus, hai bên thống nhất **bắt buộc phải thực hiện băm (Hash)** mã thẻ sinh viên/nhân viên (`cardId`) bằng thuật toán **SHA-256** trước khi đóng gói payload đưa vào hàng đợi Message Broker. Phân khu Analytics chỉ tiêu thụ chuỗi băm này để phục vụ mục đích đếm số lượt quẹt, phân tích tần suất hoặc phát hiện bất thường mà không cần biết danh tính thực sự của chủ thẻ.

3. Một lần quẹt lỗi có tạo log không?
   👉 **Trả lời:** Có. Mọi hành vi tương tác vật lý tại cổng kiểm soát (quẹt thẻ, quét mã) đều phải được ghi vết đầy đủ nhằm đảm bảo tính an ninh tổng thể. Sự khác biệt được phân tách qua loại Event phát đi: Nếu quẹt thẻ thành công và cổng mở, hệ thống sẽ bắn sự kiện `access.log.created`. Nếu quẹt thẻ thất bại (thẻ hết hạn, thẻ không có quyền vào khu vực, thẻ bị khóa), hệ thống sẽ bắn sự kiện `access.denied` kèm theo mã lý do cụ thể để Analytics tính toán chính xác chỉ số tỷ lệ từ chối (Deny Rate) theo từng khung giờ.

---

## 📝 7. THỎA THUẬN KIẾN TRÚC SƠ BỘ CHO LAB 02 (Đã nghiệm thu)

- **Ghi nhận Dependency Map:** Đăng ký thành công danh sách và cấu trúc định danh của cả 2 sự kiện (`access.log.created`, `access.denied`) vào bản đồ phụ thuộc tổng thể của hệ thống Smart Campus.
- **Kỹ thuật nâng cao OpenAPI 3.1:** Đối với trường thông tin ghi chú bổ sung khi từ chối truy cập (`denialReasonDetails`), hai bên thống nhất cấu hình dưới dạng union type với `null` (`type: [string, "null"]`) để tối ưu hóa dung lượng truyền tải trên Queue khi không có diễn giải đặc biệt từ hệ thống phần cứng.
- **Trạng thái:** Hoàn tất phiên đàm phán, đại diện hai nhóm đã ký nghiệm thu sơ bộ (Sign-off), sẵn sàng làm căn cứ thiết kế hợp đồng AsyncAPI chi tiết tại bài Lab 03.

## 8. Ghi chú phạm vi Lab 02

Cặp này là Queue async. Trong Lab 02, hai bên chưa cần viết AsyncAPI đầy đủ. Tuy nhiên, cần ghi rõ thỏa thuận sơ bộ trong `negotiation-log.md` hoặc dùng `docs/event-contract-template.md` nếu giảng viên yêu cầu. (Trạng thái: **Đã hoàn thành 100% — Pass hoàn toàn hệ thống Git check**).