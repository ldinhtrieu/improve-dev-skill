# 01c · Phương Pháp Đọc Code — Go (Golang)

---

## Phase 1 — Entry Points & Cấu hình

**File đọc theo thứ tự:**

| File | Biết được gì |
|------|-------------|
| `go.mod` | Module name, Go version, tất cả dependencies |
| `go.sum` | Lock file — versions chính xác |
| `main.go` hoặc `cmd/*/main.go` | Điểm khởi động, wiring toàn bộ app |
| `config/` hoặc `.env` | Database, ports, external services |
| `Makefile` | Các lệnh build, test, run quan trọng |

```go
// main.go — đọc function main() để hiểu hệ thống khởi động thế nào
func main() {
    // 1. Load config
    cfg := config.Load()

    // 2. Khởi tạo database
    db := database.NewPostgres(cfg.DatabaseURL)

    // 3. Wire dependencies (manual DI — không có container)
    orderRepo := repository.NewOrderRepository(db)
    inventoryService := service.NewInventoryService(db)
    orderService := service.NewOrderService(orderRepo, inventoryService)

    // 4. Setup router
    router := handler.NewRouter(orderService)

    // 5. Start server
    log.Fatal(http.ListenAndServe(":8080", router))
}
```

> 💡 **Quan trọng:** Go không có DI framework như Spring hay .NET. Toàn bộ dependency được wire **thủ công** trong `main.go` (hoặc `wire.go` nếu dùng Google Wire). Đây chính là "bản đồ hệ thống".

---

## Phase 2 — Kiến trúc tổng thể

### Cấu trúc thư mục — 2 pattern phổ biến

**Standard Go Layout (phổ biến):**
```
cmd/
├── api/main.go         → entry point API server
└── worker/main.go      → entry point background worker
internal/               → code KHÔNG thể import từ bên ngoài
├── handler/            → HTTP handlers (Controller equivalent)
├── service/            → Business logic
├── repository/         → Database layer
├── domain/             → Entities, interfaces
└── middleware/         → Auth, logging, rate limit
pkg/                    → Shared utilities (có thể import)
config/                 → Configuration
migrations/             → SQL migration files
```

**Flat layout (project nhỏ):**
```
main.go
handler.go
service.go
repository.go
model.go
```

### Interfaces — cách Go làm DI

```go
// Interface định nghĩa contract (thường trong domain/ hoặc cùng file)
type OrderRepository interface {
    FindByID(ctx context.Context, id int64) (*Order, error)
    Insert(ctx context.Context, order *Order) error
    List(ctx context.Context, filter OrderFilter) ([]*Order, error)
}

// Implementation
type postgresOrderRepository struct {
    db *sql.DB
}

func NewOrderRepository(db *sql.DB) OrderRepository {
    return &postgresOrderRepository{db: db}
}

// Khi thấy interface → tìm struct implement nó
// Ctrl+Click trên interface name → Go to Definition
// Sau đó tìm "implements OrderRepository" hoặc func (r *xxxRepository)
```

---

## Phase 3 — Data Flow

### Request lifecycle — Go HTTP server (Gin/Echo/net/http)

```
HTTP Request
    ↓
Router (gin.Engine / echo.Echo / http.ServeMux)
    ↓
Middleware chain (Auth, Logging, RateLimit, CORS)
    ↓
Handler function
    ↓ gọi service
Service (business logic)
    ↓ gọi repository
Repository
    ↓ database/sql hoặc sqlx hoặc GORM
SQL Database
    ↓ rows.Scan() → struct
Handler → c.JSON(200, response)
```

### Ví dụ trace request cụ thể

```go
// Bước 1: Tìm route registration
r.POST("/api/orders", orderHandler.Create)  // ← Gin
e.POST("/api/orders", orderHandler.Create)  // ← Echo

// Bước 2: Đọc handler
func (h *OrderHandler) Create(c *gin.Context) {
    var req CreateOrderRequest
    if err := c.ShouldBindJSON(&req); err != nil {  // ← parse request
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }

    order, err := h.service.CreateOrder(c.Request.Context(), req)  // ← nhảy vào đây
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }
    c.JSON(201, order)
}

// Bước 3: Đọc service
func (s *orderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    // business logic
    if err := s.inventory.Reserve(ctx, req.Items); err != nil {  // ← side effect
        return nil, fmt.Errorf("reserve inventory: %w", err)
    }
    return s.repo.Insert(ctx, &Order{...})  // ← nhảy vào repo
}
```

### Tìm SQL query thực tế

```go
// Repository — tìm các raw query
func (r *postgresOrderRepository) Insert(ctx context.Context, order *Order) error {
    query := `
        INSERT INTO orders (user_id, status, total_amount, created_at)
        VALUES ($1, $2, $3, NOW())
        RETURNING id
    `
    return r.db.QueryRowContext(ctx, query,
        order.UserID, order.Status, order.TotalAmount,
    ).Scan(&order.ID)
}

// Tip: Ctrl+Shift+F → tìm "INSERT INTO" hoặc "SELECT" để tìm tất cả queries
```

---

## Phase 4 — Feature Trace

### Checklist trace 1 tính năng

- [ ] Tìm route trong `main.go` hoặc `router.go`
- [ ] Tìm handler function tương ứng
- [ ] Đọc struct binding (`ShouldBindJSON`, `Decode`) → biết request shape
- [ ] Trace vào service method
- [ ] Trace vào repository → đọc SQL query
- [ ] Tìm migration file → schema thực tế của bảng

### Context — đọc `ctx` để hiểu data đi theo request

```go
// ctx trong Go không chỉ là deadline/cancellation
// Nó còn mang theo data qua middleware

// Middleware set value vào context
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID := validateToken(r.Header.Get("Authorization"))
        ctx := context.WithValue(r.Context(), "userID", userID)  // ← set
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Handler / Service lấy ra
userID := ctx.Value("userID").(int64)  // ← get

// Khi thấy ctx.Value("xxx") → tìm middleware nào đã set giá trị đó
```

---

## Phase 5 — Deep Dive

### Tìm nguồn gốc biến / struct field

```bash
# VS Code + Go extension (gopls)
F12           → Go to Definition
Shift+F12     → Find All References
Ctrl+Shift+F  → Search toàn project

# Terminal
grep -rn "funcName\|varName" ./internal/
```

### Error wrapping — trace lỗi từ đâu

```go
// Go dùng error wrapping, đọc theo chuỗi
return fmt.Errorf("orderService.Create: %w", err)
// → "orderService.Create: repository.Insert: sql: no rows"
// Đọc từ trái sang phải = từ ngoài vào trong call stack
```

### Goroutine — khi code chạy async

```go
// Thấy go func() → code chạy song song, không block
go func() {
    if err := emailService.Send(order.Email, template); err != nil {
        log.Printf("send email failed: %v", err)
    }
}()
// Tip: tìm channel (<-) và WaitGroup để hiểu cách kết quả được collect
```

### Thứ tự đọc lý tưởng — Ngày 1

```
go.mod              (5 phút  — dependencies, module name)
cmd/*/main.go       (30 phút — wiring toàn bộ app)
internal/domain/    (20 phút — interfaces, entities)
migrations/*.sql    (15 phút — schema database)
```

---

[← Angular](./01b-angular.md) · [Index](./README.md) · [Java Spring →](./01d-java-spring.md)
