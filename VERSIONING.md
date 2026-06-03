# API Versioning Policy — Smart Campus Project

## 1. Nguyên tắc đánh số phiên bản
Hệ thống tuân thủ nghiêm ngặt chuẩn **Semantic Versioning (SemVer) 2.0.0** với định dạng: `MAJOR.MINOR.PATCH`.
- **MAJOR (X.0.0):** Thay đổi lớn, phá vỡ tính tương thích ngược (Breaking Changes) của API.
- **MINOR (0.Y.0):** Bổ sung thêm tính năng mới, endpoint mới nhưng vẫn đảm bảo tương thích ngược (Backward Compatible).
- **PATCH (0.0.Z):** Sửa lỗi kỹ thuật nhỏ (Bug fixes), tối ưu hóa hiệu năng, không làm thay đổi cấu trúc dữ liệu truyền nhận.

## 2. Quản lý phiên bản trên Git & OpenAPI
- **OpenAPI Specification:** Phiên bản của API phải được khai báo tường minh tại trường `info.version` trong file `openapi.yaml` (Ví dụ: `version: 1.0.0`).
- **Git Tags:** Mỗi lần phát hành hoặc nghiệm thu một phiên bản hợp đồng API mới, các nhóm phải thực hiện gắn thẻ Git để lưu vết, ví dụ: `git tag -a v1.0.0 -m "Release Lab 02 API Spec"`.
- **Phạm vi Lab 02:** Thống nhất phiên bản khởi tạo cho toàn bộ các phân khu trong Smart Campus là `1.0.0`.