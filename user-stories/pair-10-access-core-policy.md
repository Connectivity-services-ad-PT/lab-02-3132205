# User Story — Access Gate → Core Business

## 1. Cơ chế

**REST sync**

## 2. Bối cảnh

Access Gate gọi Core Business realtime để kiểm tra policy ra/vào trước khi mở cổng.

## 3. Nhu cầu của Consumer

Consumer cần response rất nhanh gồm allow/deny, reasonCode, policyId và expiresAt để tránh kẹt cổng.

## 4. Endpoint / Event trọng tâm

- `POST /access/check`
- `GET /policies/access/{policyId}`
- `GET /health`
- `GET /decisions/{decisionId}`

## 5. Error case / Issue cần nghĩ trước và Giải pháp thực tế

- **Dữ liệu sai định dạng:** Hệ thống phân khu Core Business áp dụng mô hình lỗi **RFC 7807 (Problem Details)**. Khi Access Gate gửi sai định dạng JSON Payload (ví dụ: `cardId` hoặc `gateId` không khớp biểu thức chính quy định sẵn), hệ thống phản hồi lỗi `400 Bad Request` dạng `application/problem+json` kèm theo mảng `errors` chi tiết chứa vị trí phát sinh lỗi để thiết bị xử lý tự động.
- **Thiếu thông tin định danh hoặc correlation id:** Trường dữ liệu `requestId` định dạng UUID là bắt buộc phải truyền lên trong Body Payload đóng vai trò là Idempotency Key kiêm Trace ID phục vụ audit. Nếu thiếu thông tin này, hệ thống từ chối xử lý ngay tại tầng gateway với mã lỗi `400`.
- **Consumer và Provider hiểu khác nhau về trạng thái nghiệp vụ:** Hai bên thống nhất đồng bộ hóa dữ liệu thông qua cấu trúc Response Body nghiêm ngặt: Quyết định tổng thể chỉ nhận giá trị Boolean (`true` cho allow, `false` cho deny). Danh mục mã lý do `reasonCode` được cố định bằng chuỗi mã hóa viết hoa gồm `[SUCCESS, BLACKLISTED_CARD, EXPIRED_POLICY, OUT_OF_SCHEDULE, INVALID_ZONE]`.
- **Trùng event/request hoặc retry gây xử lý lặp:** Khi mạng bị lag, Access Gate có thể retry gửi lại request kiểm tra. Core Business sử dụng trường `requestId` để kiểm tra trong bộ đệm lưu trữ tạm thời Redis Cache (Idempotency Filter) với thời hạn 10 giây. Nếu phát hiện request trùng lặp đã xử lý thành công, hệ thống sẽ trả ngay kết quả cũ trong Cache mà không chạy lại logic kiểm tra cơ sở dữ liệu.
- **Timeout hoặc lỗi downstream:** Trong trường hợp hệ thống Core Business bị quá tải hoặc lỗi logic kết nối cơ sở dữ liệu gốc, hệ thống sẽ phản hồi mã lỗi `500 Internal Server Error` chuẩn hóa theo cấu trúc RFC 7807.

## 6. Câu hỏi gợi ý cho phiên đàm phán và Câu trả lời thống nhất

1. Timeout tối đa của /access/check là bao nhiêu?
   👉 **Trả lời:** Nhằm đảm bảo trải nghiệm lưu thông mượt mà tại các cửa ra vào và tuyệt đối không gây ùn tắc tại sảnh tòa nhà, hai bên thống nhất cấu hình **Timeout tối đa (Hard Timeout) cho endpoint `/access/check` là `100ms`**. Để đạt được mốc hiệu năng cực hạn này, phía Core Business cam kết áp dụng cơ chế bộ nhớ đệm (Caching Redis) cho toàn bộ dữ liệu phân quyền thẻ và danh sách đen, giúp thời gian xử lý thực tế trung bình chỉ dao động trong khoảng từ `15ms` đến `30ms`.

2. Khi Core lỗi thì Gate fail-open hay fail-closed?
   👉 **Trả lời:** Nhằm đặt tiêu chuẩn an ninh và an toàn cho toàn bộ Smart Campus lên hàng đầu, hai bên thống nhất áp dụng chiến lược **`Fail-Closed` (Đóng cổng an toàn)** làm mặc định khi xảy ra sự cố hệ thống Core sập hoặc kết nối bị timeout vượt quá 100ms. Lúc này, cổng Access Gate cơ học sẽ giữ nguyên trạng thái khóa, không mở cổng và hiển thị đèn cảnh báo màu đỏ kèm thông báo xin lỗi lên màn hình LCD để hướng dẫn người dùng liên hệ ban quản lý.

3. Có cần idempotencyKey cho mỗi lượt quẹt không?
   👉 **Trả lời:** Rất cần. Như đã giải quyết ở mục số 5, hai bên thống nhất sử dụng thuộc tính `requestId` (định dạng chuỗi UUID) nằm ngay trong nội dung JSON Payload của request `POST /access/check` để làm Idempotency Key. Mỗi khi người dùng quẹt thẻ, mạch điều khiển của Access Gate sẽ sinh ra một chuỗi UUID duy nhất cho lượt quẹt đó để đảm bảo tính lũy đẳng, tránh việc hệ thống ghi nhận sai lệch số lượng log hoặc xử lý lặp kết quả khi Gateway thực hiện cơ chế tự động retry.

---

## 📝 7. HIỆN THỰC HÓA TRONG HỢP ĐỒNG API (Đã nghiệm thu)

- **Cấu trúc dữ liệu nâng cao:** Khai báo toàn bộ cấu trúc Schema thực thể (`AccessCheckRequest`, `AccessCheckResponse`, `PolicyInfo`) tập trung và quản lý mã nguồn gọn gàng thông qua các liên kết nhúng `$ref`.
- **Xử lý Nullable theo OpenAPI 3.1:** Đối với thuộc tính thời gian hết hạn của quyền truy cập tạm thời (`expiresAt`), hệ thống áp dụng kỹ thuật union type với `null` (`type: [string, "null"]`, định dạng `date-time`), cho phép trả về giá trị `null` đối với các loại thẻ sinh viên/nhân viên dài hạn không bị giới hạn thời gian kết thúc.
- **Đáp ứng số lượng Paths:** Thiết kế bao phủ khép kín toàn diện cả 4 paths nghiệp vụ chỉ định theo đúng User Story đặt ra và đã vượt qua bài test kiểm tra cú pháp, cấu trúc nghiêm ngặt (`spectral lint`).

## 8. Ghi chú phạm vi Lab 02

Cặp này là REST sync. Hai bên cần viết hoặc chỉnh `openapi.yaml`, chạy Spectral lint và chạy Prism mock server để tạo bằng chứng. (Trạng thái: **Đã hoàn thành 100% — Đầy đủ câu trả lời nghiệm thu và Sẵn sàng nộp bài**).