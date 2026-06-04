# 02 · Kiến Trúc C# .NET Core 8 + Blazor Server + API

> Hiểu cấu trúc tầng lớp để biết cần đọc file nào, class nào.

---

## Sơ đồ kiến trúc tổng thể

```
┌─────────────────────────────────────────────┐
│              Blazor Server (FE)             │
│                                             │
│  Pages/          Components/    Services/   │
│  .razor files    .razor files   @inject     │
│                                             │
│  ← Kết nối server qua SignalR WebSocket →  │
└──────────────────┬──────────────────────────┘
                   │ HttpClient / Service call
┌──────────────────▼──────────────────────────┐
│           ASP.NET Core 8 API                │
│                                             │
│  Controllers/    Services/    Repositories/ │
│  [ApiController] IXxxService  IXxxRepo      │
│  [Route]         XxxService   EF Core       │
│                                             │
│  Middleware: Auth → Validation → Logging    │
└──────────────────┬──────────────────────────┘
                   │ EF Core
┌──────────────────▼──────────────────────────┐
│              Database Layer                 │
│                                             │
│  AppDbContext    Entities/     Migrations/  │
│  DbSet<T>        Models        Schema hist  │
└─────────────────────────────────────────────┘
```

---

## `Program.cs` — Trung tâm wiring toàn bộ hệ thống

Đây là file **quan trọng nhất** để đọc đầu tiên. Mọi dependency đều được khai báo ở đây.

```csharp
var builder = WebApplication.CreateBuilder(args);

// ── DATABASE ──────────────────────────────────────
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// ── REPOSITORIES ──────────────────────────────────
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<IProductRepository, ProductRepository>();

// ── SERVICES (Business Logic) ─────────────────────
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddScoped<IProductService, ProductService>();

// ── EXTERNAL HTTP CLIENTS ─────────────────────────
builder.Services.AddHttpClient<IPaymentService, PaymentService>(client =>
    client.BaseAddress = new Uri(builder.Configuration["PaymentGateway:BaseUrl"]!));

// ── AUTH ──────────────────────────────────────────
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => { ... });

// ── MIDDLEWARE PIPELINE ───────────────────────────
var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();           // API controllers
app.MapBlazorHub();             // Blazor Server SignalR hub
app.MapFallbackToPage("/_Host");
```

> 💡 **Cách đọc nhanh:** Tìm tất cả dòng `builder.Services.Add*` → đó là danh sách toàn bộ components của hệ thống.

---

## Blazor Server — Cách đọc file `.razor`

### Cấu trúc một component `.razor`

```razor
@page "/orders"                          ← 1. Route URL
@inject IOrderService OrderService       ← 2. Dependency injection
@inject NavigationManager Nav

<!-- 3. Template HTML -->
<h1>Danh sách đơn hàng</h1>

@if (orders == null)
{
    <p>Đang tải...</p>
}
else
{
    @foreach (var order in orders)
    {
        <OrderCard Order="order" OnCancel="HandleCancel" />  ← Component con
    }
}

<button @onclick="CreateOrder">Tạo đơn mới</button>

@code {
    // 4. C# logic
    private List<OrderDto>? orders;

    // 5. Lifecycle — data thường load ở đây
    protected override async Task OnInitializedAsync()
    {
        orders = await OrderService.GetAllAsync();
    }

    // 6. Event handlers
    async Task CreateOrder()
    {
        Nav.NavigateTo("/orders/create");
    }

    async Task HandleCancel(int orderId)
    {
        await OrderService.CancelAsync(orderId);
        orders = await OrderService.GetAllAsync(); // reload
    }
}
```

### Lifecycle của Blazor Component — thứ tự thực thi

```
Component được khởi tạo
    ↓
SetParametersAsync()        ← [Parameter] được set lần đầu
    ↓
OnInitialized()             ← sync, chạy 1 lần
OnInitializedAsync()        ← async, chạy 1 lần → ĐẶT BREAKPOINT Ở ĐÂY
    ↓
OnParametersSet()           ← chạy mỗi khi parameter thay đổi
OnParametersSetAsync()
    ↓
BuildRenderTree()           ← render HTML
    ↓
OnAfterRender()             ← sau khi render xong (dùng cho JS interop)
OnAfterRenderAsync()
```

---

## Service Layer — Pattern thường gặp

```csharp
// Interface — tìm file này để biết service làm được gì
public interface IOrderService
{
    Task<List<OrderDto>> GetAllAsync();
    Task<OrderDto?> GetByIdAsync(int id);
    Task<int> CreateAsync(CreateOrderRequest request);
    Task CancelAsync(int id);
}

// Implementation — đây là logic thực tế
public class OrderService : IOrderService
{
    private readonly IOrderRepository _repo;
    private readonly IInventoryService _inventory;  // ← dependency khác

    // Constructor injection — xem đây để biết service này phụ thuộc gì
    public OrderService(IOrderRepository repo, IInventoryService inventory)
    {
        _repo = repo;
        _inventory = inventory;
    }

    public async Task<int> CreateAsync(CreateOrderRequest request)
    {
        // Business logic ở đây
        await _inventory.ReserveAsync(request.Items);  // side effect!
        var order = MapToEntity(request);
        return await _repo.InsertAsync(order);
    }
}
```

> 💡 **Tip đọc service:** Nhìn vào constructor trước — biết ngay service này phụ thuộc gì và có thể làm gì.

---

## Repository / DbContext — Tầng database

### Đọc `AppDbContext` để hiểu toàn bộ schema

```csharp
public class AppDbContext : DbContext
{
    // Mỗi DbSet = 1 bảng trong database
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }
    public DbSet<Product> Products { get; set; }
    public DbSet<User> Users { get; set; }

    protected override void OnModelCreating(ModelBuilder mb)
    {
        // Fluent API — cấu hình quan hệ, index, constraint
        mb.Entity<Order>(entity =>
        {
            entity.HasKey(o => o.Id);
            entity.Property(o => o.Status).HasConversion<string>();

            // Quan hệ 1-nhiều: 1 Order có nhiều OrderItems
            entity.HasMany(o => o.Items)
                  .WithOne(i => i.Order)
                  .HasForeignKey(i => i.OrderId)
                  .OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

### Repository pattern thực tế

```csharp
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public async Task<List<Order>> GetAllAsync()
    {
        return await _context.Orders
            .Include(o => o.Items)          // JOIN với OrderItems
            .Include(o => o.Customer)       // JOIN với Users
            .Where(o => !o.IsDeleted)       // soft delete
            .OrderByDescending(o => o.CreatedAt)
            .ToListAsync();
    }

    public async Task<int> InsertAsync(Order order)
    {
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();  // ← đây là lúc SQL INSERT chạy
        return order.Id;
    }
}
```

---

## EF Core Migrations — Đọc lịch sử schema DB

```bash
# Folder Migrations/ chứa toàn bộ lịch sử thay đổi database
Migrations/
├── 20240101_InitialCreate.cs       ← schema ban đầu
├── 20240215_AddOrderStatus.cs      ← thêm cột Status
├── 20240310_AddIndexOnCreatedAt.cs ← thêm index
└── AppDbContextModelSnapshot.cs    ← schema hiện tại (đọc file này)
```

```csharp
// Trong file Migration, đọc phần Up() để biết thay đổi gì
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.AddColumn<string>(
        name: "Status",
        table: "Orders",
        nullable: false,
        defaultValue: "Pending");

    migrationBuilder.CreateIndex(
        name: "IX_Orders_CreatedAt",
        table: "Orders",
        column: "CreatedAt");
}
```

> 💡 **Tip nhanh:** Đọc `AppDbContextModelSnapshot.cs` để xem schema **hiện tại** mà không cần đọc từng migration.

---

## Bật EF Core SQL Logging — Xem câu SQL thực tế

Thêm vào `appsettings.Development.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

Khi chạy ở Debug mode, Output window sẽ hiển thị:

```sql
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (3ms)
      SELECT o.Id, o.Status, o.CreatedAt, u.Name
      FROM Orders AS o
      INNER JOIN Users AS u ON o.UserId = u.Id
      WHERE o.IsDeleted = 0
      ORDER BY o.CreatedAt DESC
```

---

[← Phương pháp đọc code](./01-phuong-phap-doc-code.md) · [Tiếp theo: Hướng dẫn Debug →](./03-huong-dan-debug.md)
