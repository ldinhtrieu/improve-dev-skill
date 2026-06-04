# 04 · Vẽ Sơ Đồ Khi Đọc Code

> Vẽ để não khỏi nhớ — không phải vẽ để đẹp.

---

## Nguyên tắc gốc

> **Một sơ đồ tốt là sơ đồ xấu nhưng đọc lại trong 10 giây là hiểu ngay.**
>
> Nếu phải ngồi đọc lại 2 phút mới hiểu — sơ đồ đó thất bại, dù đẹp đến đâu.

**Test trước khi vẽ thêm bất kỳ thứ gì:**
> *"Nếu không có thông tin này, mình có bị nhầm hoặc quên không?"*
>
> Nếu không → đừng vẽ.

---

## 3 loại sơ đồ — mỗi loại 1 mục đích

| Loại | Vẽ khi nào | Giữ bao lâu | Giới hạn cứng |
|------|-----------|-------------|---------------|
| **Map** | 1 lần lúc bắt đầu | Cả tuần, cập nhật khi thêm service | Tối đa 10 box, 1 tờ A4 |
| **Flow** | Khi trace 1 tính năng | Vẽ xong → có thể bỏ | Tối đa 8 bước, viết trong 3 phút |
| **Cheat sheet** | Ghi liên tục mỗi ngày | Giữ mãi | Mỗi mục tối đa 2 dòng |

---

## Loại 1 — Map (bản đồ hệ thống)

**Mục đích:** Biết mình đang đứng ở đâu trong hệ thống khi đọc code mới.

**Chỉ vẽ đúng 2 thứ:**
- Hình chữ nhật = module / service / project
- Mũi tên = cái này gọi cái kia

### Ví dụ thực tế — stack C# + Blazor

```
┌─────────────┐     HttpClient     ┌──────────────┐
│  BlazorApp  │ ─────────────────► │  API Project │
│  (FE/UI)    │                    │  (Backend)   │
└─────────────┘                    └──────┬───────┘
                                          │ EF Core
                                   ┌──────▼───────┐
                                   │  SQL Server  │
                                   │  (Database)  │
                                   └──────────────┘
                                          
                                   ┌──────────────┐
                              ┌───►│  SendGrid    │ (Email)
                              │    └──────────────┘
                    ┌─────────┴──┐
                    │ API Project│
                    └─────────┬──┘
                              │    ┌──────────────┐
                              └───►│  VNPay API   │ (Payment)
                                   └──────────────┘
```

**Tuyệt đối không vẽ vào Map:**
- ❌ Tên hàm, method
- ❌ Fields, properties
- ❌ Màu sắc phân loại
- ❌ Legend dài
- ❌ Nhiều hơn 3 chữ trong tên box

---

## Loại 2 — Flow (luồng tính năng)

**Mục đích:** Nhớ thứ tự gọi nhau, không cần sơ đồ hình học.

**Format chuẩn — viết text, không cần hình:**

```
[Feature: Tạo đơn hàng]

1. OrderPage.razor :: HandleSubmit()
        ↓ truyền CreateOrderRequest
2. IOrderService :: CreateAsync(request)
        ↓ validate + map sang entity
3. IInventoryService :: ReserveAsync(items)   ← side effect quan trọng!
        ↓
4. IOrderRepository :: InsertAsync(order)
        ↓ EF Core
5. DB: INSERT Orders → INSERT OrderItems (bulk)
        ↓ trả về orderId
6. NavigationManager.NavigateTo(/orders/{id})
```

### Quy tắc xử lý if/else

**Không bao giờ vẽ if/else vào flow chính.** Có 3 kỹ thuật:

---

### Kỹ thuật 1 — Ghi chú nhánh lỗi bên phải (dùng 90%)

```
1. ValidateRequest(dto)           ✗ null/invalid → 400 Bad Request
        ↓
2. CheckUser(userId)              ✗ not found → 401 Unauthorized
        ↓
3. CheckStock(items)              ✗ hết hàng → 400 + message cụ thể
        ↓
4. InsertOrder() + Items          ✗ DB fail → rollback transaction
        ↓
5. SendConfirmationEmail()        ✗ fail → log warning, không throw
        ↓
6. Return orderId
```

Dấu `✗` ở bên phải không thuộc luồng chính — não đọc tự bỏ qua khi hiểu flow, nhưng vẫn nhớ "chỗ này có xử lý lỗi".

---

### Kỹ thuật 2 — Tách flow riêng khi nhánh dẫn đến luồng khác hẳn

Dùng khi `if/else` **thay đổi thứ tự các bước tiếp theo:**

```
[Flow chính: Checkout]

1. ValidateCart()
        ↓
2. CheckPaymentMethod()
        ├── COD     → xem [Flow: COD]
        └── Online  → xem [Flow: Online Payment]

─────────────────────────────────────

[Flow: COD]
1. InsertOrder(status = PendingDelivery)
2. NotifyWarehouse()
3. Return orderId

─────────────────────────────────────

[Flow: Online Payment]
1. CallVNPayGateway(amount)       ✗ timeout → retry 3 lần
2. InsertOrder(status = PendingPayment)
3. StartPaymentTimeoutJob(30 phút)
4. Redirect đến VNPay URL
```

---

### Kỹ thuật 3 — Collapse nhiều điều kiện thành 1 dòng có tên

```csharp
// Code thực tế có thể như này — 6 điều kiện validate
if (dto == null) throw new BadRequestException("Request null");
if (dto.Items.Count == 0) throw new BadRequestException("Giỏ trống");
if (dto.Items.Any(x => x.Quantity <= 0)) throw ...
if (dto.DeliveryAddress == null) throw ...
if (string.IsNullOrEmpty(dto.PhoneNumber)) throw ...
if (!IsValidPhone(dto.PhoneNumber)) throw ...
```

Trong flow — ghi gộp 1 dòng:

```
1. ValidateOrderRequest(dto)     ✗ → 400 với message cụ thể
```

Không liệt kê từng điều kiện. Khi cần biết chi tiết → mở code ra đọc.

---

### Quyết định có vẽ nhánh hay không

```
Hỏi: "Nhánh này có thay đổi thứ tự các bước tiếp theo không?"
        │
        ├── Không (chỉ throw / return sớm)
        │       └── Ghi chú ✗ bên phải → tiếp tục flow chính
        │
        └── Có (dẫn đến 3+ bước xử lý khác)
                └── Tách thành flow riêng
```

---

## Loại 3 — Cheat Sheet (ghi chú tra cứu)

**Mục đích:** Không phải google lại những gì đã biết từ codebase này.

**Format — text thuần, không cần layout đẹp:**

```markdown
== OrderService ==
- Tạo order → CreateAsync() trong Services/OrderService.cs ~line 45
- Status enum: Pending/Processing/Shipped/Done/Cancelled
  → File: Domain/Enums/OrderStatus.cs
- Khi tạo order PHẢI gọi _inventoryService.ReserveAsync() trước (dễ quên!)
- Sau khi tạo → gửi email qua _emailService.SendOrderConfirmation()

== Auth ==
- Lấy UserId hiện tại:
  _httpContextAccessor.HttpContext?.User.GetUserId()
- Extension method GetUserId() ở:
  Extensions/ClaimsPrincipalExtensions.cs
- Role check: [Authorize(Roles = "Admin")] hoặc User.IsInRole("Admin")

== Blazor Gotcha ==
- DbContext KHÔNG inject trực tiếp vào component
  → Dùng IDbContextFactory<AppDbContext>
- UI không tự update sau async → gọi StateHasChanged()
- [CascadingParameter] AuthState lấy từ CascadingAuthenticationState
  → Được wrap trong MainLayout.razor

== Database ==
- Soft delete: cột IsDeleted (bool), KHÔNG xóa thật
- Tất cả query phải có .Where(x => !x.IsDeleted)
- CreatedAt / UpdatedAt tự động set trong AppDbContext.SaveChangesAsync()
  → Override method này trong AppDbContext.cs ~line 89

== Conventions lạ của team ==
- DTO suffix: ...Dto (input) vs ...Response (output)
- Tất cả API response wrap trong ApiResponse<T>
  → Xem Models/Common/ApiResponse.cs
```

**Ghi vào cheat sheet khi nào:**
- Phát hiện điều "à, hóa ra là vậy"
- Convention lạ của team không có trong tài liệu
- Gotcha đã bị mắc phải
- "Để làm X → vào file Y" — những shortcut sẽ dùng lại

---

## Template Flow — Copy và dùng ngay

```
[Feature: _______________]
Ngày đọc: _______________

HAPPY PATH:
1. [Component/Page].razor :: [HandlerMethod]()
        ↓ [data truyền đi]
2. I[Service] :: [method](params)         ✗ [lỗi] → [xử lý]
        ↓
3. I[Repository] :: [method]()            ✗ [lỗi] → [xử lý]
        ↓ EF Core
4. DB: [INSERT/UPDATE/SELECT] bảng [tên bảng]
        ↓
5. Return / Navigate / StateHasChanged

NHÁNH TÁCH RIÊNG:
- [Điều kiện] → xem [Flow: tên flow]

GHI CHÚ:
- [Điều quan trọng cần nhớ]
- [Side effect không rõ ràng]
```

---

## Dấu hiệu đang vẽ quá nhiều

Dừng lại nếu:
- ⚠️ Vẽ quá 10 phút mà chưa xong
- ⚠️ Sơ đồ có nhiều hơn 3 màu
- ⚠️ Có mũi tên 2 chiều phức tạp chằng chịt
- ⚠️ Phải giải thích sơ đồ mới hiểu được
- ⚠️ Đang vẽ vào chi tiết của framework (EF Core internals, middleware...)

**Khi chạm giới hạn:** Tách thành sơ đồ mới, đừng nhồi thêm vào cái cũ.

---

[← Hướng dẫn Debug](./03-huong-dan-debug.md) · [← README](./README.md)
