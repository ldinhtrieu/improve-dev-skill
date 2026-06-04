# 📚 Hướng Dẫn Đọc Hiểu Code Hệ Thống Lớn
> Stack: C# · .NET Core 8 · Blazor Server · ASP.NET Core API · EF Core

---

## Mục lục

| # | Tài liệu | Nội dung |
|---|----------|----------|
| 1 | [Phương pháp đọc code](./01-phuong-phap-doc-code.md) | 5 phase đọc hiểu hệ thống từ tổng quan xuống chi tiết |
| 2 | [Kiến trúc C# .NET + Blazor](./02-kien-truc-dotnet-blazor.md) | Cấu trúc project, Program.cs, DI, EF Core |
| 3 | [Hướng dẫn Debug](./03-huong-dan-debug.md) | Kỹ thuật debug đúng, trace data, Blazor Server specifics |
| 4 | [Vẽ sơ đồ khi đọc code](./04-ve-so-do.md) | 3 loại sơ đồ, cách xử lý if/else, template thực tế |

---

## Nguyên tắc xuyên suốt

- **Đừng đọc tuần tự từ trên xuống** — nhảy theo luồng dữ liệu và dependency
- **Dùng debugger nhiều hơn đọc code** — chạy được → breakpoint → bước từng dòng
- **Vẽ sơ đồ ngay khi đọc** — chỉ vẽ đủ để não khỏi nhớ, không vẽ để đẹp
- **Đọc test trước implementation** — test nói rõ hàm làm gì mà không cần đoán mò

---

> 💡 Mỗi file có thể đọc độc lập. Nên bắt đầu từ file 01 nếu mới tiếp cận codebase.
