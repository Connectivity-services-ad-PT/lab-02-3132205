# User Story — Core Business → Access Gate

## 1. Cơ chế

**REST sync**

## 2. Bối cảnh

Core Business cần truy xuất log quẹt thẻ, trạng thái gate hoặc thông tin card từ Access Gate để kiểm tra/audit.

## 3. Nhu cầu của Consumer

Consumer cần dữ liệu access log có cardId, gateId, direction, timestamp, status và operator note nếu có.

## 4. Endpoint / Event trọng tâm

- `GET /access/logs/recent`
- `GET /access/logs/{logId}`
- `GET /gates/{gateId}/status`
- `GET /cards/{cardId}`

## 5. Error case / Issue cần nghĩ trước và Giải pháp thực tế

- **Dữ liệu sai định dạng:** Hệ thống phân khu Access Gate áp dụng mô hình lỗi **RFC 7807 (Problem Details)**. Khi Consumer truyền sai định dạng query parameter hoặc path parameter (ví dụ: `cardId` hoặc `gateId` sai quy tắc mã hóa regex), hệ thống phản hồi lỗi `400 Bad Request` dạng `application/problem+json` kèm mảng `errors` chi tiết.
- **Thiếu thông tin định danh hoặc correlation id:** Mọi request audit từ Core Business lên hệ thống Access Gate bắt buộc phải đính kèm `traceId` hoặc `correlationId` trên tiêu đề (HTTP Header) hoặc query string phục vụ chuỗi audit tổng thể. Nếu thiếu, hệ thống trả mã lỗi `400 Bad Request`.
- **Consumer và Provider hiểu khác nhau về trạng thái nghiệp vụ:** Thống nhất các giá trị Enum nghiêm ngặt: Hướng di chuyển `direction` chỉ gồm `[IN, OUT]`; trạng thái quẹt thẻ `status` chỉ gồm `[GRANTED, DENIED, LOCKED]`.
- **Trùng event/request hoặc retry gây xử lý lặp:** Vì đây là các hành động truy xuất dữ liệu (`GET`), thuộc tính an toàn (Safe) và lũy đẳng (Idempotent) được đảm bảo theo đặc tả HTTP. Việc Consumer retry nhiều lần do lag mạng không làm thay đổi trạng thái hay tạo log mới trên thiết bị.
- **Timeout hoặc lỗi downstream:** Trong trường hợp thiết bị cổng Access Gate cơ học bị mất kết nối mạng ngoại vi (Offline) không thể phản hồi về gateway, hệ thống sẽ trả về mã lỗi `504 Gateway Timeout` hoặc `502 Bad Gateway` chuẩn cấu trúc RFC 7807.

## 6. Câu hỏi gợi ý cho phiên đàm phán và Câu trả lời thống nhất

1. Log lưu ở Access Gate bao lâu?
   👉 **Trả lời:** Do bộ nhớ lưu trữ tại chỗ của các thiết bị Access Gate có giới hạn, log quẹt thẻ thô sẽ chỉ được lưu cục bộ trong vòng **30 ngày**. Hết thời hạn này, log cũ sẽ bị ghi đè. Hệ thống khuyến nghị Core Business định kỳ đồng bộ dữ liệu về kho lưu trữ trung tâm qua luồng Async trong bài Lab sau.

2. Core có được query theo khoảng thời gian không?
   👉 **Trả lời:** Có. Đối với endpoint `GET /access/logs/recent`, hai bên thống nhất bổ sung 2 query parameters tùy chọn (optional) là `startTime` và `endTime` sử dụng định dạng chuỗi chuẩn `date-time` (ISO 8601 UTC) để Core dễ dàng lọc dữ liệu audit theo mốc thời gian mong muốn.

3. Card bị khóa thì Access Gate trả trạng thái nào?
   👉 **Trả lời:** Khi truy xuất thông tin qua `GET /cards/{cardId}` mà thẻ đang nằm trong danh sách đen (Blacklist) hoặc bị khóa, hệ thống vẫn trả về mã trạng thái `200 OK` nhưng thuộc tính `status` trong Response Body sẽ mang giá trị là `LOCKED`. Trong trường hợp thẻ hoàn toàn không tồn tại trên hệ thống, API sẽ phản hồi mã lỗi `404 Not Found`.

---

## 📝 7. HIỆN THỰC HÓA TRONG HỢP ĐỒNG API (Đã nghiệm thu)

- **Cấu trúc dữ liệu nâng cao:** Triển khai toàn bộ các mô hình thực thể dữ liệu (`AccessLog`, `GateStatus`, `CardInfo`) tập trung trong `components/schemas` và nhúng qua cơ chế `$ref` tinh gọn.
- **Xử lý Nullable theo OpenAPI 3.1:** Trường thông tin ghi chú của người vận hành (`operatorNote`) được định nghĩa bằng cấu trúc union type với `null` (`type: [string, "null"]`), cho phép trả về giá trị `null` một cách minh bạch khi lượt quẹt thẻ tự động không có ghi chú thủ công.
- **Đáp ứng số lượng Paths:** Thiết kế bao phủ trọn vẹn cả 4 paths nghiệp vụ chỉ định theo đúng User Story này và đã vượt qua bài test kiểm tra cấu trúc cú pháp nghiêm ngặt (`npm run lint`).

## 8. Ghi chú phạm vi Lab 02

Cặp này là REST sync. Hai bên cần viết hoặc chỉnh `openapi.yaml`, chạy Spectral lint và chạy Prism mock server để tạo bằng chứng. (Trạng thái: **Đã hoàn thành 100% — Đầy đủ câu trả lời nghiệm thu**).