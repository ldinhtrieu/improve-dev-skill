# 01 · Phương Pháp Đọc Hiểu Code Hệ Thống Lớn

> Áp dụng cho hệ thống có FE + API + Database, nhiều màn hình, nhiều tính năng.

---

## 5 Phase đọc code theo thứ tự

```
Phase 1 → Entry Points & Cấu hình
Phase 2 → Kiến trúc tổng thể
Phase 3 → Data Flow (FE → API → DB)
Phase 4 → Feature Trace (end-to-end)
Phase 5 → Deep Dive (biến, hàm, call chain)
```

---

## Phase 1 — Entry Points & Cấu hình

**Mục tiêu:** Hiểu hệ thống khởi động như thế nào, dùng gì.

Đọc theo thứ tự:

| File | Biết được gì |
|------|-------------|
| `Program.cs` | Toàn bộ services đăng ký, middleware pipeline |
| `appsettings.json` | Connection string, external services, config |
| `docker-compose.yml` | Có bao nhiêu service, port nào dùng gì |
| `.env` / `secrets.json` | API keys, credentials |

**Câu hỏi cần trả lời sau phase này:**
- Hệ thống có mấy project/service?
- Dùng DB gì? Cache gì? Queue gì?
- Có external service nào không (email, payment, SMS)?

---

## Phase 2 — Kiến trúc tổng thể

**Mục tiêu:** Vẽ mental map trước khi đọc từng dòng.

### Cấu trúc thư mục tiết lộ architectural pattern

```
/Controllers        → MVC / Controller-based API
/Handlers           → CQRS / Mediator pattern
/Services           → Service layer
/Repositories       → Repository pattern
/Domain             → Domain-Driven Design
/Features           → Feature Slices (Vertical Slice)
```

### Tìm Dependency Injection Container

Trong .NET Core — toàn bộ nằm trong `Program.cs`:

```csharp
// Tìm những dòng này để biết hệ thống có gì
builder.Services.AddDbContext<AppDbContext>(...);
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
builder.Services.AddHttpClient<IPaymentService, PaymentService>(...);
```

> 💡 **Tip:** Đọc hết `Program.cs` một lần — đây là bản đồ toàn bộ hệ thống.

---

## Phase 3 — Data Flow (FE → API → DB)

**Mục tiêu:** Hiểu luồng dữ liệu đi từ đâu đến đâu.

### Request lifecycle trong .NET Core

```
Browser / Blazor component
    ↓ HTTP request
Router → Controller / Minimal API endpoint
    ↓
Middleware (Auth, Validation, Logging)
    ↓
Service layer (business logic)
    ↓
Repository / DbContext
    ↓
SQL query (EF Core → Database)
    ↓
Response mapping (DTO → JSON)
```

### Cách trace: tìm endpoint trước

```bash
# Tìm route trong codebase
# Ví dụ cần hiểu API POST /api/orders

# Bước 1: Tìm controller
Ctrl+Shift+F → tìm "api/orders" hoặc "[Route("orders")]"

# Bước 2: Từ controller nhảy vào service
F12 trên tên method → Go to Definition

# Bước 3: Từ service xuống repository
F12 tiếp → trace xuống DB
```

---

## Phase 4 — Feature Trace (end-to-end)

**Mục tiêu:** Hiểu sâu 1 tính năng hoàn chỉnh thay vì đọc lan man.

### Cách chọn tính năng để trace

Chọn tính năng **nhỏ và cụ thể**:
- ✅ "User đặt lại mật khẩu"
- ✅ "Thêm sản phẩm vào giỏ hàng"
- ❌ "Toàn bộ module quản lý đơn hàng" (quá rộng)

### Checklist trace 1 tính năng

- [ ] Tìm UI component liên quan (file `.razor`)
- [ ] Tìm API call nó gọi (`HttpClient`, `@inject Service`)
- [ ] Trace toàn bộ server-side cho API đó
- [ ] Xem DB schema của bảng liên quan (`AppDbContext`, Migrations)
- [ ] Đọc unit test nếu có

> 💡 Sau 3–5 tính năng nhỏ, bức tranh tổng thể sẽ hiện ra tự nhiên.

---

## Phase 5 — Deep Dive (biến, hàm, call chain)

**Mục tiêu:** Hiểu "cái này lấy từ đâu" — câu hỏi hay gặp nhất.

### Khi thấy biến không rõ nguồn gốc

```
1. F12 (Go to Definition) → xem khai báo
2. Shift+F12 (Find All References) → xem được set ở đâu
3. Nếu từ DI: tìm nơi register trong Program.cs
4. Nếu từ DB: tìm query populate nó
```

### Khi thấy hàm không hiểu

```
1. Đọc signature (tên, params, return type) — thường đủ để đoán
2. Alt+F7 / Ctrl+K,R → Call Hierarchy (ai gọi nó, nó gọi ai)
3. Đọc test của hàm đó — test là spec ngắn gọn nhất
4. git log -p -- <filename> → xem lịch sử tại sao được thêm vào
```

### Nguồn gốc biến trong Blazor

| Thấy trong .razor | Nguồn gốc | Cách tìm |
|-------------------|-----------|----------|
| `@inject IXxxService Svc` | DI Container | Tìm trong `Program.cs` |
| `[Parameter]` | Component cha truyền vào | Tìm `<TênComponent` trong các .razor khác |
| `[CascadingParameter]` | `<CascadingValue>` wrapper | Tìm trong layout hoặc `App.razor` |
| `NavigationManager`, `IJSRuntime` | Built-in Blazor | Luôn available |

---

## Thứ tự đọc lý tưởng cho codebase mới

```
Ngày 1:
  ├── Program.cs           (30 phút — hiểu toàn bộ services)
  ├── appsettings.json     (10 phút — config, external services)
  ├── AppDbContext.cs       (20 phút — schema database)
  └── Migrations/ folder   (15 phút — lịch sử thay đổi DB)

Ngày 2–3:
  └── Chọn 3 tính năng nhỏ → trace end-to-end mỗi cái

Ngày 4+:
  └── Deep dive vào module quan trọng nhất
```

---

[← README](./README.md) · [Tiếp theo: Kiến trúc C# .NET + Blazor →](./02-kien-truc-dotnet-blazor.md)
