# Hướng Dẫn Thiết Lập Filament AI Skill Tập Trung Cho Continue.dev

Tài liệu này hướng dẫn cách thiết lập một "Kho tri thức AI" (AI Skill Repository) tập trung cho Filament. Giải pháp này giúp bạn cài đặt tự động và yêu cầu AI sinh code chuẩn kiến trúc cho các dự án Laravel + Filament thông qua Continue.dev mà không cần phải copy-paste các file rules (như `SKILL.md`, `CLAUDE.md`) vào từng dự án riêng lẻ.

## 🚀 Lợi ích

- **DRY (Don't Repeat Yourself):** Quản lý rules AI ở một nơi duy nhất.
- **Cập nhật đồng bộ:** Sửa rule một lần ở kho trung tâm, mọi dự án cũ và mới đều nhận được bối cảnh (context) mới nhất.
- **Tự động hóa toàn diện:** Khởi tạo dự án mới chuẩn kiến trúc chỉ với 1 slash command.

---

## 🛠 Các bước thiết lập

### Bước 1: Lưu trữ Repository Skill ở vị trí cố định

Lưu toàn bộ repository chứa các file rules của Filament (bao gồm `SKILL.md`, `CLAUDE.md`, và thư mục `references/`) tại một đường dẫn tuyệt đối (Absolute Path) trên máy tính của bạn.

_Ví dụ đường dẫn:_

- Trên Windows: `/frameworks/filament-skill`
- Trên Mac/Linux: `~/frameworks/filament-skill`

### Bước 2: Cấu hình lệnh tùy chỉnh (Custom Command) toàn cục

Thiết lập lệnh khởi tạo dự động trong Continue.dev để AI có thể đọc file rule từ kho trung tâm và tự động chạy script cài đặt.

1. Mở giao diện **Continue** trong IDE (VS Code / JetBrains).
2. Nhấn vào biểu tượng ⚙️ **(Settings)** ở góc dưới cùng bên phải để mở file `config.json` toàn cục.
3. Thêm cấu hình sau vào mảng `"customCommands"` _(lưu ý: hãy thay đổi đường dẫn `/frameworks/filament-skill/SKILL.md` cho khớp với đường dẫn thực tế trên máy tính của bạn)_:

```json
"customCommands": [
  {
    "name": "install_filament",
    "description": "Khởi tạo dự án Laravel và Filament v4 sạch vào thư mục hiện tại dựa trên bộ quy tắc hệ thống tập trung.",
    "prompt": "You are an expert AI Architect. First, read and strictly adhere to the infrastructure and architecture standards defined in this master skill file: 'C:/ai-frameworks/filament-skill/SKILL.md' (and its referenced documentation under the same directory if needed).\n\nNow, perform a fresh installation of Laravel and Filament v4 in the current empty directory by executing these terminal commands sequentially:\n1. `composer create-project laravel/laravel . --force`\n2. Configure `.env` to use SQLite (create `database/database.sqlite` if it doesn't exist).\n3. `php artisan migrate`\n4. `composer require filament/filament:\"^4.0\" -W`\n5. `php artisan filament:install --panels`\n6. `php artisan make:filament-user` (Auto-fill with Name: Admin, Email: admin@admin.com, Password: password).\n\nCONFIRMATION: Run these commands directly using your terminal execution capabilities. Do not ask for user input between steps unless a fatal error occurs."
  }
]
```

### Bước 3: Khởi tạo dự án mới

Quy trình làm việc khi bạn bắt đầu phát triển một dự án mới:

1. Tạo một thư mục trống hoàn toàn và mở thư mục đó bằng IDE.
2. Mở sidebar chat của Continue.dev.
3. Gõ lệnh `/install_filament` và nhấn **Enter**.
4. AI Agent sẽ tự động đọc kiến trúc từ kho trung tâm, mở terminal và thực thi tuần tự các lệnh cài đặt Laravel, cấu hình database SQLite, tải Filament v4 và tạo tài khoản Admin.

---

## 🔄 Quy trình cập nhật (Maintenance Workflow)

Vì cấu hình trong Continue.dev trỏ trực tiếp bằng **đường dẫn tuyệt đối** đến kho trung tâm, hệ thống này cực kỳ dễ bảo trì:

1. Khi có bản cập nhật mới thay đổi coding convention, bạn chỉ cần mở thư mục `/frameworks/filament-skill` và sửa đổi các file Markdown bên trong đó.
2. **Không cần thao tác lại ở bất kỳ dự án nào.** Lần gõ prompt tiếp theo hoặc lần chạy lệnh cài đặt mới, Continue sẽ tự động đọc và tuân thủ theo nội dung file cập nhật mới nhất.
