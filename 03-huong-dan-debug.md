# 03 · Hướng Dẫn Debug C# .NET Core + Blazor Server

> Giải quyết 2 vấn đề phổ biến nhất: đi quá sâu và không biết data lấy từ đâu.

---

## 2 vấn đề hay gặp khi debug

| Vấn đề | Biểu hiện | Giải pháp |
|--------|-----------|-----------|
| **Đi quá sâu** | Nhảy vào code framework, EF Core internals, không biết đang ở đâu | Dùng đúng F10/F11/Shift+F11 |
| **Data từ đâu** | Vào component thấy data đã có sẵn, không biết ai load | Breakpoint tại `OnInitializedAsync`, trace ngược |

---

## 3 phím debug — phân biệt rõ ràng

| Phím | Tên | Dùng khi nào |
|------|-----|-------------|
| `F10` | **Step Over** | Chạy dòng này, KHÔNG nhảy vào hàm — dùng **80% thời gian** |
| `F11` | **Step Into** | Nhảy vào bên trong hàm — chỉ dùng khi muốn xem hàm đó làm gì |
| `Shift+F11` | **Step Out** | Thoát ra khỏi hàm đang đứng — dùng khi lỡ F11 quá sâu |
| `Ctrl+F10` | **Run to Cursor** | Nhảy thẳng đến dòng đang click, bỏ qua phần giữa |
| `F5` | **Continue** | Chạy đến breakpoint tiếp theo |
| `Shift+F9` | **Quick Watch** | Eval bất kỳ expression nào ngay lúc đang dừng |

### Nguyên tắc

```
Mặc định dùng F10 toàn bộ.
Chỉ F11 vào hàm do MÌNH tự viết.
Khi thấy đang đứng trong EF Core / ASP.NET Core / thư viện → Shift+F11 thoát ngay.
```

---

## Vấn đề 1 — Debug đi quá sâu

### Nguyên nhân
Chỉ biết `F11 Step Into`, nhảy vào mọi thứ kể cả code framework.

### Giải pháp: Just My Code

Bật **Just My Code** trong Visual Studio:

```
Tools → Options → Debugging → General
☑ Enable Just My Code
```

Khi bật, `F11` sẽ không nhảy vào code framework — chỉ nhảy vào code của bạn.

### Call Stack — đọc ngược để biết đang ở đâu

Khi bị lost, mở **Debug → Windows → Call Stack** (`Ctrl+Alt+C`):

```
▶ OrderService.CreateAsync()           ← đang đứng đây
  OrderController.Post()               ← controller gọi service
  ASP.NET Core middleware pipeline     ← framework
  Kestrel HTTP server                  ← framework
```

Click vào bất kỳ frame nào → IDE nhảy đến đúng dòng code đó.

> 💡 **Tip:** Call Stack đọc từ trên xuống = từ trong ra ngoài. Frame trên cùng = đang đứng ở đây.

### Run to Cursor — bỏ qua code không cần

```csharp
public async Task<int> CreateAsync(CreateOrderRequest request)
{
    // Có 20 dòng validation không cần debug
    ValidateRequest(request);
    CheckPermissions(request.UserId);
    NormalizeData(request);
    // ... 17 dòng khác ...

    // Chỉ cần debug từ đây
    var order = MapToEntity(request);  // ← Click vào dòng này, bấm Ctrl+F10
    return await _repo.InsertAsync(order);
}
```

---

## Vấn đề 2 — Không biết data lấy từ đâu

### Bước 1: Tìm điểm load data của Blazor component

Data trong Blazor thường được load tại:

```csharp
// ✅ Chỗ 1 — Phổ biến nhất, đặt breakpoint ở đây trước
protected override async Task OnInitializedAsync()
{
    // ← F9 đặt breakpoint ngay dòng đầu tiên
    Orders = await OrderService.GetAllAsync();
}

// ✅ Chỗ 2 — Khi component nhận parameter từ URL
protected override async Task OnParametersSetAsync()
{
    if (OrderId != _prevOrderId)
    {
        Order = await OrderService.GetByIdAsync(OrderId);
    }
}

// ✅ Chỗ 3 — Event handler khi user tương tác
async Task HandleSearch(string keyword)
{
    Results = await SearchService.SearchAsync(keyword);
}
```

### Bước 2: Break When Value Changes

Khi thấy biến đã có data mà không biết ai set:

```
1. Chạy đến điểm có biến đó (đặt breakpoint thường trước)
2. Trong cửa sổ Locals/Watch → click phải vào biến
3. Chọn "Break When Value Changes"
4. F5 tiếp tục → VS tự dừng ngay khi biến bị gán
```

### Bước 3: Trace ngược bằng Find All References

```
Thấy: this.CurrentUser đã có data
       ↓
Shift+F12 trên "CurrentUser"
       ↓
VS hiện tất cả chỗ reference
       ↓
Tìm dòng có dấu = (assignment): CurrentUser = ...
       ↓
F9 đặt breakpoint tại đó → chạy lại → biết chính xác ai gán
```

### Bước 4: Setter breakpoint (chắc chắn nhất)

```csharp
private UserDto? _currentUser;
public UserDto? CurrentUser
{
    get => _currentUser;
    set
    {
        // Đặt F9 breakpoint ở đây
        // Mỗi lần có ai gán CurrentUser = ... đều dừng lại
        _currentUser = value;
        // Mở Call Stack → biết ngay ai đang gán
    }
}
```

---

## Blazor Server — Những điều cần biết khi debug

### 1. Breakpoint hoạt động 100% bình thường

Vì toàn bộ C# chạy trên server, đặt breakpoint trong Visual Studio hoạt động như debug app .NET thông thường — không cần cấu hình thêm.

### 2. Browser treo khi đang ở breakpoint — bình thường

```
Đang debug → VS dừng tại breakpoint
           → SignalR bị block
           → Browser hiển thị "reconnecting..." hoặc trắng màn hình
           → Đây là BÌNH THƯỜNG
           → Bấm F5 Continue → browser tự recover
```

### 3. Mỗi tab browser = 1 circuit riêng

```
Tab 1 → Circuit 1 → Scoped services instance 1
Tab 2 → Circuit 2 → Scoped services instance 2
```

Nếu mở nhiều tab, breakpoint có thể bị trigger bởi bất kỳ tab nào. Nên đóng bớt tab khi debug.

### 4. DbContext trong Blazor Server

```csharp
// ❌ SAI — inject trực tiếp vào component
@inject AppDbContext DbContext  // DbContext là Scoped, dễ bị concurrency issue

// ✅ ĐÚNG — dùng IDbContextFactory
@inject IDbContextFactory<AppDbContext> DbFactory

@code {
    async Task LoadData()
    {
        await using var db = await DbFactory.CreateDbContextAsync();
        // ← đặt breakpoint đây, xem db có data gì
        Orders = await db.Orders.ToListAsync();
    }
}
```

---

## Workflow debug chuẩn cho 1 tính năng mới

```
1. Mở VS → F5 (Debug mode)

2. Mở trang cần hiểu trên browser

3. Đặt BP tại OnInitializedAsync() của component đó

4. Reload trang (F5 trên browser) → VS dừng

5. F10 đi từng bước, nhìn Locals window xem giá trị thay đổi
   ├── Thấy service method cần hiểu → F11 vào
   ├── Xong → Shift+F11 ra
   └── Thấy code framework → Shift+F11 thoát ngay

6. Mở Call Stack nếu bị lost → biết đang ở đâu

7. Shift+F9 (Quick Watch) để eval expression bất kỳ
   Ví dụ: gõ "orders.Count" hay "order.Status == OrderStatus.Pending"

8. Bật EF Core SQL logging → xem câu SQL thực tế trong Output window
```

---

## Windows debug hay dùng nhất

| Window | Mở bằng | Dùng để |
|--------|---------|---------|
| **Locals** | Tự động hiện khi debug | Xem tất cả biến local hiện tại |
| **Watch** | Debug → Windows → Watch | Theo dõi expression cụ thể qua nhiều bước |
| **Call Stack** | `Ctrl+Alt+C` | Xem luồng gọi hàm, navigate đến frame bất kỳ |
| **Immediate Window** | `Ctrl+Alt+I` | Chạy code C# ngay lúc đang dừng |
| **Output** | `Ctrl+Alt+O` | Xem log, EF Core SQL queries |

### Quick Watch — dùng nhiều hơn

```
Đang dừng tại breakpoint
→ Shift+F9
→ Gõ bất kỳ expression C# hợp lệ:
   orders.Count
   orders.Where(o => o.Status == "Pending").ToList()
   currentUser?.Email ?? "null"
   JsonSerializer.Serialize(order)   ← xem toàn bộ object dạng JSON
```

---

[← Kiến trúc .NET Blazor](./02-kien-truc-dotnet-blazor.md) · [Tiếp theo: Vẽ sơ đồ →](./04-ve-so-do.md)
