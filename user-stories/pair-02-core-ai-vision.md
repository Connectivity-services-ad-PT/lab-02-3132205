# User Story — Core Business → AI Vision

## 1. Cơ chế

**REST sync**

## 2. Bối cảnh

Core Business lấy hoặc yêu cầu kết quả phân tích ảnh/face-match từ AI Vision để ra quyết định nghiệp vụ.

## 3. Nhu cầu của Consumer

Consumer cần thông tin detection/face-match có confidence, modelVersion, timestamp và traceId để audit.

## 4. Endpoint / Event trọng tâm

- `POST /vision/face-match`
- `GET /vision/detections/{detectionId}`
- `GET /vision/results/recent`
- `GET /health`

## 5. Error case / Issue cần nghĩ trước và Giải pháp thực tế

- **Dữ liệu sai định dạng:** Áp dụng mô hình lỗi **RFC 7807 (Problem Details)**, hệ thống phản hồi mã lỗi `400 Bad Request` kèm theo mảng `errors` chi tiết chứa thông tin về vị trí phát sinh lỗi (`field`, `code`, `message`) để Core Business tự động bóc tách.
- **Thiếu thông tin định danh hoặc correlation id:** Trường `requestId` định dạng UUID là bắt buộc trong payload để đóng vai trò làm Idempotency Key kiêm Trace ID phục vụ audit. Nếu thiếu, hệ thống trả về mã lỗi `400 Bad Request` ngay tại gateway.
- **Consumer và Provider hiểu khác nhau về trạng thái nghiệp vụ:** Thống nhất danh mục nhãn vật thể nguy cơ cố định gồm `UNKNOWN_PERSON`, `INTRUSION`, `PLATE`, `WEAPON`, `FIRE`, `SMOKE`, `ACCIDENT`. Trạng thái tổng thể trả về chỉ nằm trong enum `[SUCCESS, FAILED]`.
- **Trùng event/request hoặc retry gây xử lý lặp:** AI Vision Service sử dụng `requestId` để kiểm tra trong bộ đệm lưu trữ tạm thời (Cache). Nếu phát hiện một `requestId` trùng lặp đang hoặc đã được xử lý thành công, hệ thống phản hồi lỗi `409 Conflict`.
- **Timeout hoặc lỗi downstream:** Trong trường hợp cụm GPU xử lý AI Model bị quá tải hoặc phản hồi chậm quá 500ms, hệ thống sẽ trả về mã lỗi `500 Internal Server Error` chuẩn định dạng `application/problem+json`.

## 6. Câu hỏi gợi ý cho phiên đàm phán và Câu trả lời thống nhất

1. Core gửi faceEmbedding hay imageRef?
   👉 **Trả lời:** Để giảm tải tính toán cho phía Core Business (Consumer), Core không cần tự trích xuất vector embedding mà sẽ gửi thông tin ảnh qua cấu trúc `imagePayload` (hỗ trợ cả dạng `URL` tĩnh hoặc chuỗi mã hóa `BASE64`). Phân khu AI Vision (Provider) sẽ tự nhận diện và trích xuất đặc trưng hình ảnh.

2. Ngưỡng confidence bao nhiêu thì coi là match?
   👉 **Trả lời:** Hai bên thống nhất cấu hình ngưỡng (Threshold) mặc định là `0.80`. Mọi kết quả phân tích có độ tự tin từ `0.80` trở lên mới được ghi nhận khớp với các nhãn định danh cụ thể.

3. Khi model không chắc chắn thì trả 200 với trạng thái low_confidence hay 422?
   👉 **Trả lời:** Thống nhất vẫn trả về trạng thái thành công (`201 Created` / `200 OK`) kèm theo nhãn phân loại mặc định là `UNKNOWN_PERSON` và độ tự tin thực tế (ví dụ: `0.55`), đồng thời bổ sung thông tin cảnh báo vào trường ghi chú mở rộng `notes`. Mã lỗi `422 Unprocessable Entity` chỉ áp dụng khi file ảnh đầu vào bị hỏng cấu trúc vật lý, mờ, nhiễu nặng dẫn đến việc AI không thể đưa vào luồng quét.

---

## 📝 7. HIỆN THỰC HÓA TRONG HỢP ĐỒNG API (Đã nghiệm thu)

- **Cấu trúc dữ liệu nâng cao:** Đã khai báo mô hình dữ liệu tập trung trong `components/schemas` và nhúng qua cơ chế `$ref`. 
- **Ứng dụng đa hình:** Sử dụng cấu trúc `oneOf` phối hợp với `discriminator` để phân tách linh hoạt hai kiểu nguồn ảnh đầu vào (`URL` và `BASE64`).
- **Xử lý giá trị trống:** Áp dụng union type với `null` (`type: [string, "null"]`) cho thuộc tính ghi chú mở rộng `notes` để tối ưu bộ nhớ khi phản hồi không có cảnh báo đặc biệt.
- **Đáp ứng số lượng Paths:** Thiết kế hoàn thiện toàn diện cả 4 paths nghiệp vụ yêu cầu, vượt qua bài quét linter với tập luật `campus-spectral.yaml`.

## 8. Ghi chú phạm vi Lab 02

Cặp này là REST sync. Hai bên cần viết hoặc chỉnh `openapi.yaml`, chạy Spectral lint và chạy Prism mock server để tạo bằng chứng. (Trạng thái: **Đã hoàn thành 100% — Đầy đủ bằng chứng nghiệm thu**).