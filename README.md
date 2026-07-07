Dưới đây là bản cập nhật hoàn chỉnh cho file `README.md` (hoặc `SETUP.md`), bao gồm cả luồng làm việc chuẩn với Git Submodule và mục **Xử lý sự cố (Troubleshooting)** mà bạn vừa gặp phải. Bạn có thể copy toàn bộ nội dung này để làm tài liệu chuẩn hóa cho team.

# Hướng Dẫn Thiết Lập Filament AI Skill (Git Submodule) Cho Continue.dev

Tài liệu này hướng dẫn cách đồng bộ bộ quy tắc AI (AI Skill) cho Filament v4 vào dự án thông qua Git Submodule. Phương pháp này giúp toàn bộ team phát triển có chung một "nguồn tri thức" chuẩn xác về kiến trúc, bảo mật và hiệu năng khi sử dụng AI (Continue.dev).

---

## 🚀 1. Khởi tạo Submodule vào dự án (Dành cho người setup dự án)

Tại thư mục gốc của dự án Laravel, chạy lệnh sau để kéo bộ Skill về làm Submodule:

```bash
git submodule add git@github.com:vinhhx/filament-v4-skill.git .continue/prompts
```

## ⚙️ 2. Cấu hình Continue.dev cấp dự án (Workspace Level)

Để mọi thành viên trong team gõ `@Filamentv4` là gọi được bộ rule, chúng ta cấu hình trực tiếp vào dự án.

1. Đảm bảo đã có thư mục `.continue` ở root dự án.
2. Tạo/Mở file `.continue/config.ts` và thêm cấu hình sau:

```typescript
export function modifyConfig(config: Config): Config {
  if (!config.contextProviders) {
    config.contextProviders = [];
  }

  // Khai báo Provider trỏ vào thư mục submodule
  config.contextProviders.push({
    name: "Filamentv4",
    title: "Filamentv4",
    description: "Bộ rules kiến trúc Filament v4 chuẩn của Team",
    type: "folder",
  });

  return config;
}
```

_Lưu ý: Lần đầu tiên gõ `@Filamentv4` trong chat, IDE sẽ hiển thị hộp thoại chọn thư mục. Các thành viên chỉ cần trỏ vào thư mục `.continue/prompts` của dự án._

---

## 🔄 3. Quy trình làm việc của Team (Git Protocol)

**A. Khi Clone dự án mới về máy:**
Bắt buộc sử dụng cờ `--recurse-submodules` để Git tải luôn cả mã nguồn dự án lẫn bộ rule AI.

```bash
git clone --recurse-submodules <link_repo_du_an_cua_ban>

```

**B. Khi cần cập nhật Rule AI mới nhất:**
Nếu có ai đó cập nhật file `SKILL.md` trên repo gốc, các thành viên khác chỉ cần chạy lệnh sau để kéo bản cập nhật về dự án hiện tại:

```bash
git submodule update --remote

```

---

## 🛠 4. Xử lý sự cố thường gặp (Troubleshooting)

### Lỗi: `fatal: '.continue/prompts' already exists and is not a valid git repo`

**Nguyên nhân:** Git Submodule yêu cầu thư mục đích (`.continue/prompts`) phải chưa được tạo sẵn trên ổ cứng hoặc chưa bị Git của dự án chính index.

**Cách khắc phục (Thực hiện lần lượt 3 bước):**

1. **Xóa thư mục đang cản đường:** (Hãy backup nếu bên trong đang chứa file do bạn tự tạo)

```bash
rm -rf .continue/prompts

```

2. **Xóa cache của Git:** (Xóa dấu vết thư mục trong Git index)

```bash
git rm -r --cached .continue/prompts

```

_(Nếu terminal báo lỗi `pathspec did not match any files`, bạn cứ bỏ qua vì điều đó chứng tỏ Git chưa track nó)._ 3. **Chạy lại lệnh thêm Submodule:**

```bash
git submodule add git@github.com:vinhhx/filament-v4-skill.git .continue/prompts

```
