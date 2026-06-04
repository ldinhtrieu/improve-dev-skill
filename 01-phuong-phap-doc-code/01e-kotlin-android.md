# 01e · Phương Pháp Đọc Code — Kotlin Android (Jetpack + MVVM)

---

## Phase 1 — Entry Points & Cấu hình

**File đọc theo thứ tự:**

| File | Biết được gì |
|------|-------------|
| `app/build.gradle.kts` | Dependencies, SDK versions, build variants |
| `AndroidManifest.xml` | Activities, permissions, entry points, deep links |
| `Application.kt` (class kế thừa `Application`) | Global init: DI, logging, crash tracking |
| `gradle.properties` / `local.properties` | API keys, build config |

```xml
<!-- AndroidManifest.xml — đọc để biết app có gì -->
<manifest>
    <!-- Permissions — biết app cần quyền gì -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.CAMERA" />

    <application
        android:name=".MyApplication">  <!-- ← Application class -->

        <!-- LAUNCHER activity = màn hình đầu tiên khi mở app -->
        <activity android:name=".ui.MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- Deep links -->
        <activity android:name=".ui.OrderDetailActivity">
            <intent-filter android:autoVerify="true">
                <data android:scheme="https" android:host="app.example.com"
                      android:pathPrefix="/orders" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

```kotlin
// Application.kt — global initialization
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // Đọc đây biết các thư viện được khởi tạo
        startKoin { modules(appModule, networkModule, repositoryModule) }  // Koin DI
        Timber.plant(Timber.DebugTree())      // logging
        FirebaseCrashlytics.getInstance()     // crash tracking
    }
}
```

---

## Phase 2 — Kiến trúc tổng thể

### Cấu trúc thư mục → nhận diện pattern

**MVVM + Clean Architecture (phổ biến nhất):**
```
app/src/main/java/com/example/
├── di/                     → Dependency Injection modules
│   ├── AppModule.kt
│   ├── NetworkModule.kt
│   └── RepositoryModule.kt
├── data/
│   ├── remote/             → API calls (Retrofit)
│   │   ├── ApiService.kt
│   │   └── dto/
│   ├── local/              → Room database
│   │   ├── AppDatabase.kt
│   │   ├── dao/
│   │   └── entity/
│   └── repository/         → Repository implementations
├── domain/
│   ├── model/              → Domain models (pure Kotlin)
│   ├── repository/         → Repository interfaces
│   └── usecase/            → Business logic (Use Cases)
└── ui/
    ├── MainActivity.kt
    ├── orders/
    │   ├── OrderListFragment.kt
    │   ├── OrderListViewModel.kt
    │   └── adapter/
    └── common/
```

### DI Framework — Hilt (Dagger) hoặc Koin

```kotlin
// Hilt — tìm @Module để biết dependencies
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit =
        Retrofit.Builder()
            .baseUrl(BuildConfig.API_BASE_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .build()

    @Provides
    fun provideOrderApi(retrofit: Retrofit): OrderApi =
        retrofit.create(OrderApi::class.java)
}

// Koin — tìm module { } blocks
val repositoryModule = module {
    single { OrderRepositoryImpl(get(), get()) as OrderRepository }
    factory { GetOrdersUseCase(get()) }
}
```

### MVVM Data Flow

```
View (Fragment/Activity)
    ↓ observes LiveData / StateFlow
ViewModel
    ↓ calls UseCase / Repository
UseCase (optional — business logic)
    ↓ calls Repository interface
Repository
    ├── Remote: Retrofit API call
    └── Local: Room DAO
```

---

## Phase 3 — Data Flow

### Màn hình mới mở — lifecycle

```kotlin
// Fragment lifecycle — thứ tự thực thi
onAttach()
onCreate()           ← init ViewModel ở đây
onCreateView()       ← inflate layout
onViewCreated()      ← setup UI, observe LiveData/StateFlow ← ĐỌC ĐÂY
onStart()
onResume()           ← màn hình hiển thị
...
onDestroyView()      ← clean up binding
```

```kotlin
// onViewCreated — nơi data flow bắt đầu
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)

    // Observe ViewModel state — đây là nguồn data chính
    viewLifecycleOwner.lifecycleScope.launch {
        viewModel.uiState.collect { state ->  // ← StateFlow
            when (state) {
                is UiState.Loading -> showLoading()
                is UiState.Success -> showOrders(state.orders)  // ← data đến đây
                is UiState.Error -> showError(state.message)
            }
        }
    }

    // Observe events (one-time)
    viewModel.navigateToDetail.observe(viewLifecycleOwner) { orderId ->
        findNavController().navigate(R.id.orderDetailFragment,
            bundleOf("orderId" to orderId))
    }
}
```

### Trace từ UI action xuống API

```kotlin
// 1. User click → ViewModel method
binding.btnCreateOrder.setOnClickListener {
    viewModel.createOrder(binding.etAddress.text.toString())
}

// 2. ViewModel → UseCase / Repository
class OrderViewModel(
    private val createOrderUseCase: CreateOrderUseCase  // ← inject
) : ViewModel() {
    fun createOrder(address: String) {
        viewModelScope.launch {  // ← coroutine scope
            _uiState.value = UiState.Loading
            createOrderUseCase(address)  // ← nhảy vào đây
                .onSuccess { order -> _uiState.value = UiState.Success(order) }
                .onFailure { e -> _uiState.value = UiState.Error(e.message) }
        }
    }
}

// 3. UseCase → Repository
class CreateOrderUseCase(private val repo: OrderRepository) {
    suspend operator fun invoke(address: String): Result<Order> =
        repo.createOrder(CreateOrderRequest(address))  // ← nhảy vào repo
}

// 4. Repository → Retrofit API call
class OrderRepositoryImpl(
    private val api: OrderApi,
    private val dao: OrderDao
) : OrderRepository {
    override suspend fun createOrder(request: CreateOrderRequest): Result<Order> =
        runCatching {
            val response = api.createOrder(request)  // ← Retrofit call
            response.toDomain()
        }
}

// 5. Retrofit API interface
interface OrderApi {
    @POST("api/orders")
    suspend fun createOrder(@Body request: CreateOrderRequest): OrderDto
}
```

---

## Phase 4 — Feature Trace

### Checklist trace 1 tính năng

- [ ] Tìm Fragment/Activity của màn hình (tên thường khớp với UI)
- [ ] Đọc `onViewCreated()` → thấy observe gì, setup gì
- [ ] Tìm ViewModel tương ứng → xem `uiState` / `LiveData` được tính thế nào
- [ ] Trace vào UseCase hoặc Repository
- [ ] Tìm Retrofit `@GET/@POST` interface → biết API endpoint
- [ ] Tìm Room `@Dao` → biết local cache query

### Nguồn gốc data trong Fragment/Activity

| Thấy trong code | Nguồn gốc | Cách tìm |
|-----------------|-----------|----------|
| `viewModel.xxx.observe(...)` | ViewModel LiveData | Tìm `_xxx` MutableLiveData trong VM |
| `viewModel.uiState.collect` | ViewModel StateFlow | Tìm `_uiState` trong VM |
| `arguments?.getLong("id")` | Bundle từ navigation | Tìm `navigate(... bundleOf(...))` |
| `requireContext()`, `resources` | Android system | Built-in, không cần trace |
| `hiltViewModel<OrderVM>()` | Hilt DI | Tìm constructor của ViewModel |

### Navigation — tìm màn hình từ flow

```kotlin
// Jetpack Navigation — tìm nav_graph XML
// res/navigation/nav_graph.xml → thấy toàn bộ màn hình và luồng điều hướng

// Code navigate
findNavController().navigate(
    R.id.action_orderList_to_orderDetail,
    bundleOf("orderId" to id)
)
// → Tìm "action_orderList_to_orderDetail" trong nav_graph.xml
```

---

## Phase 5 — Deep Dive

### Coroutines — hiểu async flow

```kotlin
// Thấy suspend fun → hàm này phải gọi từ coroutine scope
// Thấy viewModelScope.launch → chạy trong ViewModel lifecycle
// Thấy withContext(Dispatchers.IO) → chuyển sang background thread

// Flow operators — đọc như pipeline
repository.getOrders()          // Flow<List<Order>>
    .filter { it.isNotEmpty() } // lọc
    .map { orders ->            // transform
        orders.sortedBy { it.createdAt }
    }
    .catch { e ->               // error handling
        emit(emptyList())
    }
    .collect { orders ->        // nhận kết quả cuối
        _uiState.value = UiState.Success(orders)
    }
```

### Room Database — đọc schema

```kotlin
// AppDatabase.kt — bản đồ local database
@Database(
    entities = [OrderEntity::class, ProductEntity::class],  // ← danh sách bảng
    version = 5  // ← version hiện tại, xem migrations
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun orderDao(): OrderDao
    abstract fun productDao(): ProductDao
}

// Migration — lịch sử thay đổi schema
val MIGRATION_4_5 = object : Migration(4, 5) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE orders ADD COLUMN notes TEXT")
    }
}
```

### Thứ tự đọc lý tưởng — Ngày 1

```
AndroidManifest.xml      (15 phút — activities, permissions, entry points)
Application.kt           (10 phút — global init, DI setup)
di/ modules              (20 phút — tất cả dependencies)
AppDatabase.kt + DAO     (15 phút — local schema)
ApiService interfaces    (10 phút — tất cả API endpoints)
```

---

[← Java Spring](./01d-java-spring.md) · [Index](./README.md) · [Swift iOS →](./01f-swift-ios.md)
