# Quy tắc & Lưu ý cho File Hướng Dẫn Mastra

## 1. Tuân thủ cấu trúc thư mục chuẩn
- Giữ nguyên các thư mục gốc như `agents/`, `tools/`, và file `index.ts` trong `src/mastra/`.
- Khi mở rộng, chỉ thêm vào các thư mục: `core/`, `channels/`, `llm/` theo cấu trúc đã định.
- Mỗi channel (telegram, whatsapp, web, line) phải nằm trong `src/mastra/channels/` và chỉ chứa code liên quan đến channel đó.

**Cấu trúc con chuẩn mỗi kênh (`src/mastra/channels/<channel>/`):**
- `config/`: Cấu hình riêng cho kênh (token, endpoint, mapping hằng số).
- `tests/`: Kiểm thử adapter và xử lý đặc thù kênh.
Chỉ chứa adapter & xử lý đặc thù kênh; không chứa logic nghiệp vụ chung.

## 2. Hỗ trợ cho code trong file `mastra/p2-1-core-implementation.md`
- Các hướng dẫn phải đảm bảo tương thích với kiến trúc đa kênh, tách biệt (decoupled multi-channel).
- Logic nghiệp vụ phải tập trung ở `core/`, không phân tán sang các channel.
- Các channel chỉ đóng vai trò adapter, không chứa logic nghiệp vụ chung.

## 3. Thiết kế dựa trên Interface (TypeScript)
- Sử dụng interface thay vì kế thừa (no base class).
- Đảm bảo các thành phần được tách rời, dễ kiểm thử và bảo trì.

## 4. Định nghĩa chuẩn cho message và response
- Sử dụng các interface chuẩn cho message, user, context, response như đã định nghĩa trong `core/models/message.ts`.
- Đảm bảo mọi channel đều tuân thủ format này khi xử lý và truyền nhận dữ liệu.

## 5. Không làm thay đổi hoặc xung đột với phần Mastra gốc
- Chỉ mở rộng, không thay thế hoặc chỉnh sửa các thành phần gốc của Mastra.
