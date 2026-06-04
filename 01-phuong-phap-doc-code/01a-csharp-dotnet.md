# 01a · Phương Pháp Đọc Code — C# .NET Core 8 + Blazor Server

---

## Phase 1 — Entry Points & Cấu hình

**File đọc theo thứ tự:**

| File | Biết được gì |
|------|-------------|
| `Program.cs` | Toàn bộ services, middleware pipeline, wiring DI |
| `appsettings.json` | Connection string, external services, feature flags |
| `appsettings.Development.json` | Config override cho môi trường dev |
| `docker-compose.yml` | Có bao nhiêu service, port nào dùng gì |

```csharp
// Program.cs — đọc tất cả dòng AddXxx để biết hệ thống có gì
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
builder.Services.AddHttpClient<IPaymentService, PaymentService>(client =>
    client.BaseAddress = new Uri(config["PaymentGateway:BaseUrl"]!));
```

---

## Phase 2 — Kiến trúc tổng thể

### Cấu trúc thư mục → nhận diện pattern

```
/Controllers        → Controller-based API (MVC)
/Endpoints          → Minimal API (.NET 7+)
/Services           → Business Logic layer
/Repositories       → Repository pattern
/Domain             → Domain-Driven Design
/Features           → Vertical Slice Architecture
/Middlewares        → Custom middleware
/Migrations         → EF Core database history
```

### Dependency Injection — toàn bộ trong `Program.cs`

```csharp
// Lifetime — quan trọng khi đọc code
builder.Services.AddSingleton<T>()   // 1 instance cả app
builder.Services.AddScoped<T>()      // 1 instance per request (API) / per circuit (Blazor)
builder.Services.AddTransient<T>()   // tạo mới mỗi lần inject
```

### Constructor injection — đọc đây biết ngay dependency

```csharp
public class OrderService : IOrderService
{
    // Nhìn vào constructor → biết service này phụ thuộc gì
    public OrderService(
        IOrderRepository repo,        // ← data access
        IInventoryService inventory,  // ← side effect
        IEmailService email,          // ← notification
        ILogger<OrderService> logger) // ← logging
    { ... }
}
```

---

## Phase 3 — Data Flow

### Request lifecycle — Blazor Server

```
User click / input (browser)
    ↓ SignalR WebSocket
Component event handler (@onclick, EventCallback)
    ↓ @inject Service
Service layer (business logic)
    ↓ IRepository
Repository / EF Core DbContext
    ↓ LINQ → SQL
Database (SQL Server / PostgreSQL)
    ↓ entity → DTO
StateHasChanged() → re-render component
```

### Request lifecycle — ASP.NET Core API

```
HTTP Request
    ↓
Middleware pipeline (UseAuthentication → UseAuthorization → ...)
    ↓
Controller [ApiController] / Minimal API MapPost(...)
    ↓ model binding + validation [Required], FluentValidation
Service layer
    ↓
Repository → EF Core → SQL
    ↓
DTO mapping (AutoMapper / manual) → JSON response
```

### Tìm endpoint nhanh

```bash
# Tìm route từ URL
Ctrl+Shift+F → tìm "api/orders" hoặc [Route("orders")]

# Từ controller nhảy vào service
F12 trên tên interface → Go to Definition
Ctrl+F12 → Go to Implementation (từ interface → class)
Shift+F12 → Find All References
```

---

## Phase 4 — Feature Trace

### Checklist trace 1 tính năng — Blazor

- [ ] Tìm file `.razor` của màn hình (tìm `@page "/ten-route"`)
- [ ] Đọc `@inject` ở đầu file → biết dùng service nào
- [ ] Đọc `OnInitializedAsync()` → biết data load từ đâu
- [ ] Trace vào service method → F12
- [ ] Trace xuống repository → F12 tiếp
- [ ] Xem `AppDbContext` → `DbSet<T>` → bảng nào liên quan
- [ ] Đọc `Migrations/` file mới nhất → schema thực tế

### Nguồn gốc biến trong Blazor component

| Thấy trong `.razor` | Nguồn gốc | Cách tìm |
|---------------------|-----------|----------|
| `@inject IXxxService Svc` | DI Container | Tìm `AddScoped<IXxx` trong `Program.cs` |
| `[Parameter]` | Component cha | Tìm `<TênComponent` trong các `.razor` khác |
| `[CascadingParameter]` | `<CascadingValue>` wrapper | Tìm trong layout / `App.razor` |
| `NavigationManager` | Built-in Blazor service | Luôn available |

---

## Phase 5 — Deep Dive

### Tìm nguồn gốc biến

```csharp
// Thấy: this.CurrentUser đã có data, không biết từ đâu
// Bước 1: Shift+F12 trên CurrentUser → Find All References
// Bước 2: Lọc dòng có dấu = (assignment)
// Bước 3: F9 đặt breakpoint tại đó → chạy lại → xem Call Stack
```

### Đọc AppDbContext — bản đồ database

```csharp
public class AppDbContext : DbContext
{
    // Mỗi DbSet = 1 bảng → đọc đây biết toàn bộ schema
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }
    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder mb)
    {
        // Quan hệ, index, constraint — đọc đây hiểu DB design
        mb.Entity<Order>()
            .HasMany(o => o.Items)
            .WithOne(i => i.Order)
            .HasForeignKey(i => i.OrderId);
    }
}
```

### Bật EF Core SQL logging

```json
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
// → Output window hiện SQL thực tế đang chạy
```

### Thứ tự đọc lý tưởng — Ngày 1

```
Program.cs          (30 phút — toàn bộ services)
appsettings.json    (10 phút — config, external)
AppDbContext.cs      (20 phút — schema database)
Migrations/ folder  (15 phút — lịch sử DB)
```

---

[← Index](./README.md) · [Angular →](./01b-angular.md)
