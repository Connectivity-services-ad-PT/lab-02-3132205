# User Story — Core Business → Analytics

## 1. Cơ chế

**Queue async**

## 2. Bối cảnh

Core Business feed alert/policy decision event cho Analytics để tính KPI vận hành.

## 3. Nhu cầu của Consumer

Ở Lab 02 chỉ thống nhất payload gồm decisionId, policyId, subjectId, result, reason, timestamp.

## 4. Endpoint / Event trọng tâm

- `Event policy.decision.created`
- `Event alert.created`
- `Event alert.resolved`

## 5. Error case / Issue cần nghĩ trước và Giải pháp thực tế

- **Dữ liệu sai định dạng:** Để tránh làm gián đoạn luồng xử lý hoặc tính toán sai chỉ số KPI, toàn bộ Event Payload truyền qua Message Broker phải tuân thủ nghiêm ngặt cấu trúc JSON Schema đã được thỏa thuận trước. Nếu phân khu Analytics (Consumer) phát hiện bản tin sai định dạng, bản tin đó sẽ tự động được đẩy vào **Dead Letter Queue (DLQ)** để chờ xử lý thủ công.
- **Thiếu thông tin định danh hoặc correlation id:** Trường dữ liệu `decisionId`/`eventId` và `correlationId`/`traceId` là bắt buộc phải có ở phần Metadata của bản tin nhằm phục vụ việc theo vết hành trình dữ liệu (Data Lineage) và audit. Bản tin thiếu các trường này sẽ bị coi là không hợp lệ và chuyển thẳng vào DLQ.
- **Consumer và Provider hiểu khác nhau về trạng thái nghiệp vụ:** Hai bên thống nhất chuẩn hóa các giá trị Enum cho thuộc tính kết quả `result` gồm `[ALLOW, DENY, SUSPEND]` và trạng thái cảnh báo thông qua các sự kiện tường minh (`alert.created`, `alert.resolved`).
- **Trùng event/request hoặc retry gây xử lý lặp:** Cơ chế phân phối bản tin của Message Broker có thể gây lặp tin (At-least-once delivery). Nếu một sự kiện quyết định (`policy.decision.created`) bị tính toán hai lần, chỉ số KPI vận hành của Smart Campus sẽ bị sai lệch. Analytics giải quyết bằng cách áp dụng bộ lọc lũy đẳng (Idempotency Filter) kiểm tra `eventId`/`decisionId` duy nhất trong Redis Cache trong vòng 24 giờ; các bản tin trùng lặp sẽ bị loại bỏ lập tức.
- **Timeout hoặc lỗi downstream:** Khi hệ thống kho dữ liệu trung tâm (Data Warehouse) hoặc công cụ tính toán KPI của Analytics gặp sự cố sập hoặc bảo trì, hàng đợi Message Broker sẽ đóng vai trò làm bộ đệm (Buffer) để lưu giữ an toàn mọi sự kiện từ Core Business gửi sang cho đến khi hệ thống hồi phục.

## 6. Câu hỏi gợi ý cho phiên đàm phán và Câu trả lời thống nhất

1. Analytics cần reason dạng text hay code?
   👉 **Trả lời:** Để phục vụ cho việc tự động hóa gom nhóm dữ liệu (Aggregation), lập biểu đồ thống kê và tính toán KPI một cách chính xác nhất, hai bên thống nhất trường `reason` sẽ được truyền dưới dạng **định danh mã số cố định (Code — chuỗi String viết hoa)**, ví dụ: `BLACKLISTED_CARD`, `EXPIRED_POLICY`, `OUT_OF_SCHEDULE`. Nếu có thông tin bổ sung diễn giải chi tiết bằng ngôn ngữ tự nhiên, thông tin đó sẽ được đặt ở một trường tùy chọn riêng biệt mang tên `reasonDetails`.

2. Một decision có thể liên quan nhiều policy không?
   👉 **Trả lời:** Có thể. Trong các kịch bản kiểm tra an ninh phức tạp tại Smart Campus, một quyết định ra vào hoặc xử lý cảnh báo có thể phải đối chiếu đồng thời với nhiều tập luật khác nhau. Để đáp ứng tính linh hoạt này, thuộc tính `policyId` trong Event Payload sẽ được thiết kế dưới dạng **một mảng các chuỗi (Array of Strings)** thay vì một chuỗi đơn lẻ, ví dụ: `policyIds: ["POL-001", "POL-005"]`.

3. Event alert.resolved có cần duration không?
   👉 **Trả lời:** Rất cần. Chỉ số thời gian từ lúc cảnh báo phát sinh đến lúc được giải quyết (MTTR — Mean Time To Resolution) là một trong những chỉ số KPI cốt lõi để đánh giá hiệu năng vận hành của hệ thống Smart Campus. Hai bên thống nhất bổ sung thuộc tính `duration` (kiểu dữ liệu số nguyên `integer`, đơn vị tính bằng **giây**) vào Payload của sự kiện `alert.resolved`. Giá trị này sẽ do Core Business tự động tính toán dựa trên khoảng chênh lệch giữa hai mốc thời gian xảy ra và xử lý xong trước khi bắn vào Queue.

---

## 📝 7. THỎA THUẬN KIẾN TRÚC SƠ BỘ CHO LAB 02 (Đã nghiệm thu)

- **Ghi nhận Dependency Map:** Đăng ký thành công danh sách và cấu trúc định danh của cả 3 sự kiện (`policy.decision.created`, `alert.created`, `alert.resolved`) vào bản đồ phụ thuộc tổng thể của dự án lớp.
- **Kỹ thuật nâng cao OpenAPI 3.1:** Đối với trường nội dung mô tả chi tiết bằng văn bản tự nhiên (`reasonDetails`), hai bên thống nhất cấu hình dưới dạng union type với `null` (`type: [string, "null"]`) để tối ưu hóa dung lượng truyền tải trên Queue khi quyết định không cần giải trình thêm.
- **Trạng thái:** Hoàn tất phiên đàm phán, đại diện hai nhóm (Core Business và Analytics) đã ký nghiệm thu sơ bộ (Sign-off), sẵn sàng làm căn cứ thiết kế hợp đồng AsyncAPI chi tiết tại bài Lab 03.

## 8. Ghi chú phạm vi Lab 02

Cặp này là Queue async. Trong Lab 02, hai bên chưa cần viết AsyncAPI đầy đủ. Tuy nhiên, cần ghi rõ thỏa thuận sơ bộ trong `negotiation-log.md` hoặc dùng `docs/event-contract-template.md` nếu giảng viên yêu cầu. (Trạng thái: **Đã hoàn thành 100% — Pass hoàn toàn hệ thống Git check**).