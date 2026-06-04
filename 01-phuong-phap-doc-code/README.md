# 01 · Phương Pháp Đọc Hiểu Code — Theo Ngôn Ngữ

> 5 phase đọc code áp dụng cho mọi ngôn ngữ.  
> Chọn ngôn ngữ bạn đang làm việc để xem ví dụ cụ thể.

---

## 5 Phase — Áp dụng cho mọi stack

```
Phase 1 → Entry Points & Cấu hình      (hệ thống khởi động thế nào)
Phase 2 → Kiến trúc tổng thể           (cấu trúc thư mục, DI, pattern)
Phase 3 → Data Flow (FE → API → DB)    (luồng dữ liệu đi qua đâu)
Phase 4 → Feature Trace (end-to-end)   (trace 1 tính năng hoàn chỉnh)
Phase 5 → Deep Dive (biến, hàm)        (tìm nguồn gốc biến, call chain)
```

---

## Bắt đầu từ đây

> 👉 **Đọc file chung trước** — áp dụng cho mọi ngôn ngữ, sau đó chọn file ngôn ngữ cụ thể.

| File | Nội dung |
|------|----------|
| **[00-nguyen-tac-chung.md](./00-nguyen-tac-chung.md)** | ⭐ Nguyên tắc chung — ngôn ngữ nào cũng áp dụng được |

---

## Chọn ngôn ngữ / platform

| # | Ngôn ngữ | Stack | File |
|---|----------|-------|------|
| 1 | **C# .NET Core** | Blazor Server + ASP.NET Core API | [01a-csharp-dotnet.md](./01a-csharp-dotnet.md) |
| 2 | **Angular** | TypeScript + Angular + RxJS | [01b-angular.md](./01b-angular.md) |
| 3 | **Go** | Go + Gin/Echo/net/http | [01c-golang.md](./01c-golang.md) |
| 4 | **Java** | Spring Boot + Maven/Gradle | [01d-java-spring.md](./01d-java-spring.md) |
| 5 | **Kotlin Android** | Android + Jetpack + MVVM | [01e-kotlin-android.md](./01e-kotlin-android.md) |
| 6 | **Swift iOS** | UIKit / SwiftUI + Combine | [01f-swift-ios.md](./01f-swift-ios.md) |
| 7 | **Objective-C iOS** | UIKit + MVC legacy | [01g-objc-ios.md](./01g-objc-ios.md) |

---

## Nguyên tắc chung — dùng cho mọi ngôn ngữ

### Thứ tự đọc lý tưởng khi vào codebase mới

```
Ngày 1 (2–3 giờ):
  ├── Tìm file entry point (main / Program.cs / AppDelegate / Application.java)
  ├── Đọc file config (appsettings / application.yml / Info.plist / build.gradle)
  ├── Đọc file DI / wiring (nơi các dependency được kết nối)
  └── Đọc schema database (migration / entity / model)

Ngày 2–3:
  └── Chọn 3 tính năng nhỏ → trace end-to-end từng cái

Ngày 4+:
  └── Deep dive module quan trọng nhất
```

### Câu hỏi cần trả lời sau ngày đầu

- Hệ thống có mấy layer / module / service?
- DI Container ở đâu? Ai wire dependency cho ai?
- DB schema trông như thế nào? Bảng chính là gì?
- Có external service nào không (email, payment, push notification)?

### Khi thấy biến / hàm không hiểu — quy trình chung

```
1. Go to Definition (F12 / Cmd+Click) → xem khai báo
2. Find All References → xem ai đang dùng / gán
3. Đọc test của hàm đó → test là spec ngắn gọn nhất
4. git log -p -- <filename> → tại sao code này tồn tại
```

---

[← Quay lại README chính](../README.md)
