# Event Contract sơ bộ — dùng cho dependency Queue async

> File này chỉ dùng cho các cặp Queue async ở Lab 02 để ghi nhận thỏa thuận ban đầu. Đặc tả chi tiết bằng AsyncAPI sẽ chuyển sang Lab 03.

## 1. Thông tin dependency

- **Dependency số:** Pair 04
- **Producer:** Core Business Service (Dịch vụ quản lý nghiệp vụ trung tâm)
- **Consumer:** Notification Service (Dịch vụ thông báo đa kênh)
- **Cơ chế:** Queue async (Hàng đợi thông điệp bất đồng bộ)
- **Event/topic dự kiến:** `campus.core.access.denied`
- **Người ghi:** Nhóm kiến trúc hạ tầng Smart Campus
- **Ngày:** 03/06/2026

---

## 2. Mục đích nghiệp vụ

Thông điệp bất đồng bộ này được sinh ra (Publish) ngay khi hệ thống Core Business thẩm định và đưa ra quyết định từ chối một lượt quẹt thẻ ra vào (ví dụ: phát hiện thẻ giả mạo, thẻ nằm trong danh sách đen hoặc sinh viên cố tình đi vào khu vực cấm). 

Dịch vụ Notification (Consumer) sau khi nhận được sự kiện này sẽ tiến hành xử lý và ngay lập tức gửi cảnh báo đẩy (Push Notification) qua ứng dụng Mobile của sinh viên, đồng thời gửi tin nhắn khẩn cấp về ứng dụng của Đội bảo vệ Campus (Security Dashboard) để kịp thời can thiệp.

---

## 3. Event name / topic

| Mục | Giá trị |
|---|---|
| **Event name** | `campus.core.access.denied` |
| **Topic/queue** | `q.campus.core.notification.access-denied` |
| **Producer** | `core-business-service` |
| **Consumer** | `notification-service` |

---

## 4. Payload tối thiểu

```json
{
  "eventId": "f81d4fae-7dec-11d0-a765-00a0c91e6bf6",
  "eventType": "campus.core.access.denied",
  "occurredAt": "2026-06-03T17:45:00.123Z",
  "correlationId": "req-998877-gate01",
  "source": "core-business-service",
  "data": {
    "cardId": "473a218d8e5b412ea0c01a6134b7f5c7",
    "gateId": "GATE-BLOCK-A-01",
    "reasonCode": "BLACKLISTED_CARD",
    "zoneId": "ZONE-HIGH-SECURITY",
    "severity": "CRITICAL"
  }
}