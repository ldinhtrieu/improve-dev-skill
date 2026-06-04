# 01d · Phương Pháp Đọc Code — Java Spring Boot

---

## Phase 1 — Entry Points & Cấu hình

**File đọc theo thứ tự:**

| File | Biết được gì |
|------|-------------|
| `pom.xml` / `build.gradle` | Dependencies, plugins, Spring Boot version |
| `src/main/resources/application.yml` | DB, ports, external services, profiles |
| `*Application.java` (có `@SpringBootApplication`) | Entry point, component scan scope |
| `src/main/resources/application-dev.yml` | Override config cho dev environment |

```java
// XxxApplication.java — điểm khởi động
@SpringBootApplication          // = @Configuration + @EnableAutoConfiguration + @ComponentScan
@EnableJpaRepositories          // nếu có annotation thêm → biết feature được bật
@EnableScheduling               // → có scheduled jobs
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}
```

```yaml
# application.yml — đọc để biết hệ thống kết nối gì
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orders_db
  jpa:
    hibernate:
      ddl-auto: validate       # ← validate/update/create — quan trọng!
    show-sql: true             # ← bật để thấy SQL thực tế

external-services:
  payment:
    base-url: https://payment.example.com
  email:
    api-key: ${EMAIL_API_KEY}  # ← lấy từ environment variable
```

---

## Phase 2 — Kiến trúc tổng thể

### Cấu trúc thư mục → nhận diện pattern

**Layered Architecture (phổ biến nhất):**
```
src/main/java/com/example/
├── controller/         → REST Controllers (@RestController)
├── service/            → Business logic (@Service)
├── repository/         → Data access (@Repository / JpaRepository)
├── entity/ / domain/   → JPA Entities (@Entity)
├── dto/                → Data Transfer Objects
├── config/             → @Configuration classes
├── exception/          → Custom exceptions + @ControllerAdvice
└── mapper/             → Entity ↔ DTO mapping (MapStruct)
```

**Domain-Driven Design:**
```
src/main/java/com/example/
├── order/              → Order bounded context
│   ├── OrderController.java
│   ├── OrderService.java
│   ├── OrderRepository.java
│   └── Order.java
└── inventory/
```

### Spring DI — Annotations quan trọng

```java
@Component      // generic bean
@Service        // business logic layer
@Repository     // data access layer (+ exception translation)
@Controller     // MVC controller
@RestController // = @Controller + @ResponseBody

// Tìm toàn bộ beans:
// Ctrl+Shift+F → tìm "@Service" hoặc "@Component"
// Hoặc dùng IntelliJ Spring Beans panel (View → Tool Windows → Spring)
```

```java
// Constructor injection (recommended — dễ đọc nhất)
@Service
public class OrderService {
    private final OrderRepository orderRepo;
    private final InventoryService inventoryService;
    private final EmailService emailService;

    // @Autowired có thể bỏ nếu chỉ có 1 constructor (Spring 4.3+)
    public OrderService(OrderRepository orderRepo,
                        InventoryService inventoryService,
                        EmailService emailService) {
        this.orderRepo = orderRepo;           // ← đọc constructor biết dependency
        this.inventoryService = inventoryService;
        this.emailService = emailService;
    }
}
```

---

## Phase 3 — Data Flow

### Request lifecycle — Spring MVC

```
HTTP Request
    ↓
DispatcherServlet (Front Controller)
    ↓
HandlerMapping → tìm @RequestMapping match
    ↓
Filter chain (Security, CORS, Logging)
    ↓
@RestController method
    ↓ @Valid → validation
@Service method (business logic)
    ↓
@Repository / JpaRepository
    ↓ Hibernate → SQL
Database
    ↓ Entity → DTO (MapStruct / manual)
Jackson → JSON response
```

### Tìm endpoint nhanh

```java
// Tìm từ URL → tìm @RequestMapping / @GetMapping / @PostMapping
// Ctrl+Shift+F → tìm "/api/orders"

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<OrderResponse> create(
            @Valid @RequestBody CreateOrderRequest request,  // ← validate input
            @AuthenticationPrincipal UserDetails user) {    // ← current user
        OrderResponse response = orderService.create(request, user.getUsername());
        return ResponseEntity.status(201).body(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> getById(@PathVariable Long id) {
        return orderService.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }
}
```

### JPA / Hibernate — đọc query

```java
// Spring Data JPA — method name = query
public interface OrderRepository extends JpaRepository<Order, Long> {
    // Đọc tên method → biết SQL tương đương
    List<Order> findByUserIdAndStatus(Long userId, OrderStatus status);
    // → SELECT * FROM orders WHERE user_id = ? AND status = ?

    // JPQL query rõ ràng hơn
    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.userId = :userId")
    List<Order> findWithItemsByUserId(@Param("userId") Long userId);

    // Native SQL
    @Query(value = "SELECT * FROM orders WHERE total_amount > :amount",
           nativeQuery = true)
    List<Order> findExpensiveOrders(@Param("amount") BigDecimal amount);
}
```

### Bật SQL logging

```yaml
# application-dev.yml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true  # in SQL có indent cho dễ đọc

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql: TRACE  # in cả parameter values
```

---

## Phase 4 — Feature Trace

### Checklist trace 1 tính năng

- [ ] Tìm `@RequestMapping` / `@GetMapping` / `@PostMapping` → controller
- [ ] Đọc `@Service` method được gọi từ controller
- [ ] Trace vào `@Repository` → đọc tên method / `@Query`
- [ ] Tìm `@Entity` class → xem mapping với bảng DB
- [ ] Tìm `@Migration` / `schema.sql` / Flyway / Liquibase → schema thực tế

### Entity → hiểu DB schema

```java
@Entity
@Table(name = "orders")  // ← tên bảng thực tế
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;  // ← lưu dạng string hay int?

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<OrderItem> items;  // ← LAZY = không load tự động → dễ gây N+1

    @CreationTimestamp
    private LocalDateTime createdAt;
}
```

### Nguồn gốc data — các annotation quan trọng

| Thấy trong code | Nguồn gốc | Cách tìm |
|-----------------|-----------|----------|
| `@Autowired` / constructor param | Spring DI | Tìm `@Bean` hoặc `@Service/@Component` |
| `@Value("${key}")` | `application.yml` | Tìm key trong config files |
| `@AuthenticationPrincipal` | Spring Security context | Xem Security config, UserDetailsService |
| `@PathVariable`, `@RequestParam` | URL / query string | Xem controller mapping |

---

## Phase 5 — Deep Dive

### Tìm nguồn gốc bean / service

```bash
# IntelliJ IDEA
Ctrl+Alt+B     → Go to Implementation (từ interface → class)
Ctrl+B / F12   → Go to Definition
Alt+F7         → Find Usages
Ctrl+F12       → File structure (tất cả methods trong file)

# Tìm toàn project
Ctrl+Shift+F   → Find in Files
Ctrl+N         → Find Class
Ctrl+Shift+N   → Find File
```

### Spring AOP — khi code "tự nhiên" có behavior

```java
// Thấy method tự nhiên có transaction / logging / caching mà không thấy code?
// Tìm @Aspect classes và @Around, @Before, @After

@Aspect
@Component
public class LoggingAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object logMethod(ProceedingJoinPoint jp) throws Throwable {
        log.info("Calling: {}", jp.getSignature());
        Object result = jp.proceed();
        log.info("Done: {}", jp.getSignature());
        return result;
    }
}

// @Transactional → tìm nơi @EnableTransactionManagement được bật
// @Cacheable → tìm @EnableCaching và CacheManager bean
```

### Thứ tự đọc lý tưởng — Ngày 1

```
pom.xml / build.gradle    (10 phút — dependencies)
application.yml           (15 phút — config, DB, external)
*Application.java         (5 phút  — entry point, annotations)
entity/ hoặc domain/      (20 phút — data model, quan hệ)
repository/ interfaces    (15 phút — queries có sẵn)
```

---

[← Go](./01c-golang.md) · [Index](./README.md) · [Kotlin Android →](./01e-kotlin-android.md)
