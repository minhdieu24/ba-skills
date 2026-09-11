---
name: pm-task
description: Tạo / sửa / tra cứu MỘT task lẻ trên PM Tool của Kafi (kai-foundry.kafisc.vn) qua MCP `kafi-pm` — không đụng wbs.md. Dùng khi user nói "tạo task lẻ trên PM Tool", "log task cho tôi", "tạo ticket PM Tool", "sửa task 6.x", "đổi status/assignee task", "tìm task trên PM Tool", "list task của tôi". Title bắt buộc prefix layer (BE:/FE:/DB:...); description theo khung Actual/Expect. Assignee hỏi mỗi lần (để trống được). KHÔNG dùng để sync WBS (đó là pm-wbs-sync), không tạo sprint/release/bug.
---

# Quản lý task lẻ trên PM Tool (kafi-pm)

Điều phối các tool MCP `kafi-pm` để **tạo / sửa / tra cứu 1 task đơn lẻ** trên PM Tool — task ad-hoc theo yêu cầu, **không** bắt nguồn từ `wbs.md`. Việc theo kế hoạch WBS (nhiều UOW, có traceability) thuộc skill `pm-wbs-sync`, không phải skill này.

Không có code/driver — mọi thao tác gọi thẳng tool `kafi-pm`.

## Điều kiện chạy
- MCP `kafi-pm` đã kết nối. Kiểm tra rẻ bằng `list_users` (cũng dùng để resolve assignee). Lỗi 401/403 hoặc không có tool → báo user MCP `kafi-pm` chưa kết nối và dừng, không tự đoán.

## Hằng số
- **Assignee**: HỎI mỗi lần — nhập tên, resolve qua `list_users` (khớp lỏng; 2 người khớp hoặc không ai khớp → hỏi lại). Để trống được (không assign).
- **Link ticket**: `https://kai-foundry.kafisc.vn/list/?item=<uuid>`
- **Default status** `TODO`, **default priority** `MEDIUM`.

## Convention bắt buộc (áp cho CREATE và khi UPDATE title/description)

### Title — phải có prefix layer
Dạng `<LAYER>: <mô tả>`. Layer thường dùng: `BE:`, `FE:`, `DB:` (cho phép khác: `Full:`, `Config:`, `Infra:`...).
- Ví dụ: `BE: Điều chỉnh API tính hoa hồng`, `FE: Mapping UI màn hình duyệt`.
- User đưa title **chưa có prefix** → HỎI layer rồi mới ghép, không tự đoán.

### Description — khung Actual / Expect
Nhãn **in đậm**, mỗi nhãn 1 dòng, nội dung xuống dòng ngay dưới, cách nhau 1 dòng trống:
```
**Actual:**
<nội dung>

**Expect:**
<nội dung>
```
- User đưa sẵn Actual/Expect → điền vào đúng 2 mục.
- User chỉ đưa mô tả thô → dựng theo khung này; thiếu vế nào thì hỏi cho đủ, không bịa nội dung.
- **KHÔNG dùng dấu nháy kép thẳng `"` trong description** — PM Tool escape thành `&#34;` và render nguyên chuỗi (bug hiển thị). Trích dẫn dùng nháy cong `“ ”` hoặc nháy đơn `'`.

---

## Luồng 1 — CREATE task

1. **Thu thập input.** Bắt buộc: `title` (đã chuẩn prefix). Tùy chọn: description (khung Actual/Expect), assignee, status, priority, due date.
2. **Chọn parent (mỗi lần).**
   - Lấy danh sách item: `list_work_items`. Kết quả có thể lớn → lưu ra file rồi lọc bằng python, chỉ hiển thị tier PROGRAM/PROJECT/EPIC/FEATURE, sort theo `positionId`, in dạng cây cho user chọn `positionId`.
   - User đưa `positionId` → tra ra UUID, `get_work_item` (hoặc đối chiếu trong danh sách) để **echo lại title · tier** của parent, xác nhận đúng chỗ. Parent hợp lệ là item **nông hơn TASK** (EPIC/FEATURE/STORY đều nhận task con vì TASK sâu hơn).
3. **Xác nhận 1 dòng** với user: parent · title · assignee · status · priority. Chờ "ok".
4. **Tạo**: `create_work_item(parentId, tier="TASK", title, status?)`.
5. **Bổ sung field**: `update_work_item(id, description?, ownerId?, priority?, dueDate?)` — description theo khung; ownerId = người user chỉ định (HỎI assignee nếu chưa nói; resolve tên qua `list_users`, khớp lỏng; 2 người khớp hoặc không ai khớp → hỏi lại). User không muốn assign → để trống.
6. **Trả về** bảng gọn + link ticket.

## Luồng 2 — UPDATE task

1. **Xác định task**: user cho `positionId` hoặc tên → `search_work_items` (theo title/positionId). Nhiều kết quả → liệt kê cho user chọn.
2. `get_work_item(id)` để **echo title · tier · breadcrumb**, xác nhận đúng ticket.
3. **Sửa** field user yêu cầu qua `update_work_item`.
   - Đổi title/description → vẫn theo convention prefix + khung Actual/Expect.
   - Đổi assignee → resolve qua `list_users`.
   - **Status quirk**: PM Tool chặn nhảy thẳng sang `DONE`. Đi `IN_REVIEW` → `DONE` (2 lệnh). Báo cho user biết đã đi 2 bước.
4. Xác nhận 1 dòng trước khi ghi. Trả về kết quả sau khi ghi.

## Luồng 3 — LIST / SEARCH task

- Theo title/positionId: `search_work_items(query)`.
- Theo owner/status: `list_work_items(ownerId?, status?)` (kết quả lớn → lọc bằng python từ file).
- In bảng gọn: `positionId · title · status · owner`. Không ghi gì (read-only).

## Archive (chỉ khi user yêu cầu xóa)
PM Tool **không hard-delete**. Muốn "xóa" → `archive_work_item` (khôi phục bằng `restore_work_item`). **Luôn hỏi xác nhận rõ ràng** trước khi archive, echo title ticket sẽ archive.

## Ràng buộc chung
- **Chỉ task lẻ.** Không tạo/sửa hàng loạt theo WBS (dùng `pm-wbs-sync`), không đụng `wbs.md`.
- **Không có tool upload ảnh/attachment.** Muốn nhúng ảnh trong description phải có URL công khai (`![](https://...)`); ảnh dán trong chat không host được — báo user tự attach trên web hoặc đưa URL.
- **Trước mọi lệnh ghi** (create/update/archive): xác nhận 1 dòng rồi mới chạy.
- Ngoài phạm vi (cần thì mở rộng sau): sprint, release, bug report, dependency.

## Tool kafi-pm dùng trong skill
`list_users` · `list_work_items` · `search_work_items` · `get_work_item` · `create_work_item` · `update_work_item` · `archive_work_item` / `restore_work_item`.
