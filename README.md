# ba-skills

Bộ skill cá nhân cho công việc BA tại Kafi, đóng gói dưới dạng Claude Code plugin.

## Skill hiện có

| Skill | Chức năng |
|---|---|
| `pm-task` | Tạo / sửa / tra cứu **1 task lẻ** trên PM Tool (kai-foundry.kafisc.vn) qua MCP `kafi-pm`. |

## Yêu cầu

- Đã cấu hình MCP server **`kafi-pm`** trong Claude Code (kết nối tới PM Tool của Kafi). Không có server này thì skill không chạy được.

## Cài đặt

```bash
claude plugin marketplace add https://github.com/minhdieu24/ba-skills.git
claude plugin install ba-skills
```

Cập nhật:

```bash
claude plugin update ba-skills
```

Skill nạp ở **session mới** — mở phiên Claude Code mới sau khi cài. Gọi bằng `/pm-task` hoặc nói tự nhiên ("tạo task lẻ trên PM Tool", "log task cho tôi"...).

## Lưu ý

`pm-task` **hỏi assignee mỗi lần** khi tạo/sửa task (nhập tên, resolve qua roster; để trống nếu không gán) — không hardcode người nào.
