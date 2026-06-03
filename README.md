
[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/SP1CQRF3)
# Bài tập Lab 02 - OpenAPI
Sinh viên thực hiện: 3132205
# FIT4110 — Lab 02 OpenAPI 3.1 Contract-First (Nhóm A4 - AI Vision)

Repo này dùng cho **Lab 02 — Thực hành đàm phán và viết OpenAPI 3.1** của học phần **Dịch vụ kết nối và công nghệ nền tảng (FIT4110)**.

Lab 02 nối tiếp trực tiếp Lab 01 trong repo `FIT4110_setup`:
- Lab 01: Sinh viên thiết lập môi trường và nộp `service-boundary.md`.
- Lab 02: Sinh viên chuyển Service Boundary thành **hợp đồng API** bằng `openapi.yaml`.
- Hai nhóm ở hai vai **Consumer** và **Provider** phải đàm phán trước khi code.
- Dependency Map đầy đủ của Smart Campus gồm **10 cặp phụ thuộc**: REST sync viết bằng OpenAPI trong Lab 02, Queue async ghi nhận để chuyển sang Lab 03.

> Nguyên tắc trọng tâm: **contract-first** — không viết code trước khi chốt hợp đồng API.

---

## 1. Kết quả hoàn thành của Nhóm A4 (AI Vision — Pair 02)

Tuân thủ nghiêm ngặt yêu cầu của bài Lab, cặp đàm phán **Pair 02 (Core Business A6 ↔ AI Vision A4)** đã hoàn thiện và nộp đầy đủ các artefact sau:

* **`openapi.yaml`**: Bản đặc tả hợp đồng API sử dụng **OpenAPI 3.1.0** chuẩn hóa toàn bộ các endpoint nghiệp vụ nhận diện hình ảnh của phân khu AI Vision.
* **`negotiation-log.md`**: Ghi lại lịch sử đàm phán giữa nhóm A4 và nhóm A6 với đầy đủ bối cảnh, đề xuất, quyết định và chữ ký nghiệm thu (sign-off) từ đại diện 2 bên.
* **`docs/analysis-provider.md`**: Bản phân tích độc lập dưới góc nhìn Provider của nhóm A4 (AI Vision).
* **`docs/analysis-consumer.md`**: Bản phân tích độc lập dưới góc nhìn Consumer của nhóm A6 (Core Business).
* **`evidence/buoi-02/spectral-report.txt`**: Báo cáo kiểm tra chất lượng tệp thiết kế, đảm bảo vượt qua (pass) linter của lớp.
* **`evidence/buoi-02/mock-screenshots/`**: Bộ 5 ảnh chụp màn hình minh chứng chạy thử nghiệm thành công 5 request mẫu bằng `curl` thông qua Prism Mock Server.
* **`VERSIONING.md`**: Tài liệu quy định chiến lược quản lý phiên bản hợp đồng API.

### Đánh giá tiêu chí kỹ thuật đạt được trong `openapi.yaml`:
- **Tối thiểu 4 paths nghiệp vụ:** Bao gồm `/health`, `/vision/detect`, `/vision/detect/{requestId}`, và `/vision/results/recent`.
- **Tách biệt Schemas:** Toàn bộ cấu trúc dữ liệu được quản lý tập trung trong `components/schemas` và nhúng qua cơ chế `$ref`.
- **Kỹ thuật nâng cao OpenAPI 3.1:** - Cài cắm thành công cấu trúc **`oneOf` kết hợp `discriminator`** để phân tách luồng dữ liệu ảnh đầu vào (`imageType: URL` hoặc `imageType: BASE64`).
  - Áp dụng **union type với `null`** (`type: [string, "null"]`) tại trường ghi chú mở rộng (`notes`).
- **Chuẩn hóa thông báo lỗi:** Tất cả phản hồi mã lỗi 4xx/5xx đều được cấu hình theo chuẩn **`Problem Details` (RFC 7807)** thông qua kiểu nội dung `application/problem+json`.

---

## 2. Cấu trúc repo hiện tại

```text
FIT4110_lab02_openapi/
  README.md
  openapi.yaml
  campus-spectral.yaml
  negotiation-log.md
  package.json
  VERSIONING.md
  docs/
    lab02-guide.md
    pairing-matrix.md
    openapi-authoring-guide.md
    swagger-online-workflow.md
    event-contract-template.md
    analysis-provider.md
    analysis-consumer.md
  user-stories/
    README.md
    pair-02-core-ai-vision.md
  evidence/
    buoi-02/
      README.md
      checklist.md
      known-issues.md
      spectral-report.txt
      mock-screenshots/
        req-01-health.png
        req-02-detect-url.png
        req-03-detect-base64.png
        req-04-get-by-id.png
        req-05-recent-results.png
  scripts/
    install_lab02_cli.sh
    install_lab02_cli.ps1
    lint_openapi.sh
    lint_openapi.ps1
    mock_openapi.sh
    mock_openapi.ps1
    collect_session02_evidence.sh
    collect_session02_evidence.ps1
  .github/workflows/check-lab02.yml

```

---

## 3. Cài đặt công cụ

Yêu cầu tối thiểu:

* Git
* Node.js LTS phiên bản 20 trở lên
* npm
* **Swagger Editor Online** để soạn và xem trước `openapi.yaml`
* `curl` để gọi thử Prism mock server.

### Cài đặt qua CLI tùy theo hệ điều hành:

* **macOS/Linux:**
```bash
chmod +x scripts/*.sh
./scripts/install_lab02_cli.sh

```


* **Windows PowerShell:**
```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\scripts\install_lab02_cli.ps1

```



Hoặc sử dụng npm script được tích hợp sẵn:

```bash
npm run install:cli

```

---

## 4. Quy trình thực hành và kiểm tra (Pair 02)

### Bước 1. Đàm phán thiết kế và kiểm tra cú pháp với Linter

Nhóm A4 đưa tệp thiết kế `openapi.yaml` lên hệ thống kiểm tra quy tắc riêng của lớp bằng lệnh:

```bash
npm run lint

```

Báo cáo kiểm định chất lượng được xuất ra và lưu trữ tại `evidence/buoi-02/spectral-report.txt` thông qua lệnh tự động:

```bash
npm run lint:report

```

### Bước 2. Khởi chạy Mock Server độc lập bằng Prism

Chạy giả lập API dựa trên hợp đồng đã thống nhất tại cổng mặc định `4010`:

```bash
npm run mock

```

Server giả lập sẽ hoạt động tại địa chỉ: `http://localhost:4010`

### Bước 3. Thực hiện kiểm thử bằng lệnh `curl`

Tiến hành gọi thử nghiệm các request mẫu để kiểm tra tính chính xác của dữ liệu phản hồi (Response Body) và mã trạng thái (Status Code):

```bash
# Gọi kiểm tra sức khỏe hệ thống
curl -i http://localhost:4010/health

# Gửi yêu cầu phân tích dữ liệu hình ảnh (Yêu cầu Token xác thực)
curl -i -X POST http://localhost:4010/vision/detect \
  -H "Authorization: Bearer local-dev-token" \
  -H "Content-Type: application/json" \
  -d '{"requestId": "0196fb3d-4ad7-7d1e-9f49-5d5148d2babc", "cameraId": "CAM-001", "capturedAt": "2026-06-03T11:12:00Z", "imagePayload": {"imageType": "URL", "imageUrl": "[https://campus.local/images/cam-001/frame-123.jpg](https://campus.local/images/cam-001/frame-123.jpg)"}}'

```

*Toàn bộ hình ảnh ghi nhận kết quả kiểm thử đã được lưu đầy đủ vào thư mục `evidence/buoi-02/mock-screenshots/` đúng theo quy định.*

---

## 5. Lệnh kiểm tra nhanh cứu cánh

```bash
node --version
npm --version
spectral --version
prism --version
spectral lint openapi.yaml --ruleset campus-spectral.yaml
prism mock openapi.yaml --port 4010

```

---

## 6. Lệnh nộp bài lên GitHub Classroom

```bash
git status
git add openapi.yaml negotiation-log.md VERSIONING.md docs evidence/buoi-02
git commit -m "submit: lab02 openapi contract evidence"
git push origin main

```

---

## 7. Các tệp tin tuyệt đối không commit

Tuân thủ quy định chặn của GitHub Actions, dự án loại bỏ hoàn toàn các tệp tin sau khỏi lịch sử commit:

* Các file tài liệu văn phòng dạng binary (`*.doc`, `*.docx`, `*.ppt`, `*.pptx`).
* Các file cấu hình bảo mật môi trường (`.env`).
* Thư mục cài đặt thư viện (`node_modules/`).

---

## 8. Tinh thần của Lab 02

> Nhóm A4 (AI Vision) cam kết: **Không nộp “API em nghĩ là đúng”, mà nộp hợp đồng API đã được đàm phán kỹ lưỡng với nhóm A6, kiểm tra cú pháp nghiêm ngặt và có bằng chứng chạy mock thực tế thành công.**

```

---

Bạn tạo file hoặc dán nội dung này đè vào file `README.md` hiện tại là chuẩn chỉnh luôn nha! Đúng form, đúng tên nhóm và đúng các đầu mục yêu cầu luôn.

```