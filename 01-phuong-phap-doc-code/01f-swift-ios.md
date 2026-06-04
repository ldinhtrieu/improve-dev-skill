# 01f · Phương Pháp Đọc Code — Swift iOS (UIKit / SwiftUI)

---

## Phase 1 — Entry Points & Cấu hình

**File đọc theo thứ tự:**

| File | Biết được gì |
|------|-------------|
| `Info.plist` | App permissions, URL schemes, capabilities |
| `*.xcconfig` / `Config.swift` | API URLs, keys theo environment |
| `AppDelegate.swift` | App lifecycle, global setup (Firebase, DI) |
| `SceneDelegate.swift` (iOS 13+) | Window/scene setup, initial ViewController |
| `Package.swift` / `Podfile` | Dependencies (SPM / CocoaPods) |

```swift
// AppDelegate.swift — global initialization
@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions options: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        // Đọc đây biết các services được khởi tạo
        FirebaseApp.configure()
        DIContainer.shared.setup()     // ← DI container
        AppearanceManager.configure()  // ← global UI styling
        return true
    }
}

// SceneDelegate.swift — màn hình đầu tiên
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, ...) {
        guard let windowScene = scene as? UIWindowScene else { return }
        window = UIWindow(windowScene: windowScene)
        // ← Đây là root ViewController — entry point của UI
        window?.rootViewController = UINavigationController(
            rootViewController: HomeViewController()
        )
        window?.makeKeyAndVisible()
    }
}
```

**Nếu dùng SwiftUI:**
```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()  // ← root view
                .environmentObject(AppViewModel())  // ← global state
        }
    }
}
```

---

## Phase 2 — Kiến trúc tổng thể

### Cấu trúc thư mục → nhận diện pattern

**MVVM (phổ biến với SwiftUI + Combine):**
```
MyApp/
├── App/
│   ├── AppDelegate.swift
│   └── SceneDelegate.swift
├── Features/
│   ├── Orders/
│   │   ├── Views/
│   │   │   ├── OrderListViewController.swift  (UIKit)
│   │   │   └── OrderListView.swift            (SwiftUI)
│   │   ├── ViewModels/
│   │   │   └── OrderListViewModel.swift
│   │   └── Models/
├── Services/
│   ├── Network/
│   │   ├── APIClient.swift
│   │   └── OrderAPI.swift
│   └── Storage/
│       └── CoreDataManager.swift
├── Domain/
│   ├── Models/
│   └── Repositories/
└── DI/
    └── DIContainer.swift
```

### Dependency Injection trong Swift

```swift
// Swift không có built-in DI — tìm các patterns này

// 1. Manual DI Container
class DIContainer {
    static let shared = DIContainer()

    lazy var orderRepository: OrderRepository = {
        OrderRepositoryImpl(apiClient: apiClient, storage: coreDataManager)
    }()

    lazy var orderViewModel: OrderListViewModel = {
        OrderListViewModel(repository: orderRepository)  // ← biết dependency
    }()
}

// 2. Swinject (thư viện phổ biến)
let container = Container()
container.register(OrderRepository.self) { _ in OrderRepositoryImpl() }
container.register(OrderListViewModel.self) { r in
    OrderListViewModel(repository: r.resolve(OrderRepository.self)!)
}

// 3. SwiftUI @EnvironmentObject / @StateObject
struct ContentView: View {
    @StateObject var viewModel = OrderListViewModel()
    // hoặc
    @EnvironmentObject var appState: AppState  // ← inject từ parent
}
```

---

## Phase 3 — Data Flow

### UIKit — ViewController lifecycle

```swift
// Thứ tự thực thi
viewDidLoad()       // ← setup UI, khởi tạo, ĐỌC ĐÂY để biết data load từ đâu
viewWillAppear()    // ← reload data nếu cần (khi quay lại từ màn hình khác)
viewDidAppear()     // ← animations, tracking
viewWillDisappear() // ← cleanup
viewDidDisappear()

class OrderListViewController: UIViewController {
    private var viewModel: OrderListViewModel!
    private var cancellables = Set<AnyCancellable>()

    override func viewDidLoad() {
        super.viewDidLoad()
        setupBindings()      // ← observe ViewModel
        viewModel.loadOrders()  // ← trigger data load
    }

    private func setupBindings() {
        // Combine publishers — đọc sink để biết data flow đến UI thế nào
        viewModel.$orders
            .receive(on: DispatchQueue.main)
            .sink { [weak self] orders in
                self?.tableView.reloadData()  // ← data đến đây
            }
            .store(in: &cancellables)
    }
}
```

### SwiftUI — declarative data flow

```swift
struct OrderListView: View {
    @StateObject var viewModel = OrderListViewModel()
    // hoặc @ObservedObject nếu VM được inject từ ngoài

    var body: some View {
        List(viewModel.orders) { order in
            OrderRowView(order: order)
        }
        .onAppear {
            viewModel.loadOrders()  // ← data load khi view xuất hiện
        }
        .alert(item: $viewModel.error) { error in
            Alert(title: Text(error.message))
        }
    }
}

// ViewModel với @Published — thay đổi tự động update UI
class OrderListViewModel: ObservableObject {
    @Published var orders: [Order] = []   // ← UI observe property này
    @Published var isLoading = false
    @Published var error: AppError?

    private let repository: OrderRepository

    func loadOrders() {
        isLoading = true
        Task {
            do {
                let result = try await repository.getOrders()
                await MainActor.run {  // ← phải update UI trên main thread
                    orders = result
                    isLoading = false
                }
            } catch {
                await MainActor.run { self.error = AppError(error) }
            }
        }
    }
}
```

### Network Layer — Alamofire / URLSession

```swift
// APIClient — tìm file này để biết base URL, headers, interceptors
class APIClient {
    static let shared = APIClient()
    private let session: Session  // Alamofire

    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        let response = await session.request(
            endpoint.url,
            method: endpoint.method,
            parameters: endpoint.parameters,
            headers: endpoint.headers  // ← auth header thêm ở đây hoặc interceptor
        ).serializingDecodable(T.self).response

        switch response.result {
        case .success(let value): return value
        case .failure(let error): throw APIError(error)
        }
    }
}

// Endpoint enum — tìm đây để biết tất cả API calls
enum OrderEndpoint: Endpoint {
    case list
    case detail(id: Int)
    case create(request: CreateOrderRequest)

    var path: String {
        switch self {
        case .list: return "/api/orders"
        case .detail(let id): return "/api/orders/\(id)"
        case .create: return "/api/orders"
        }
    }
}
```

---

## Phase 4 — Feature Trace

### Checklist trace 1 tính năng

- [ ] Tìm ViewController / View của màn hình
- [ ] Đọc `viewDidLoad()` / `onAppear` → biết setup và data load
- [ ] Tìm ViewModel được bind → đọc `@Published` properties
- [ ] Trace vào Repository / Service method
- [ ] Tìm `APIClient.request()` hoặc `URLSession.dataTask` → endpoint
- [ ] Tìm CoreData entity / UserDefaults → local storage

### Nguồn gốc data

| Thấy trong code | Nguồn gốc | Cách tìm |
|-----------------|-----------|----------|
| `viewModel.$xxx.sink` | ViewModel @Published | Tìm `@Published var xxx` trong VM |
| `@EnvironmentObject var x` | App-level state | Tìm `.environmentObject(x)` ở parent |
| `@AppStorage("key")` | UserDefaults | Search key string trong project |
| `segue.destination` | Storyboard segue | Mở Storyboard, tìm segue identifier |
| `navigationController?.pushViewController` | Code navigation | Cmd+Click để trace |

### Storyboard vs Code — cách tìm ViewController

```swift
// Nếu dùng Storyboard:
// Main.storyboard → tìm ViewController có Initial View Controller checkbox
// Hoặc tìm Storyboard ID trong attribute inspector → instantiateViewController(identifier:)

// Nếu dùng code (programmatic):
// SceneDelegate.swift → tìm rootViewController
// Tìm UINavigationController, UITabBarController để hiểu navigation structure
```

---

## Phase 5 — Deep Dive

### async/await và Combine — đọc async flow

```swift
// async/await — đọc thẳng từ trên xuống
func createOrder(address: String) async throws -> Order {
    let request = CreateOrderRequest(address: address)
    let orderDto = try await apiClient.request(.create(request))  // ← network call
    try await localCache.save(orderDto)                           // ← cache
    return orderDto.toDomain()
}

// Combine pipeline — đọc operators theo thứ tự
publisher
    .filter { $0.status == .active }
    .map { $0.toDomain() }
    .receive(on: DispatchQueue.main)    // ← chuyển về main thread
    .sink { self.orders = $0 }
    .store(in: &cancellables)
```

### IDE shortcuts — Xcode

| Phím | Tác dụng |
|------|---------|
| `Cmd+Click` | Jump to Definition |
| `Ctrl+1` → Find Call Hierarchy | Ai đang gọi hàm này |
| `Shift+Cmd+O` | Open Quickly (tìm file/class) |
| `Shift+Cmd+F` | Find in Project |
| `Ctrl+6` | Document items (methods trong file) |

### Thứ tự đọc lý tưởng — Ngày 1

```
AppDelegate / SceneDelegate    (15 phút — entry point, global setup)
Info.plist                     (5 phút  — permissions, capabilities)
DI Container / setup           (20 phút — dependency wiring)
APIClient + Endpoints          (15 phút — tất cả API calls)
CoreData Model (.xcdatamodel)  (10 phút — local schema)
```

---

[← Kotlin Android](./01e-kotlin-android.md) · [Index](./README.md) · [Objective-C →](./01g-objc-ios.md)
