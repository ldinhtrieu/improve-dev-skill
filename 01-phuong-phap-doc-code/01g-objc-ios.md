# 01g · Phương Pháp Đọc Code — Objective-C iOS (UIKit + MVC Legacy)

---

## Phase 1 — Entry Points & Cấu hình

**File đọc theo thứ tự:**

| File | Biết được gì |
|------|-------------|
| `Info.plist` | App permissions, URL schemes, bundle ID |
| `AppDelegate.m` | Global setup, 3rd party init, root ViewController |
| `Podfile` / `Podfile.lock` | Dependencies (CocoaPods) |
| `*.xcconfig` | Build config theo environment (Debug/Release/Staging) |
| `Constants.h` / `Config.m` | API URLs, keys, constants |

```objc
// AppDelegate.m — điểm khởi động, đọc didFinishLaunchingWithOptions
- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {

    // Đọc đây biết hệ thống khởi tạo gì
    [FIRApp configure];                          // Firebase
    [[AFNetworkReachabilityManager sharedManager] startMonitoring]; // Network
    [self setupRootViewController];              // ← Root VC

    return YES;
}

- (void)setupRootViewController {
    // Đây là màn hình đầu tiên
    MainTabBarController *tabBar = [[MainTabBarController alloc] init];
    self.window.rootViewController = tabBar;
    [self.window makeKeyAndVisible];
}
```

> 💡 **Tip cho codebase Objective-C cũ:** Tìm file `Constants.h` hoặc `AppConfig.h` — thường chứa toàn bộ API URLs, timeout values, và feature flags của hệ thống.

---

## Phase 2 — Kiến trúc tổng thể

### Cấu trúc thư mục — MVC (pattern mặc định)

```
MyApp/
├── AppDelegate.h / .m
├── Controllers/            → ViewControllers (View + Controller gộp lại)
│   ├── OrderListViewController.h / .m
│   └── OrderDetailViewController.h / .m
├── Models/                 → Data models, business logic
│   ├── Order.h / .m
│   └── OrderManager.h / .m  (Singleton service thường gặp)
├── Views/                  → Custom UIView, XIB files
├── Networking/             → API layer
│   ├── APIClient.h / .m
│   └── OrderAPI.h / .m
├── Helpers/ / Utils/       → Utility classes
├── Categories/             → Objective-C Categories (mở rộng class)
│   └── NSString+Validation.h
└── Resources/
    ├── Main.storyboard
    └── Assets.xcassets
```

### Đọc file `.h` trước — interface là tài liệu

```objc
// OrderManager.h — đọc file header để hiểu class làm gì
// KHÔNG cần đọc .m trước — .h là public API

@interface OrderManager : NSObject

// Singleton pattern — rất phổ biến trong Obj-C codebase cũ
+ (instancetype)sharedManager;

// Methods — đọc tên + params để hiểu chức năng
- (void)fetchOrdersWithCompletion:(void (^)(NSArray<Order *> *orders,
                                           NSError *error))completion;
- (void)createOrder:(OrderRequest *)request
         completion:(void (^)(Order *order, NSError *error))completion;
- (void)cancelOrderWithID:(NSString *)orderID
               completion:(void (^)(BOOL success, NSError *error))completion;

// Properties — biết state của object
@property (nonatomic, readonly) NSArray<Order *> *cachedOrders;
@property (nonatomic, assign) NSInteger pageSize;

@end
```

### Singleton — pattern cực phổ biến trong Obj-C

```objc
// Khi thấy [XxxManager sharedManager] hoặc [XxxService sharedInstance]
// → đó là Singleton, có 1 instance duy nhất trong app

// Cách nhận diện Singleton trong .m file:
+ (instancetype)sharedManager {
    static OrderManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

// Các Singletons hay gặp:
[OrderManager sharedManager]
[NetworkClient sharedClient]
[UserSession currentSession]    // ← current user info
[DatabaseManager sharedDB]
```

---

## Phase 3 — Data Flow

### UIViewController lifecycle

```
viewDidLoad          ← setup IBOutlets, delegates, initial data load — ĐỌC ĐÂY
viewWillAppear:      ← refresh data khi quay lại màn hình
viewDidAppear:       ← animations, tracking
viewWillDisappear:   ← save state
viewDidDisappear:    ← cleanup, cancel requests
```

```objc
// Ví dụ trace data flow từ viewDidLoad
- (void)viewDidLoad {
    [super viewDidLoad];

    // Setup delegates — tìm protocol implementations
    self.tableView.delegate = self;
    self.tableView.dataSource = self;

    // Data load — theo block completion
    [[OrderManager sharedManager] fetchOrdersWithCompletion:^(NSArray *orders, NSError *error) {
        if (error) {
            [self showErrorAlert:error];
            return;
        }
        // ← data đến đây, luôn phải dispatch về main thread
        dispatch_async(dispatch_get_main_queue(), ^{
            self.orders = orders;              // ← gán data
            [self.tableView reloadData];       // ← update UI
        });
    }];
}
```

### Networking — AFNetworking / NSURLSession

```objc
// APIClient.m — tìm baseURL và common headers
- (void)GET:(NSString *)path
 parameters:(NSDictionary *)params
    success:(void (^)(NSURLSessionDataTask *task, id responseObject))success
    failure:(void (^)(NSURLSessionDataTask *task, NSError *error))failure {

    NSString *fullURL = [self.baseURL stringByAppendingString:path];
    [self.sessionManager GET:fullURL
                  parameters:params
                    progress:nil
                     success:success
                     failure:failure];
}

// Cách gọi API trong Manager
- (void)fetchOrdersWithCompletion:(void (^)(NSArray *, NSError *))completion {
    [[APIClient sharedClient] GET:@"/api/orders"
                       parameters:nil
                          success:^(NSURLSessionDataTask *task, id response) {
        // Parse response
        NSArray *ordersData = response[@"data"];
        NSArray *orders = [ordersData bk_map:^id(NSDictionary *dict) {
            return [Order orderFromDictionary:dict];  // ← model parsing
        }];
        completion(orders, nil);
    } failure:^(NSURLSessionDataTask *task, NSError *error) {
        completion(nil, error);
    }];
}
```

### Delegation pattern — rất phổ biến trong Obj-C

```objc
// Khi thấy xxx.delegate = self → tìm protocol
// Ctrl+Click trên protocol name → xem methods cần implement

// Ví dụ: UITableView
// 1. Tìm tableView.delegate = self và tableView.dataSource = self
// 2. Tìm các method implement protocol:
- (NSInteger)tableView:(UITableView *)tableView
 numberOfRowsInSection:(NSInteger)section {
    return self.orders.count;  // ← data source ở đây
}

- (UITableViewCell *)tableView:(UITableView *)tableView
         cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    Order *order = self.orders[indexPath.row];  // ← data lấy từ đây
    // ...
}

- (void)tableView:(UITableView *)tableView
didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    // ← user tap action
}
```

---

## Phase 4 — Feature Trace

### Checklist trace 1 tính năng

- [ ] Tìm ViewController tương ứng (tên file thường rõ ràng)
- [ ] Đọc file `.h` trước → hiểu public interface
- [ ] Đọc `viewDidLoad` trong `.m` → biết data load từ đâu
- [ ] Tìm Singleton Manager được gọi (`[XxxManager sharedManager]`)
- [ ] Trace vào Manager → tìm API call
- [ ] Tìm `[Model modelFromDictionary:]` → biết JSON mapping

### Tìm navigation / màn hình tiếp theo

```objc
// Push (NavigationController)
OrderDetailViewController *detailVC = [[OrderDetailViewController alloc] initWithOrderID:orderId];
[self.navigationController pushViewController:detailVC animated:YES];

// Modal
[self presentViewController:detailVC animated:YES completion:nil];

// Storyboard segue
[self performSegueWithIdentifier:@"showOrderDetail" sender:order];
// → Tìm prepareForSegue: để biết data được truyền thế nào

- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender {
    if ([segue.identifier isEqualToString:@"showOrderDetail"]) {
        OrderDetailViewController *destVC = segue.destinationViewController;
        destVC.order = self.selectedOrder;  // ← data truyền sang màn hình mới
    }
}
```

### Nguồn gốc data trong ViewController

| Thấy trong code | Nguồn gốc | Cách tìm |
|-----------------|-----------|----------|
| `[XxxManager sharedManager]` | Singleton | Tìm class XxxManager |
| `self.xxx = sender` trong `initWithXxx:` | Init param | Tìm nơi `alloc] initWithXxx:` |
| `segue.destinationViewController` | Storyboard segue | Tìm `prepareForSegue:` |
| `[[NSUserDefaults standardUserDefaults] objectForKey:]` | UserDefaults | Search key string |
| `[NSNotificationCenter defaultCenter] addObserver:` | Notification | Tìm `postNotificationName:` |

---

## Phase 5 — Deep Dive

### Block (closure) — đọc async Obj-C

```objc
// Block là callback function — đọc từ ngoài vào trong
[service fetchData:params completion:^(NSArray *data, NSError *error) {
    // ← kết quả đến đây
    // Chú ý: thường phải dispatch_async main thread để update UI
    dispatch_async(dispatch_get_main_queue(), ^{
        self.tableData = data;
        [self.tableView reloadData];
    });
}];

// Thấy __weak typeof(self) weakSelf = self → tránh retain cycle
// Bên trong block phải dùng weakSelf thay vì self
__weak typeof(self) weakSelf = self;
[manager doSomethingWithCompletion:^{
    [weakSelf updateUI];  // ← không bị retain cycle
}];
```

### Categories — mở rộng class có sẵn

```objc
// NSString+Validation.h → NSString thêm methods validate
// UIColor+Theme.h → UIColor thêm custom colors
// UIViewController+Alert.h → thêm helper method show alert

// Cách tìm: Ctrl+Shift+F → tìm "category" tên class
// Khi thấy [NSString xxx] mà không có trong docs → tìm Category file
```

### IDE shortcuts — Xcode

| Phím | Tác dụng |
|------|---------|
| `Cmd+Click` | Jump to Definition (qua `.h` trước) |
| `Ctrl+Cmd+↑/↓` | Toggle giữa `.h` và `.m` |
| `Shift+Cmd+O` | Open Quickly — tìm file nhanh |
| `Shift+Cmd+F` | Find in Project |
| `Ctrl+6` | Methods list trong file hiện tại |

### Đọc `.h` vs `.m` — khi nào đọc cái nào

```
.h (header) → đọc trước:
  - Biết class có properties và methods gì
  - Biết protocols implement
  - Biết public interface

.m (implementation) → đọc sau:
  - Hiểu logic thực tế
  - Tìm private methods / properties
  - Đọc #pragma mark để nhảy đến section
```

### Thứ tự đọc lý tưởng — Ngày 1

```
AppDelegate.m              (15 phút — entry point, global setup)
Constants.h / Config.h     (10 phút — API URLs, keys, enums)
APIClient.h + .m           (20 phút — network layer)
Core Manager .h files      (20 phút — đọc headers của Singletons quan trọng)
Main.storyboard            (10 phút — navigation structure tổng thể)
```

---

[← Swift iOS](./01f-swift-ios.md) · [Index](./README.md)
