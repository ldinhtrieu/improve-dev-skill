# 📚 Hướng Dẫn Đọc Hiểu Code Hệ Thống Lớn

> Dành cho developer muốn nhanh chóng làm chủ một codebase mới —  
> từ hiểu kiến trúc tổng thể đến debug chi tiết và ghi chú đúng cách.

---

## 🗺️ Sơ đồ tài liệu

```
📁 Toàn bộ tài liệu
│
├── 📄 00 · Nguyên tắc chung          ← Bắt đầu từ đây nếu mới đọc lần đầu
│
├── 📁 01 · Phương pháp đọc code      ← 5 Phase + ví dụ theo từng ngôn ngữ
│   ├── 00 · Nguyên tắc chung (mọi ngôn ngữ)
│   ├── 01a · C# .NET Core + Blazor
│   ├── 01b · Angular (TypeScript)
│   ├── 01c · Go
│   ├── 01d · Java Spring Boot
│   ├── 01e · Kotlin Android
│   ├── 01f · Swift iOS
│   └── 01g · Objective-C iOS
│
├── 📄 02 · Kiến trúc C# .NET + Blazor  ← Chi tiết stack chính
├── 📄 03 · Hướng dẫn Debug             ← Kỹ thuật debug đúng cách
└── 📄 04 · Vẽ sơ đồ khi đọc code      ← Ghi chú không bị quên
```

---

## 📖 Mục lục đầy đủ

### Tài liệu nền tảng — đọc trước

| # | Tài liệu | Mô tả | Dành cho |
|---|----------|-------|----------|
| ⭐ | [**Nguyên tắc chung**](./01-phuong-phap-doc-code/00-nguyen-tac-chung.md) | 5 phase, data flow, công cụ, checklist ngày đầu | Mọi ngôn ngữ |
| 03 | [**Hướng dẫn Debug**](./03-huong-dan-debug.md) | F10/F11/Shift+F11, trace data, Blazor Server | C# + Blazor |
| 04 | [**Vẽ sơ đồ khi đọc code**](./04-ve-so-do.md) | Map / Flow / Cheat sheet, xử lý if/else | Mọi ngôn ngữ |

---

### Phương pháp đọc code — theo ngôn ngữ

| # | Ngôn ngữ / Platform | Stack | Tài liệu |
|---|---------------------|-------|----------|
| 1 | **C# .NET Core** | Blazor Server · ASP.NET Core API · EF Core | [01a-csharp-dotnet.md](./01-phuong-phap-doc-code/01a-csharp-dotnet.md) |
| 2 | **Angular** | TypeScript · RxJS · NgRx | [01b-angular.md](./01-phuong-phap-doc-code/01b-angular.md) |
| 3 | **Go** | Gin / Echo · net/http · GORM | [01c-golang.md](./01-phuong-phap-doc-code/01c-golang.md) |
| 4 | **Java** | Spring Boot · JPA/Hibernate · Maven | [01d-java-spring.md](./01-phuong-phap-doc-code/01d-java-spring.md) |
| 5 | **Kotlin Android** | Jetpack · MVVM · Coroutines · Hilt | [01e-kotlin-android.md](./01-phuong-phap-doc-code/01e-kotlin-android.md) |
| 6 | **Swift iOS** | UIKit · SwiftUI · Combine · async/await | [01f-swift-ios.md](./01-phuong-phap-doc-code/01f-swift-ios.md) |
| 7 | **Objective-C iOS** | UIKit · MVC · AFNetworking | [01g-objc-ios.md](./01-phuong-phap-doc-code/01g-objc-ios.md) |

---

### Kiến trúc chi tiết — C# .NET Core + Blazor Server

| # | Tài liệu | Nội dung |
|---|----------|----------|
| 02 | [**Kiến trúc C# .NET + Blazor**](./02-kien-truc-dotnet-blazor.md) | Program.cs · DI · Blazor lifecycle · EF Core · AppDbContext · Migrations |

---

## 🚀 Bắt đầu từ đâu?

### Nếu bạn mới vào một codebase lần đầu

```
Bước 1 → Đọc 00-nguyen-tac-chung.md         (30 phút)
Bước 2 → Chọn file ngôn ngữ tương ứng       (30 phút)
Bước 3 → Đọc 04-ve-so-do.md                 (20 phút)
Bước 4 → Mở codebase, bắt đầu Phase 1       (thực hành)
```

### Nếu bạn đang làm việc với C# + Blazor Server

```
Bước 1 → 00-nguyen-tac-chung.md             (nền tảng)
Bước 2 → 01a-csharp-dotnet.md               (đọc code theo stack)
Bước 3 → 02-kien-truc-dotnet-blazor.md      (kiến trúc chi tiết)
Bước 4 → 03-huong-dan-debug.md              (debug đúng cách)
Bước 5 → 04-ve-so-do.md                     (ghi chú khi đọc)
```

### Nếu bạn chỉ muốn giải quyết vấn đề cụ thể

| Vấn đề | Đọc file |
|--------|----------|
| Không biết biến lấy từ đâu | [03 · Debug](./03-huong-dan-debug.md) — mục "Vấn đề 2" |
| Debug đi quá sâu, bị lạc | [03 · Debug](./03-huong-dan-debug.md) — mục "Vấn đề 1" |
| Vẽ flow có quá nhiều if/else | [04 · Sơ đồ](./04-ve-so-do.md) — mục "Kỹ thuật xử lý if/else" |
| Không biết bắt đầu đọc từ đâu | [00 · Nguyên tắc chung](./01-phuong-phap-doc-code/00-nguyen-tac-chung.md) — Phase 1 |
| Mới vào project .NET, đọc gì trước | [02 · Kiến trúc Blazor](./02-kien-truc-dotnet-blazor.md) — mục Program.cs |

---

## 💡 4 Nguyên tắc xuyên suốt

> Dù ngôn ngữ nào, 4 điều này luôn đúng.

**1. Đọc theo luồng dữ liệu, không đọc theo file**
Dữ liệu đi từ input → validation → business logic → data access → output.
H�y follow theo hướng đó thay vì mở file theo thứ tự alphabet.

**2. Bản đồ trước, chi tiết sau**
30 phút đọc entry point + config + DI wiring = biết 70% hệ thống.
Nhảy vào chi tiết khi chưa có bản đồ = bị lạc ngay.

**3. Debugger nhanh hơn đọc code 10 lần**
Chạy được → đặt breakpoint ở đầu hành động → F10 từng bước.
Không F11 vào code framework. Dùng Call Stack khi bị lost.

**4. Vẽ ngay, ghi ngay — hoặc quên ngay**
Không cần đẹp. Cần đọc lại trong 10 giây là hiểu.
Cheat sheet là tài sản quý nhất sau vài tuần đọc codebase.

---

## 📋 Checklist — Ngày đầu tiên với codebase mới

```
□ Đọc README của project (nếu có)
□ Tìm và đọc entry point (main / Program.cs / AppDelegate...)
□ Đọc file config / environment
□ Tìm nơi wire dependency → biết toàn bộ components
□ Đọc schema database
□ Vẽ Map sơ bộ trên notepad (tối đa 15 phút, tối đa 10 box)
□ Chọn 1 tính năng nhỏ → trace end-to-end
□ Bắt đầu Cheat Sheet: ghi những gì "à hóa ra là vậy"
```

---

*Tài liệu này được tổ chức để đọc theo nhu cầu — không cần đọc tuần tự từ đầu đến cuối.*
