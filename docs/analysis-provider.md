# Phân tích yêu cầu — vai Provider

- **Cặp đàm phán:** Pair 10
- **Product:** Hệ thống quản lý và kiểm soát an ninh ra vào Smart Campus
- **Provider service:** Core Business Service (Dịch vụ quản lý chính sách và quyết định)
- **Consumer service:** Access Gate Service (Dịch vụ kiểm soát cổng vật lý)
- **Người viết:** Nhóm phát triển Core Business
- **Ngày:** 03/06/2026

---

## 1. Resource chính

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `AccessDecision` | Kết quả thẩm định quyền ra/vào của một lượt quẹt thẻ cụ thể sau khi chạy qua công cụ luật chính sách. | `requestId`, `allow`, `reasonCode` | `policyId`, `expiresAt` |
| `AccessPolicy` | Tập luật cấu hình điều kiện an ninh (Ví dụ: danh sách khu vực, khung giờ, đối tượng sinh viên/giảng viên được phép đi qua). | `policyId`, `policyName`, `status`, `zoneId` | `description`, `effectiveDates` |
| `SystemHealth` | Trạng thái sức khỏe và độ sẵn sàng của dịch vụ Core cùng các kết nối downstream liên quan (Cơ sở dữ liệu, Redis Cache). | `status` | `timestamp`, `components` |

---

## 2. Action/API dự kiến

| Method | Path | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| `POST` | `/access/check` | Thẩm định thời gian thực và trả về quyết định cho phép mở cổng (`allow: true`) hoặc từ chối (`allow: false`). | Khi có sinh viên/nhân viên quẹt thẻ vật lý hoặc quét mã QR an ninh tại cửa kiểm soát Access Gate. |
| `GET` | `/policies/access/{policyId}` | Cung cấp thông tin cấu hình chi tiết của một tập luật phân quyền cụ thể để kiểm tra tính hiệu lực. | Khi thiết bị Access Gate cần kiểm tra cấu hình hệ thống hoặc đồng bộ dữ liệu giám sát định kỳ. |
| `GET` | `/decisions/{decisionId}` | Truy xuất lại toàn bộ thông tin lịch sử của một lượt quẹt thẻ đã được hệ thống lưu vết. | Khi người quản trị vận hành cần đối chiếu dữ liệu log, giải quyết khiếu nại hoặc điều tra sự cố kẹt cổng. |
| `GET` | `/health` | Phản hồi trạng thái hoạt động hiện tại của phân khu Core để phục vụ giám sát hạ tầng. | Thiết bị Access Gate tự động gọi định kỳ (Heartbeat) để xác định xem hệ thống trung tâm có đang trực tuyến hay không. |

---

## 3. Error case

> Toàn bộ cấu trúc response body lỗi dưới đây đều tuân thủ nghiêm ngặt mô hình **Problem Details (RFC 7807)** sử dụng header `application/problem+json`.

| Status | Tình huống | Response body dự kiến |
|---:|---|---|
| **400** | Payload gửi lên sai định dạng cấu trúc JSON, thiếu thuộc tính bắt buộc `cardId` hoặc mã `requestId` không phải định dạng UUID. | `{"type": "https