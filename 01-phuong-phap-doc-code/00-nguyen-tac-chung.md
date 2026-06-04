# 00 · Phương Pháp Đọc Hiểu Code — Nguyên Tắc Chung

> Áp dụng cho mọi ngôn ngữ, mọi stack, mọi quy mô hệ thống.  
> Đọc file này trước khi đọc file ngôn ngữ cụ thể.

---

## Tại sao đọc code khó?

Viết code là tuyến tính — từ trên xuống dưới, từ yêu cầu đến triển khai.  
Đọc code là phi tuyến — hệ thống lớn không viết để đọc như sách.

Ba cái bẫy hay gặp:

```
Bẫy 1: Đọc từ đầu file xuống cuối → bị ngợp, không biết cái gì quan trọng
Bẫy 2: Nhảy vào chi tiết quá sớm → mất phương hướng, không biết đang ở đâu
Bẫy 3: Không vẽ, không ghi chú   → đọc xong 1 ngày là quên gần hết
```

---

## Nguyên tắc nền tảng

### 1. Đọc theo luồng dữ liệu, không đọc theo file

```
❌ Sai: Mở file đầu tiên trong folder → đọc từ trên xuống → mở file tiếp theo
✅ Đúng: Tìm điểm bắt đầu của 1 hành động → theo dữ liệu đi qua các tầng
```

Dữ liệu luôn đi theo một hướng có thể dự đoán:

```
Input (user action / API request / event)
    ↓
Validation / parsing
    ↓
Business logic
    ↓
Data access (DB / cache / external API)
    ↓
Output (response / UI update / side effect)
```

Khi biết luồng này, bạn biết cần nhảy đến đâu tiếp theo.

---

### 2. Luôn bắt đầu từ Entry Point

Mọi hệ thống đều có 1 điểm khởi động. Tìm nó trước — đây là "cổng vào" của toàn bộ logic.

| Loại hệ thống | Entry Point |
|---------------|-------------|
| Web Backend | `main()`, `Program.cs`, `Application.java`, `app.go` |
| Web Frontend | `main.ts`, `index.js`, `App.jsx` |
| Mobile iOS | `AppDelegate`, `@main App` struct |
| Mobile Android | `AndroidManifest.xml` → `MainActivity` |
| CLI Tool | `main()` function |
| Serverless | Handler function (`handler(event, context)`) |

Entry point thường làm 3 việc — đọc để biết ngay:
1. Load config (DB, external services, API keys)
2. Wire dependencies (ai phụ thuộc vào ai)
3. Start listening (HTTP port, message queue, event loop)

---

### 3. Phân biệt "Bản đồ" và "Chi tiết"

Não người không thể hiểu chi tiết mà không có bản đồ trước.

```
Bản đồ (đọc trước):
  "Hệ thống có module A, B, C. A gọi B. B gọi C."
  → Cần 30 phút, đọc tên file/folder + config

Chi tiết (đọc sau):
  "Hàm processOrder() trong module A làm gì từng bước"
  → Cần khi đang giải quyết vấn đề cụ thể
```

Sai lầm phổ biến: nhảy vào chi tiết khi chưa có bản đồ → bị lạc.

---

## 5 Phase đọc code

### Phase 1 — Entry Points & Cấu hình
**Mục tiêu:** Hiểu hệ thống có gì, kết nối gì.

Tìm và đọc:
- File khởi động chính
- File config / environment
- File khai báo dependency (package.json, pom.xml, go.mod, Podfile...)

**Câu hỏi cần trả lời:**
- Hệ thống có mấy tầng / service?
- Dùng DB gì? Cache gì? Queue gì?
- Có external service nào (email, payment, SMS, push notification)?

---

### Phase 2 — Kiến trúc tổng thể
**Mục tiêu:** Vẽ mental map — biết mình đang đứng ở đâu.

**Cấu trúc thư mục luôn tiết lộ architectural pattern:**

```
controllers/ + services/ + repositories/  → Layered Architecture (MVC)
features/ hoặc modules/                   → Feature-based / Vertical Slice
domain/ + application/ + infrastructure/  → Clean Architecture / DDD
handlers/ + middleware/                   → Pipeline pattern (Go, Node)
ViewModels/ + Views/ + Models/            → MVVM (Mobile, WPF)
```

**Tìm "nơi wire dependency"** — đây là bản đồ chi tiết nhất của hệ thống:

| Ngôn ngữ / Framework | Nơi wire dependency |
|-----------------------|---------------------|
| .NET Core | `Program.cs` — `builder.Services.AddXxx()` |
| Spring Boot | `@Configuration` classes + `@ComponentScan` |
| Angular | `AppModule` providers hoặc `bootstrapApplication()` |
| Go | `main()` — manual wiring |
| Android (Hilt) | `@Module` classes |
| iOS | `AppDelegate` / DI Container class |

---

### Phase 3 — Data Flow (Input → Output)
**Mục tiêu:** Hiểu luồng dữ liệu đi từ đâu đến đâu.

**Template luồng tổng quát:**

```
[Trigger]
    ↓ dữ liệu đầu vào
[Validation / Parsing]           ← ai validate? validate gì?
    ↓
[Business Logic]                 ← rule nào được áp dụng?
    ↓ side effects (nếu có)
[External Calls]                 ← gọi DB? API khác? Queue?
    ↓
[Mapping / Transform]            ← convert sang format output
    ↓
[Output / Response]              ← trả về gì? update UI gì?
```

**Cách trace nhanh — tìm endpoint/action trước:**

```
Bước 1: Xác định URL / màn hình / event cần hiểu
Bước 2: Tìm nơi xử lý nó (route config / event listener)
Bước 3: Follow the code — nhảy vào hàm được gọi
Bước 4: Ở mỗi bước, hỏi "dữ liệu này đến từ đâu, đi đến đâu?"
```

---

### Phase 4 — Feature Trace (End-to-End)
**Mục tiêu:** Hiểu sâu 1 tính năng hoàn chỉnh.

**Nguyên tắc chọn tính năng:**
- ✅ Nhỏ, cụ thể: "user reset mật khẩu", "thêm item vào giỏ"
- ✅ Có đầy đủ cả read lẫn write (thú vị hơn chỉ đọc)
- ❌ Quá rộng: "toàn bộ module thanh toán"

**Checklist trace 1 tính năng:**

```
□ Tìm điểm bắt đầu (UI action / API endpoint / event)
□ Trace vào layer xử lý đầu tiên
□ Ghi lại từng bước (xem 04-ve-so-do.md)
□ Chú ý side effects (email gửi đi, cache invalidate, event publish)
□ Tìm schema DB của bảng liên quan
□ Đọc test nếu có — test là spec ngắn gọn nhất
```

> 💡 Sau 3–5 tính năng nhỏ, bức tranh tổng thể sẽ hiện ra tự nhiên.

---

### Phase 5 — Deep Dive (Biến, Hàm, Call Chain)
**Mục tiêu:** Hiểu "cái này lấy từ đâu" — câu hỏi hay gặp nhất.

**Quy trình tìm nguồn gốc của bất kỳ thứ gì:**

```
1. Go to Definition       → xem khai báo, kiểu dữ liệu
2. Find All References    → ai đang dùng / ai đang gán
3. Đọc test liên quan     → "hàm này nhận gì, trả về gì"
4. git log / git blame    → tại sao code này tồn tại, ai viết, khi nào
```

**Khi đọc một hàm lạ — đọc theo thứ tự:**

```
Bước 1: Tên hàm + parameters + return type
        → 70% hiểu được hàm làm gì mà không cần đọc body

Bước 2: Đọc dòng đầu và dòng cuối của hàm
        → biết input nhận gì, output trả gì

Bước 3: Tìm điều kiện đặc biệt (if lỗi, edge case)
        → hiểu hàm fail như thế nào

Bước 4: Đọc phần giữa
        → hiểu logic chính
```

---

## Công cụ đọc code hiệu quả

### IDE / Editor — Shortcuts cần biết

Dù dùng ngôn ngữ nào, 4 thao tác này là thiết yếu:

| Thao tác | VS Code | IntelliJ / Rider | Xcode |
|----------|---------|-----------------|-------|
| Go to Definition | `F12` | `Ctrl+B` | `Cmd+Click` |
| Find All References | `Shift+F12` | `Alt+F7` | `Ctrl+1` → Call Hierarchy |
| Search toàn project | `Ctrl+Shift+F` | `Ctrl+Shift+F` | `Shift+Cmd+F` |
| Tìm file nhanh | `Ctrl+P` | `Shift+Shift` | `Shift+Cmd+O` |

### Git — đọc lịch sử để hiểu ý định

```bash
# Xem ai viết dòng này và tại sao
git blame <filename>

# Xem lịch sử thay đổi của 1 file
git log --oneline -- <filename>

# Xem nội dung thay đổi của 1 commit
git show <commit-hash>

# Tìm commit nào thêm vào 1 đoạn code cụ thể
git log -S "tên hàm hoặc đoạn code" --oneline

# Xem lịch sử với diff
git log -p -- <filename>
```

> 💡 Commit message tốt giải thích **tại sao** thay đổi, không phải **cái gì** thay đổi. Đọc message để hiểu business context.

### Debugger — nhanh hơn đọc code 10 lần

```
Nguyên tắc dùng debugger:
  ├── Đặt breakpoint tại điểm BẮT ĐẦU của hành động (không phải kết quả)
  ├── Dùng Step Over (F10) mặc định — KHÔNG nhảy vào mọi thứ
  ├── Chỉ Step Into (F11) vào code của mình, không vào framework
  ├── Step Out (Shift+F11) ngay khi lỡ nhảy vào chỗ không cần
  └── Đọc Call Stack khi bị lost — hiện ngay "đến đây bằng con đường nào"
```

### Tests — tài liệu luôn up-to-date

```
Unit test:        hiểu 1 hàm làm gì, nhận gì, trả gì, fail khi nào
Integration test: hiểu 2-3 layer phối hợp thế nào
E2E test:         hiểu 1 user flow hoàn chỉnh từ đầu đến cuối

Cách đọc test nhanh:
  1. Đọc tên test case → biết scenario
  2. Đọc phần "Arrange" (setup) → biết điều kiện đầu vào
  3. Đọc phần "Assert" (expect) → biết kết quả mong đợi
  4. Bỏ qua phần "Act" nếu tên test đã rõ
```

---

## Kỹ thuật vẽ sơ đồ khi đọc

> Không vẽ = quên sau 1 ngày. Vẽ quá nhiều = sơ đồ không dùng được.

### 3 loại sơ đồ — mỗi loại 1 mục đích

**Loại 1 — Map (bản đồ hệ thống):**
Vẽ 1 lần lúc bắt đầu. Chỉ vẽ boxes (module/service) và arrows (gọi nhau).
Tối đa 10 box, 1 tờ A4. Không vẽ tên hàm, field, màu sắc.

**Loại 2 — Flow (luồng tính năng):**
Viết text theo thứ tự, không cần hình học. Tối đa 8 bước.
Nhánh lỗi ghi `✗` bên phải — không vẽ vào luồng chính.

```
1. [Layer].[Hàm](params)         ✗ [lỗi gì] → [xử lý gì]
        ↓ [data truyền đi]
2. [Layer].[Hàm](params)
```

**Loại 3 — Cheat sheet:**
Ghi liên tục dạng text. Format: `"Để làm X → vào file Y"`, `"Biến Z lấy từ W"`.
Giữ mãi, cập nhật liên tục.

### Test trước khi vẽ thêm

> *"Nếu không có thông tin này, mình có bị nhầm hoặc quên không?"*  
> Nếu không → đừng vẽ.

---

## Thứ tự đọc lý tưởng — Áp dụng mọi stack

```
Ngày 1 — Bản đồ (2–3 giờ)
  ├── Entry point                    → hiểu khởi động thế nào
  ├── Config / environment files     → DB, external services
  ├── Dependency wiring              → ai phụ thuộc vào ai
  └── Database schema                → data model trông như thế nào

Ngày 2–3 — Feature trace
  └── 3 tính năng nhỏ, trace end-to-end từng cái
      (chọn 1 read, 1 write, 1 có side effect)

Ngày 4+ — Deep dive
  └── Module quan trọng nhất với business domain
```

---

## Dấu hiệu đang đọc sai cách

| Dấu hiệu | Vấn đề | Giải pháp |
|----------|--------|-----------|
| Đọc 2 giờ mà không biết hệ thống làm gì | Đọc chi tiết trước bản đồ | Quay lại Phase 1–2 |
| Mở 20+ file cùng lúc | Không có định hướng | Đóng hết, chọn 1 tính năng để trace |
| Đọc xong không nhớ gì | Không ghi chú | Mở notepad, vẽ flow ngay khi đọc |
| Debug bị lạc sâu trong framework | Dùng F11 quá nhiều | Dùng F10 mặc định, Shift+F11 để thoát |
| Thấy biến có data mà không biết từ đâu | Bắt đầu từ kết quả | Đặt breakpoint tại lifecycle init |

---

## Câu hỏi định hướng khi bị stuck

Khi không biết đọc tiếp từ đâu, hỏi 1 trong 5 câu này:

```
1. "Data này đến từ đâu?"
   → Go to Definition → Find References → trace ngược về setter

2. "Hàm này được gọi từ đâu?"
   → Find All References → Call Hierarchy

3. "Đây là tính năng gì ngoài đời thực?"
   → Đặt tên business cho đoạn code → hiểu ngay context

4. "Test case nào cover đoạn code này?"
   → Tìm test file tương ứng → đọc test để hiểu intent

5. "Ai viết dòng này và tại sao?"
   → git blame + git log → đọc commit message
```

---

## Checklist — Ngày đầu tiên với codebase mới

```
□ Đọc README.md (nếu có) — thường có hướng dẫn setup và overview
□ Tìm và đọc entry point
□ Đọc file config / environment
□ Tìm nơi wire dependency → biết toàn bộ components
□ Đọc schema database (migration files, entity/model classes)
□ Vẽ Map sơ bộ trên notepad (tối đa 15 phút)
□ Chọn 1 tính năng nhỏ → trace end-to-end
□ Bắt đầu Cheat Sheet: ghi những gì "à hóa ra là vậy"
□ Đặt câu hỏi cho người biết codebase (nếu có)
  → Hỏi "module nào quan trọng nhất?" thay vì "giải thích toàn bộ cho tôi"
```

---

## Tham khảo theo ngôn ngữ

| Ngôn ngữ | File chi tiết |
|----------|---------------|
| C# .NET Core + Blazor | [01a-csharp-dotnet.md](./01a-csharp-dotnet.md) |
| Angular (TypeScript) | [01b-angular.md](./01b-angular.md) |
| Go | [01c-golang.md](./01c-golang.md) |
| Java Spring Boot | [01d-java-spring.md](./01d-java-spring.md) |
| Kotlin Android | [01e-kotlin-android.md](./01e-kotlin-android.md) |
| Swift iOS | [01f-swift-ios.md](./01f-swift-ios.md) |
| Objective-C iOS | [01g-objc-ios.md](./01g-objc-ios.md) |

---

[← Quay lại README chính](../README.md)
