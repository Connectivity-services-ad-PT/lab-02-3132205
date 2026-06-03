# User Story — Camera Stream → AI Vision

## 1. Cơ chế
**REST sync**

## 2. Bối cảnh
Camera Stream gửi frame hoặc image metadata khi phát hiện motion để AI Vision chạy detect.

## 3. Nhu cầu của Consumer
Consumer cần response nhanh gồm detectionId, objects, confidence và riskLevel để chuyển tiếp cho Core Business khi có bất thường.

## 4. Endpoint / Event trọng tâm
- `POST /vision/detect`
- `GET /vision/detections/{detectionId}`
- `GET /vision/models/info`
- `GET /health`

## 5. Error case / Issue cần nghĩ trước
- Dữ liệu sai định dạng.
- Thiếu thông tin định danh hoặc correlation id.
- Consumer và Provider hiểu khác nhau về trạng thái nghiệp vụ.
- Trùng event/request hoặc retry gây xử lý lặp.
- Timeout hoặc lỗi downstream.

## 6. Câu hỏi gợi ý cho phiên đàm phán
1. Ảnh gửi dạng multipart hay URL?
   👉 **Trả lời:** Hai bên thống nhất sử dụng cấu trúc `imagePayload` áp dụng kỹ thuật **`oneOf` + `discriminator`** trong OpenAPI 3.1. Cơ chế này cho phép Consumer linh hoạt truyền dữ liệu theo 2 cách: Gửi đường dẫn tĩnh (`imageType: URL`) hoặc truyền chuỗi nhị phân mã hóa (`imageType: BASE64`) tùy thuộc vào độ ổn định mạng của camera.

2. Giới hạn kích thước frame là bao nhiêu?
    **Trả lời:** Giới hạn dung lượng tối đa cho mỗi yêu cầu xử lý ảnh (cả URL payload và chuỗi BASE64) là **`5MB`**. Mọi tệp ảnh vượt quá dung lượng này sẽ bị hệ thống từ chối ngay lập tức với mã lỗi `400 Bad Request` nhằm tránh làm nghẽn mạch hoặc quá tải bộ đệm vùng nhận diện của AI.

3. AI Vision trả kết quả đồng bộ hay trả detectionId để polling?
    **Trả lời:** Thống nhất áp dụng cơ chế **Xử lý đồng bộ trực tiếp (REST sync)**. Khi Consumer gửi ảnh qua `POST /vision/detect`, AI Vision Service sẽ tính toán nhanh và trả trực tiếp kết quả (gồm tọa độ `boundingBox`, nhãn vật thể và độ tin cậy) ngay trong nội dung phản hồi (Response Body) với thời gian trễ cam kết dưới `500ms`, hoàn toàn loại bỏ cơ chế polling.

---

## 📝 7. KẾT QUẢ ĐÀM PHÁN VÀ GIẢI PHÁP CHO ERROR CASE (Đã hoàn thiện)

- **Xử lý dữ liệu sai định dạng / Sai Regex:** Áp dụng mô hình lỗi **RFC 7807 (Problem Details)**, phản hồi mã lỗi `400 Bad Request` kèm theo mảng `errors` chi tiết (ví dụ: `body.cameraId` không khớp mẫu `^CAM-[0-9]{3}$`).
- **Thiếu thông tin định danh:** Bắt buộc trường `requestId` định dạng UUID đóng vai trò là Idempotency Key. Nếu thiếu, hệ thống trả về mã lỗi `400` ngay tại gateway.
- **Trùng lặp request khi retry:** AI Vision sử dụng `requestId` làm khóa lưu trạng thái vào Cache tạm thời. Nếu phát hiện trùng lặp request đang xử lý, hệ thống trả về `409 Conflict` để tránh chạy lại model AI gây lãng phí tài nguyên GPU.
- **Timeout hoặc lỗi downstream:** Nếu cụm GPU quá tải hoặc xảy ra lỗi logic backend, hệ thống phản hồi lỗi `500 Internal Server Error` chuẩn định dạng `application/problem+json`.

## 8. Ghi chú phạm vi Lab 02
Cặp này là REST sync. Hai bên cần viết hoặc chỉnh `openapi.yaml`, chạy Spectral lint và chạy Prism mock server để tạo bằng chứng. (Trạng thái: **Đã hoàn thành 100% — Đầy đủ câu trả lời nghiệm thu**).