# lifecycle_kmp — Public API

`lifecycle_kmp` là module Kotlin Multiplatform cung cấp điểm truy cập tập trung vào ngữ cảnh runtime của app chủ (host app): `Application` / `Activity` hiện tại trên Android (trên iOS là `UIViewController` thông qua typealias `AppHost`), cùng tiện ích theo dõi trạng thái mạng (`NetworkMonitor`). Module phụ thuộc `wayCore` ở mức `api` (nên type `AppHost` của `wayCore` được re-export cho consumer).

Trong kiến trúc WaySDK, `lifecycle_kmp` là api-dependency của `wayPay` và `wayAd`: các module đó dùng `LifecycleProvider` để lấy `Context`/`Activity` hiện hành mà không cần app truyền vào từng lời gọi.

## Khởi tạo

**Android — hoàn toàn tự động, không cần gọi gì.** Module đăng ký sẵn `androidx.startup.InitializationProvider` trong `AndroidManifest.xml` của thư viện với initializer `com.lifecycle_kmp.startup.AndroidProviderInitializer`. Khi app khởi động (trước `Application.onCreate` của app), initializer này gán `Application` cho `LifecycleProvider` và tự động `registerActivityLifecycleCallbacks` để theo dõi Activity hiện tại. Manifest merger sẽ gộp provider này vào app — app **không** được xoá `androidx.startup.InitializationProvider` khỏi manifest, nếu xoá thì `LifecycleProvider.application` sẽ là `null` và `requireApplication()` sẽ throw.

```kotlin
// Android: KHÔNG cần code khởi tạo. Chỉ cần dùng:
val app: Application? = LifecycleProvider.application
val activity: Activity? = LifecycleProvider.activity
```

Nếu app đã tắt auto-init của androidx.startup (dùng `tools:node="remove"` cho initializer này), có thể khởi tạo tay qua `AppInitializer`:

```kotlin
AppInitializer.getInstance(context)
    .initializeComponent(AndroidProviderInitializer::class.java)
```

Lưu ý: setter gán `Application` là `internal`, chỉ nhận đúng một lần (lần gán thứ hai sẽ throw `IllegalStateException` "`application` is already set."), nên con đường khởi tạo duy nhất từ phía app là qua androidx.startup.

**iOS — không có cơ chế khởi tạo.** Bản actual iOS hiện tại là stub: `appHost` luôn trả về `null`, không có API public nào để gán `UIViewController`. (Xem mục "Lưu ý platform".)

## Public API

### `LifecycleProvider` (commonMain, `expect object`)

Singleton giữ tham chiếu tới ngữ cảnh app chủ. Ở commonMain chỉ lộ ra một property đa nền tảng:

| Property | Kiểu | Ý nghĩa | Nullable khi nào |
|---|---|---|---|
| `appHost` | `AppHost?` | "Vật chủ" UI hiện tại của app. `AppHost` là typealias của `wayCore`: Android = `android.app.Activity`, iOS = `platform.UIKit.UIViewController` | Android: `null` khi chưa có Activity nào ở foreground (trước khi Activity đầu tiên created, hoặc sau khi Activity hiện tại paused/stopped/destroyed). iOS: **luôn** `null` |

### `LifecycleProvider` (androidMain, `actual object`)

Trên Android, `LifecycleProvider` đồng thời implement `Application.ActivityLifecycleCallbacks` và tự đăng ký vào `Application` ngay khi được khởi tạo bởi `AndroidProviderInitializer`. Activity hiện tại được giữ bằng `WeakReference` và cập nhật theo lifecycle:

- Gán ở `onActivityCreated` / `onActivityStarted` / `onActivityResumed`.
- Xoá (về `null`) ở `onActivityPaused` / `onActivityStopped` / `onActivityDestroyed` — chỉ khi Activity đó đúng là instance đang được giữ. Nghĩa là khi app xuống background, `activity`/`appHost` trở thành `null`.

| Property | Kiểu | Ý nghĩa | Nullable khi nào |
|---|---|---|---|
| `application` | `Application?` | `Application` của app chủ, gán một lần bởi androidx.startup | `null` nếu `InitializationProvider` bị xoá khỏi manifest / auto-init bị tắt mà không khởi tạo tay |
| `activity` | `Activity?` | Activity hiện tại ở foreground (weak reference) | `null` khi không có Activity nào giữa created→paused (ví dụ app ở background) |
| `context` | `Context?` | Ngữ cảnh tiện dụng: trả `application`, nếu `null` thì fallback sang `activity` | `null` khi cả `application` lẫn `activity` đều `null` |
| `appHost` | `AppHost?` (= `Activity?`) | Chính là `activity` hiện tại | Như `activity` |

#### `fun LifecycleProvider.requireApplication(): Application` *(extension, chỉ Android)*

Trả về `application`, hoặc **throw `IllegalStateException`** nếu chưa được khởi tạo. Message lỗi hướng dẫn kiểm tra lại `androidx.startup.InitializationProvider` trong `AndroidManifest.xml` (kèm ví dụ cách remove initializer cụ thể bằng `tools:node="remove"` mà vẫn giữ provider).

**Trả về:** `Application` non-null của app chủ.

#### `fun LifecycleProvider.requireActivity(): Activity` *(extension, chỉ Android)*

Trả về `activity`, hoặc **throw `IllegalStateException`** với message `"There's no current Activity."` nếu không có Activity foreground.

**Trả về:** `Activity` non-null đang ở foreground.

### `AndroidProviderInitializer` (androidMain)

`androidx.startup.Initializer<Unit>` thực hiện việc gán `Application` cho `LifecycleProvider`. Bình thường app không đụng tới class này — nó chạy tự động; chỉ cần dùng trực tiếp khi app tắt auto-init của androidx.startup.

#### `fun create(context: Context)`

| Param | Kiểu | Mô tả |
|---|---|---|
| `context` | `Context` | Context do androidx.startup cấp; hàm lấy `context.applicationContext` (ép kiểu `Application`) gán cho `LifecycleProvider` |

**Trả về:** `Unit`. Sau lời gọi này `LifecycleProvider.application` non-null và lifecycle callbacks đã được đăng ký.

#### `fun dependencies(): List<Class<out Initializer<*>>>`

**Trả về:** danh sách rỗng — initializer không phụ thuộc initializer nào khác.

### `NetworkMonitor` (commonMain, `expect object`)

Singleton kiểm tra và quan sát trạng thái kết nối mạng.

#### `fun isNetworkAvailable(): Boolean`

**Trả về:** `true` nếu đang có kết nối mạng dùng được.

- **Android** (yêu cầu permission `ACCESS_NETWORK_STATE`): kiểm tra `activeNetwork` qua `ConnectivityManager`; chỉ trả `true` khi network có cả capability `NET_CAPABILITY_INTERNET` **và** `NET_CAPABILITY_VALIDATED` (tức mạng đã được hệ thống xác thực là thực sự ra Internet, không chỉ "có Wi-Fi"). **Lưu ý:** hàm lấy `ConnectivityManager` qua `LifecycleProvider.requireApplication()` — sẽ **throw `IllegalStateException`** nếu `LifecycleProvider` chưa được khởi tạo.
- **iOS**: đọc giá trị mới nhất từ `NWPathMonitor` (Network.framework). Giá trị khởi đầu mặc định là `Connected` (lạc quan) cho tới khi monitor phát update đầu tiên.

#### `fun observeNetworkState(): Flow<NetworkState>`

**Trả về:** `Flow<NetworkState>` phát trạng thái mạng theo thời gian thực.

- **Android** (yêu cầu permission `ACCESS_NETWORK_STATE`): `callbackFlow` đăng ký `NetworkCallback` cho các transport WIFI / CELLULAR / ETHERNET; phát trạng thái ban đầu ngay khi collect, sau đó phát theo `onCapabilitiesChanged` (dựa trên INTERNET + VALIDATED) và `onLost`. Flow đã áp `distinctUntilChanged`; callback được tự động unregister khi ngừng collect. Cũng throw `IllegalStateException` nếu `LifecycleProvider` chưa khởi tạo.
- **iOS**: trả về `StateFlow` (read-only) được cập nhật bởi `NWPathMonitor` chạy trên dispatch queue riêng; monitor khởi động lazy lần đầu object được chạm tới và chạy suốt vòng đời process.

## Public models

### `NetworkState` (sealed class, commonMain)

Trạng thái kết nối mạng, dùng làm phần tử của `observeNetworkState()`.

| Thành phần | Kiểu | Ý nghĩa |
|---|---|---|
| `NetworkState.Connected` | `object` | Đang có kết nối mạng |
| `NetworkState.Disconnected` | `object` | Mất kết nối mạng |
| `isConnected` | `Boolean` (property) | `true` khi instance là `Connected` |

## Lưu ý platform

- **Android**: khởi tạo hoàn toàn tự động qua androidx.startup; API đầy đủ (`application`, `activity`, `context`, `appHost`, `requireApplication()`, `requireActivity()`). `activity`/`appHost` là weak reference và bị xoá ngay khi Activity paused — khi app ở background chúng là `null`; đừng cache kết quả, hãy đọc tại thời điểm cần dùng. `NetworkMonitor` cần permission `ACCESS_NETWORK_STATE` trong manifest của app và phụ thuộc `LifecycleProvider` đã khởi tạo (nếu không sẽ throw `IllegalStateException`, không trả `null`).
- **iOS**: `LifecycleProvider` hiện là **stub** — `appHost` luôn trả về `null` và không có API public nào để gán `UIViewController` cho SDK; các module dùng `appHost` trên iOS phải tự xử lý trường hợp `null`. Ngược lại, `NetworkMonitor` trên iOS hoạt động đầy đủ (Network.framework/NWPathMonitor), không cần permission, và không phụ thuộc `LifecycleProvider`; trạng thái ban đầu trước update đầu tiên của monitor là `Connected`.
- Hành vi "chưa init": các property nullable (`application`, `activity`, `context`, `appHost`) trả `null`; chỉ hai extension `require*()` và các hàm `NetworkMonitor` phía Android là throw.

## Log TAG

Module log với TAG `WayLifecycle/NetworkMonitor`. Toàn bộ quy ước TAG của WaySDK: xem `wayCore/API.md`.
