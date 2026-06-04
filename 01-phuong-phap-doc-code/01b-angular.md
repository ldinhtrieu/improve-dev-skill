# 01b · Phương Pháp Đọc Code — Angular (TypeScript)

---

## Phase 1 — Entry Points & Cấu hình

**File đọc theo thứ tự:**

| File | Biết được gì |
|------|-------------|
| `angular.json` | Build config, assets, environments, project structure |
| `package.json` | Dependencies, scripts (`ng serve`, `ng build`) |
| `src/main.ts` | Bootstrap app — điểm khởi động thực sự |
| `src/app/app.module.ts` | Modules, providers, imports toàn bộ app |
| `src/environments/` | Config theo môi trường (API URL, feature flags) |

```typescript
// main.ts — điểm khởi động
platformBrowserDynamic()
  .bootstrapModule(AppModule)  // ← nhảy vào AppModule
  .catch(err => console.error(err));
```

```typescript
// app.module.ts — "Program.cs" của Angular
@NgModule({
  imports: [
    BrowserModule,
    HttpClientModule,
    RouterModule.forRoot(routes),  // ← routes định nghĩa ở đây
    SharedModule,
    OrderModule,                   // ← feature modules
  ],
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
    { provide: API_BASE_URL, useValue: environment.apiUrl },
  ],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

> 💡 **Tip:** Nếu dùng **Standalone Components** (Angular 14+), không có `AppModule` — tìm `bootstrapApplication()` trong `main.ts` thay thế.

---

## Phase 2 — Kiến trúc tổng thể

### Cấu trúc thư mục → nhận diện pattern

```
src/app/
├── core/               → Singleton services (AuthService, ApiService, guards)
├── shared/             → Components/pipes/directives dùng chung
├── features/           → Feature modules (mỗi module = 1 tính năng lớn)
│   ├── orders/
│   │   ├── components/
│   │   ├── services/
│   │   ├── models/
│   │   └── orders.module.ts
│   └── products/
├── layouts/            → Shell components (header, sidebar, footer)
└── app-routing.module.ts
```

### Dependency Injection trong Angular

```typescript
// Injectable service — đọc providedIn để biết scope
@Injectable({ providedIn: 'root' })   // ← Singleton toàn app
export class OrderService { ... }

@Injectable({ providedIn: 'any' })    // ← mỗi lazy module 1 instance
export class CartService { ... }

// Inject vào component
@Component({ ... })
export class OrderListComponent {
  constructor(
    private orderService: OrderService,   // ← biết ngay dependency
    private router: Router,
    private store: Store<AppState>        // ← có dùng NgRx không?
  ) {}
}
```

### Routing — cách tìm component từ URL

```typescript
// app-routing.module.ts
const routes: Routes = [
  { path: 'orders', component: OrderListComponent },
  {
    path: 'orders/:id',
    component: OrderDetailComponent,
    canActivate: [AuthGuard]  // ← guard kiểm tra auth
  },
  {
    path: 'admin',
    loadChildren: () => import('./features/admin/admin.module')
      .then(m => m.AdminModule)  // ← lazy loaded module
  }
];
// Tìm URL → tìm route → tìm component → đọc component đó
```

---

## Phase 3 — Data Flow

### Request lifecycle — Angular Component

```
User action (click, input, route change)
    ↓
Component method / ngOnInit()
    ↓ Observable / Promise
Service (Angular service)
    ↓ HttpClient
HTTP Interceptor (auth header, error handling)
    ↓
API endpoint
    ↓ response
RxJS operators (map, catchError, tap)
    ↓
Component property → template binding {{ data }}
    ↓
Angular Change Detection → re-render
```

### RxJS — cách đọc Observable chain

```typescript
// Đọc từ trên xuống theo pipe()
this.orderService.getOrders().pipe(
  tap(orders => console.log('Loaded:', orders.length)),  // side effect
  map(orders => orders.filter(o => o.status === 'active')), // transform
  catchError(err => {                                    // error handling
    this.error = err.message;
    return EMPTY;
  })
).subscribe(orders => {
  this.orders = orders;  // ← data đến đây
});

// Tip: đọc subscribe() trước để biết kết quả cuối cùng là gì
// rồi đọc ngược lên pipe() để hiểu từng bước transform
```

### Tìm nơi gọi API

```typescript
// Trong service, tìm HttpClient calls
@Injectable({ providedIn: 'root' })
export class OrderService {
  private apiUrl = `${environment.apiUrl}/orders`;

  constructor(private http: HttpClient) {}

  getOrders(): Observable<Order[]> {
    return this.http.get<Order[]>(this.apiUrl);  // ← GET /api/orders
  }

  createOrder(dto: CreateOrderDto): Observable<Order> {
    return this.http.post<Order>(this.apiUrl, dto);  // ← POST /api/orders
  }
}
```

---

## Phase 4 — Feature Trace

### Checklist trace 1 tính năng

- [ ] Tìm route URL trong `app-routing.module.ts` hoặc feature routing
- [ ] Tìm component được load cho route đó
- [ ] Đọc `ngOnInit()` → biết data load từ đâu
- [ ] Tìm service được inject → đọc method được gọi
- [ ] Tìm `HttpClient.get/post` → biết API endpoint nào
- [ ] Xem HTTP Interceptors → có transform request/response không
- [ ] Đọc template HTML → binding với property nào

### Nguồn gốc data trong component

| Thấy trong component | Nguồn gốc | Cách tìm |
|----------------------|-----------|----------|
| `this.xxx` trong `ngOnInit` | Service call | Tìm method trong service |
| `@Input() data` | Component cha | Tìm `<app-component [data]=` |
| `this.route.snapshot.params` | URL params | Xem route config |
| `this.store.select(selector)` | NgRx Store | Tìm selector, tìm reducer |
| `this.activatedRoute.data` | Route resolver | Tìm `resolve:` trong route config |

### State Management — nếu dùng NgRx

```typescript
// Luồng NgRx đọc theo thứ tự:
// Component → dispatch Action → Effect → API → Action → Reducer → Store → Selector → Component

// Bước 1: Tìm action được dispatch
this.store.dispatch(loadOrders());

// Bước 2: Tìm Effect xử lý action đó
loadOrders$ = createEffect(() =>
  this.actions$.pipe(
    ofType(loadOrders),
    switchMap(() => this.orderService.getOrders().pipe(
      map(orders => loadOrdersSuccess({ orders })),
      catchError(err => of(loadOrdersFailure({ error: err })))
    ))
  )
);

// Bước 3: Tìm Reducer cập nhật state
// Bước 4: Tìm Selector đọc state trong component
```

---

## Phase 5 — Deep Dive

### Tìm nguồn gốc Observable / data

```bash
# Tìm nơi Subject/BehaviorSubject được .next() gọi
Ctrl+Shift+F → tìm "subject.next" hoặc "behaviorSubject.next"

# Tìm nơi data được emit
Ctrl+Shift+F → tìm tên method trong service
```

### IDE shortcuts (VS Code)

| Phím | Tác dụng |
|------|---------|
| `F12` | Go to Definition |
| `Shift+F12` | Find All References |
| `Alt+F12` | Peek Definition (xem ngay tại chỗ) |
| `Ctrl+Shift+F` | Search toàn bộ project |
| `Ctrl+P` | Tìm file nhanh |
| `Ctrl+T` | Tìm symbol (class, method, interface) |

### Thứ tự đọc lý tưởng — Ngày 1

```
main.ts / app.module.ts   (20 phút — toàn bộ modules, providers)
app-routing.module.ts     (15 phút — tất cả routes)
core/services/            (20 phút — singleton services quan trọng)
environments/             (5 phút  — API URLs, config)
```

---

[← C# .NET](./01a-csharp-dotnet.md) · [Index](./README.md) · [Go →](./01c-golang.md)
