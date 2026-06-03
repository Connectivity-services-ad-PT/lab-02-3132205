# Phân tích yêu cầu — vai Consumer

- **Cặp đàm phán:** Pair 10
- **Product:** Hệ thống quản lý và kiểm soát an ninh ra vào Smart Campus
- **Consumer service:** Access Gate Service (Dịch vụ kiểm soát cổng vật lý)
- **Provider service:** Core Business Service (Dịch vụ quản lý chính sách và quyết định)
- **Người viết:** Nhóm phát triển Access Gate
- **Ngày:** 03/06/2026

---

## 1. Resource Consumer cần nhận/gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `AccessCheckRequest` | Đóng gói thông tin thẻ quẹt, mã định danh cổng và thời gian thực tế để gửi lên Core Business xin quyền mở cổng. | `requestId`, `cardId`, `gateId`, `timestamp` | Không có |
| `AccessCheckResponse` | Nhận kết quả thẩm định chính sách từ Core Business để điều khiển rơ-le mở cửa cơ học hoặc chặn lại. | `allow`, `reasonCode` | `policyId`, `expiresAt` |
| `PolicyInfo` | Truy vấn và đồng bộ thông tin chi tiết của cấu hình tập luật phân quyền để phục vụ ghi vết log dưới Gate. | `policyId`, `policyName`, `status` | `description`, `rules` |
| `DecisionLog` | Tra cứu lại bản ghi chi tiết của một quyết định kiểm tra an ninh cụ thể khi cần đối chiếu hoặc xử lý sự cố. | `decisionId`, `timestamp`, `result` | `operatorNote` |

---

## 2. API Consumer cần gọi

| Method | Path | Lúc nào gọi? | Kỳ vọng response |
|---|---|---|---|
| `POST` | `/access/check` | Gọi thời gian thực (realtime) ngay khi có sự kiện sinh viên hoặc cán bộ quẹt thẻ/quét mã tại cổng Access Gate. | `200 OK` chứa trạng thái `allow: true/false` và mã lý do cụ thể với thời gian phản hồi cực nhanh (<100ms). |
| `GET` | `/policies/access/{policyId}` | Gọi khi hệ thống khởi động hoặc cần đồng bộ danh sách luật phân quyền từ trung tâm xuống thiết bị ngoại vi. | `200 OK` trả về chi tiết cấu hình và trạng thái hiệu lực của chính sách bảo mật được chỉ định. |
| `GET` | `/decisions/{decisionId}` | Gọi khi người quản lý tại cổng cần bấm kiểm tra hoặc truy vết lại một quyết định ra vào gây tranh chấp. | `200 OK` chứa toàn bộ siêu dữ liệu (metadata) của lượt quyết định đó từ Core Business. |
| `GET` | `/health` | Gọi định kỳ (Cơ chế Heartbeat cứ mỗi 30 giây) để kiểm tra trạng thái sống sót và kết nối kết nối đến Core Business. | `200 OK` kèm payload trạng thái `status: "UP"` để bảo đảm hệ thống trực tuyến. |

---

## 3. Error case Consumer cần xử lý

> Toàn bộ các trường hợp lỗi dưới đây đều được đồng bộ xử lý tự động dựa trên cấu trúc mô tả lỗi tiêu chuẩn định dạng **Problem Details (RFC 7807)** từ hệ thống Provider gửi về.

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---:|---|---|
| **400** | Request sai schema (ví dụ: `cardId` hoặc `gateId` gửi sai định dạng chuỗi UUID/Regex quy định). | Từ chối xử lý, không kích hoạt relay mở cổng, ghi nhận lỗi vào log cục bộ và hiển thị thông báo "Thẻ không hợp lệ" lên màn hình LCD. |
| **401** | Thiếu token hoặc sai cấu hình khóa bảo mật (Access Token) của thiết bị Access Gate gửi lên Core Business. | Khóa cổng cơ học, gửi cảnh báo khẩn cấp về trung tâm giám sát (Dashboard hệ thống) để kỹ thuật viên kiểm tra lại cấu hình kết nối. |
| **403** | Quyết định ra vào bị từ chối từ hệ thống phân quyền của Core Business (Do thẻ nằm trong danh sách đen hoặc ngoài khung giờ ra vào). | Giữ nguyên trạng thái khóa cổng, chớp đèn LED màu đỏ, phát âm thanh cảnh báo ngắn (Beep) và hiển thị mã lý do `reasonCode` tương ứng lên màn hình. |
| **404** | Không tìm thấy tài nguyên hệ thống (Ví dụ: Tra cứu một mã chính sách `policyId` hoặc mã quyết định `decisionId` không tồn tại). | Hiển thị thông báo lỗi cấu hình dữ liệu lên màn hình điều khiển và từ chối thực hiện tác vụ liên quan. |
| **409** | Xung đột nghiệp vụ dữ liệu thời gian thực (Ví dụ: Thẻ quẹt liên tiếp nhiều lần tại cùng một cổng trong thời gian dưới 1 giây - lỗi Anti-passback). | Bỏ qua các yêu cầu quẹt lặp, yêu cầu người dùng chờ từ 3 đến 5 giây rồi thực hiện thao tác quẹt thẻ lại. |
| **422** | Vi phạm quy tắc nghiệp vụ hệ thống (Ví dụ: Thiết bị gửi yêu cầu kiểm tra nhưng mốc thời gian `timestamp` bị lệch quá xa so với giờ hệ thống server Core). | Đồng bộ lại thời gian thực của thiết bị Access Gate qua giao thức NTP và yêu cầu người dùng quẹt lại thẻ. |
| **500 / Timeout** | Hệ thống Core Business gặp sự cố sập nguồn, quá tải hoặc không phản hồi kết quả kiểm tra ra vào trong vòng **100ms**. | Áp dụng chiến lược an ninh tối cao **Fail-Closed (Đóng cổng an toàn)**. Cổng sẽ khóa chặt để bảo vệ khuôn viên Campus, đồng thời chuyển log quẹt thẻ vào bộ nhớ tạm Local Storage để chờ đồng bộ sau khi hệ thống khôi phục. |

---

## 4. Giả định bổ sung

- **Giả định 1:** Phân khu Core Business cam kết thời gian xử lý và phản hồi cho endpoint `/access/check` trung bình luôn dưới `30ms` nhờ áp dụng công nghệ lưu trữ dữ liệu phân quyền trên bộ nhớ đệm (Redis Cache), giúp tổng thời gian phản hồi ở thiết bị Gate luôn dưới mốc `100ms`.
- **Giả định 2:** Mã định danh thẻ (`cardId`) truyền tải trên đường truyền REST sync bắt buộc đã được băm bằng thuật toán bảo mật SHA-256 từ tầng phần cứng của Access Gate để tuân thủ quy định an toàn dữ liệu cá nhân của Smart Campus.
- **Giả định 3:** Định dạng chuỗi ngày tháng thời gian trao đổi giữa hai phân khu thống nhất sử dụng chuẩn ISO 8601 quốc tế (Định dạng: `YYYY-MM-DDTHH:mm:ss.sssZ`).

---

## 5. Câu hỏi cho Provider

1. Khi một chính sách phân quyền bị hủy kích hoạt (`status: "INACTIVE"`), hệ thống Core Business sẽ phản hồi mã lý do cụ thể nào trong thuộc tính `reasonCode` của endpoint `/access/check`?
2. Trong trường hợp hệ thống Core Business cần bảo trì định kỳ, endpoint `/health` sẽ trả về HTTP Status Code `503 Service Unavailable` trước bao nhiêu phút để Access Gate chủ động chuyển sang chế độ hoạt động ngoại tuyến (Offline Mode)?
3. Phía Provider có hỗ trợ cơ chế gộp nhiều yêu cầu kiểm tra mã thẻ quẹt (`Bulk Check`) trên một request đơn lẻ để giảm tải băng thông mạng vào các khung giờ cao điểm sinh viên đến trường đông hay không?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Provider tự ý thay đổi kiểu dữ liệu hoặc cấu trúc của các trường thông tin trong Response Body. | Hệ thống Consumer không thể bóc tách dữ liệu (Parse lỗi), gây treo mạch điều khiển hoặc kẹt cổng kiểm soát ra vào. | Hai bên cam kết chốt cứng định dạng thông qua các ràng buộc dữ liệu cụ thể (`type`, `format`, `pattern`) bằng JSON Schema nâng cao trong file `openapi.yaml`. Mọi thay đổi cấu trúc bắt buộc phải nâng cấp phiên bản API theo tài liệu SemVer quy định trong file `VERSIONING.md`. |
| Provider phản hồi thiếu mã lỗi nghiệp vụ hoặc chỉ trả về chuỗi text diễn giải thô bằng ngôn ngữ tự nhiên. | Hệ thống lập trình tự động của Consumer không thể nhận diện để đưa ra quyết định hiển thị thông báo LCD chuẩn xác cho sinh viên. | Thống nhất áp dụng nghiêm ngặt cấu trúc báo lỗi chi tiết theo chuẩn **RFC 7807 (Problem Details)** với header bắt buộc là `application/problem+json`. Tất cả các mã lỗi nghiệp vụ phải được chuẩn hóa thành chuỗi Enum viết hoa cố định (ví dụ: `EXPIRED_POLICY`, `INVALID_ZONE`) đặt trong trường `reasonCode`. |