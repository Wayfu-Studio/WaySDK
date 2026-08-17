# adjust_kmp — Public API

`adjust_kmp` là module Kotlin Multiplatform đóng vai trò **lớp nền Adjust** cho toàn bộ WaySDK: nó không tự wrap lại API của Adjust SDK thành facade riêng, mà (1) cung cấp helper chuyển đổi tiền tệ dùng chung khi track revenue, và (2) là **chủ sở hữu duy nhất của cinterop bridge `cocoapods.Adjust`** trên iOS — mọi module khác (`wayInstall`, `wayAd`, `wayPay`) đều `api(project(":adjust_kmp"))` để dùng trực tiếp các type Adjust gốc trên từng platform.

Cụ thể, public surface của module gồm 3 phần:

- **Kotlin chung (`commonMain`)**: `setupAdjust(...)` — hàm `expect/actual` khởi tạo Adjust SDK cho cả hai platform; và `AdjustMoney.kt` — helper quy đổi số tiền dạng micros về đơn vị tiền tệ mà Adjust yêu cầu khi set revenue.
- **iOS**: `com.adjust_kmp.att.AttPermissionRequester` / `AttState` — xin quyền ATT (IDFA) trước khi init Adjust; và package cinterop `cocoapods.Adjust` (generate từ `src/nativeInterop/cinterop/Adjust.def`, module Objective-C `AdjustSdk`), expose nguyên bộ API của Adjust iOS SDK: `Adjust`, `ADJConfig`, `ADJEvent`, `ADJAdRevenue`, `ADJAttribution`, các hằng `ADJEnvironmentProduction` / `ADJEnvironmentSandbox`, `ADJLogLevelVerbose`, ...
- **Android**: re-export (`api`) hai thư viện `com.adjust.sdk:adjust-android` và `com.adjust.sdk:adjust-android-webbridge` (v5.8.0) — consumer dùng thẳng `com.adjust.sdk.*` (`Adjust`, `AdjustConfig`, `AdjustEvent`, `AdjustAdRevenue`, ...).

## Khởi tạo

Dùng `setupAdjust(...)` từ `commonMain` — không cần chạm vào type platform:

```kotlin
import com.adjust_kmp.setupAdjust

setupAdjust(tokenAdjust = token, isDebug = isDebug) { isSuccess ->
    // Adjust đã init xong
}
```

### `fun setupAdjust(tokenAdjust: String, isDebug: Boolean, onComplete: (isSuccess: Boolean) -> Unit)`

| Param | Kiểu | Mô tả |
|---|---|---|
| `tokenAdjust` | `String` | App token lấy từ dashboard Adjust. |
| `isDebug` | `Boolean` | `true` → environment sandbox, `false` → environment production. |
| `onComplete` | `(Boolean) -> Unit` | Android: luôn `true` sau khi init. iOS: `isSuccess` của trạng thái ATT (`true` khi `AUTHORIZED`). |

Cả hai platform đều bật `enableCostDataInAttribution()` và log level `VERBOSE`.

- **Android** (`SetupAdjust.android.kt`): lấy `Context` từ `LifecycleProvider.context`; nếu null → log lỗi và thoát, **không** gọi `onComplete`.
- **iOS** (`SetupAdjust.ios.kt`): tạo `ADJConfig(appToken, environment)`, **tự xin quyền ATT** qua [`AttPermissionRequester`](#att-ios-package-comadjust_kmpatt) — nếu trạng thái còn `NOT_DETERMINED` thì hiện dialog ATT rồi mới `Adjust.initSdk`, ngược lại init luôn. Caller không cần làm gì thêm.

### ATT (iOS): package `com.adjust_kmp.att`

`object AttPermissionRequester` — xin quyền App Tracking Transparency (IDFA), chỉ có trên `iosMain`.

| API | Mô tả |
|---|---|
| `fun getCurrentStatus(): AttState` | Trạng thái ATT hiện tại. Máy iOS < 14.5 → `NOT_APPLICABLE`. |
| `fun requestPermission(onCompletion: (AttState) -> Unit)` | Hiện dialog ATT sau delay 1s trên main queue; iOS < 14.5 → callback ngay với `NOT_APPLICABLE`. |

`enum class AttState`: `NOT_DETERMINED`, `DENIED`, `AUTHORIZED`, `RESTRICTED`, `NOT_APPLICABLE`; property `isSuccess` = `true` khi `AUTHORIZED`.

`setupAdjust` đã gọi sẵn hai API này, chỉ dùng trực tiếp khi cần đọc / xin quyền ATT ngoài luồng init Adjust.

> App phải khai báo `NSUserTrackingUsageDescription` trong `Info.plist`, nếu không dialog ATT sẽ không hiện.
>
> Trước đây hai type này nằm ở `com.wayad.common.utils.att` trong module `wayAd` — đã chuyển hẳn sang đây, import cũ không còn.

Track revenue dùng helper của module (chung cho cả hai platform):

```kotlin
import com.adjust_kmp.adjustRevenueFromMicros

event.setRevenue(amountMicros.adjustRevenueFromMicros(), currency)
```

## Public API

### `AdjustMoney.kt` (top-level, package `com.adjust_kmp`)

Helper quy đổi tiền tệ khi báo revenue cho Adjust. Các API billing (Google Play Billing `priceAmountMicros`, StoreKit sau khi nhân hệ số) trả về số tiền dạng **micros** (1 đơn vị tiền = 1.000.000 micros), trong khi `setRevenue` của Adjust nhận `Double` theo đơn vị tiền tệ.

| Property | Kiểu | Mô tả |
|---|---|---|
| `MICROS_PER_UNIT` | `const Long` | Hằng số quy đổi: `1_000_000` micros = 1 đơn vị tiền tệ. |

#### `fun Long.adjustRevenueFromMicros(): Double`

Quy đổi số tiền dạng micros (receiver `Long`) về đơn vị tiền tệ thực để truyền vào `setRevenue` của Adjust.

| Param | Kiểu | Mô tả |
|---|---|---|
| *(receiver)* | `Long` | Số tiền dạng micros (ví dụ `priceAmountMicros` từ Play Billing). |

**Trả về:** `Double` — kết quả của phép chia `this / 1_000_000.0` (ví dụ `4_990_000L` → `4.99`). Không làm tròn, không kiểm tra giá trị âm.

### Bridge iOS: package `cocoapods.Adjust`

Đây là API "mượn" — module không định nghĩa type nào, nhưng là nơi duy nhất trong WaySDK khai báo cinterop nên mọi import `cocoapods.Adjust.*` ở các module khác đều đi qua module này. Các type Adjust iOS SDK 5.8.0 mà WaySDK đang dùng qua bridge:

| Symbol | Vai trò |
|---|---|
| `Adjust` | Class tĩnh chính: `initSdk`, `trackEvent`, `trackAdRevenue`, đọc attribution, ... |
| `ADJConfig` | Config khởi tạo (app token, environment, log level). |
| `ADJEvent` | Event track thường / kèm revenue (`setRevenue`). |
| `ADJAdRevenue` | Track doanh thu quảng cáo (ad revenue). |
| `ADJAttribution` | Dữ liệu attribution trả về từ Adjust. |
| `ADJEnvironmentProduction`, `ADJEnvironmentSandbox`, `ADJLogLevelVerbose` | Hằng cấu hình môi trường / log. |

### Re-export Android: `com.adjust.sdk.*`

`androidMain` khai báo `api(adjust-android)` và `api(adjust-android-webbridge)` (v5.8.0), nên consumer của `adjust_kmp` dùng trực tiếp Adjust Android SDK (`Adjust`, `AdjustConfig`, `AdjustEvent`, `AdjustAdRevenue`, webbridge cho WebView) mà không cần tự khai báo dependency.

## Public models

Chỉ có `AttState` (enum, iOS only — xem phần ATT ở trên). Mọi model còn lại (event, config, attribution, ad revenue) là type gốc của Adjust SDK trên từng platform, được expose transitively như mô tả ở trên.

## Lưu ý platform

- **iOS — cinterop trực tiếp qua XCFramework (không dùng CocoaPods plugin):**
  - `Adjust.def`: `language = Objective-C`, `modules = AdjustSdk`, `package = cocoapods.Adjust` (giữ tên package `cocoapods.*` để tương thích code cũ, nhưng cơ chế là cinterop trực tiếp).
  - Cinterop compile với `-fmodules -F<slice>` trỏ vào `native-frameworks/AdjustSdk-<version>/AdjustSdk.xcframework`, slice `ios-arm64` cho `iosArm64` và `ios-arm64_x86_64-simulator` cho `iosSimulatorArm64`. XCFramework được task `:downloadInteropFrameworks` (root project) tải từ GitHub release `adjust/ios_sdk` (bản **Dynamic**) rồi bày ra layout cố định `<name>-<version>/<name>.xcframework`; mọi task `cinterop*` phụ thuộc task này. Version lấy từ `iosAdjustSdk` trong `gradle/libs.versions.toml` — không hardcode trong module.
  - Cinterop chỉ cung cấp header lúc compile — **app iOS cuối phải tự link binary `AdjustSdk.xcframework` (qua SPM)**. Root project có task `verifyInteropVersions` đối chiếu version trong `Package.resolved` của `iosApp` (identity `ios_sdk` → `AdjustSdk`) với version cinterop để bảo đảm hai bên không lệch nhau.
  - Framework Kotlin output: `adjust_kmpKit`, static. Target hỗ trợ: `iosArm64`, `iosSimulatorArm64` (không có x86_64 riêng ngoài slice simulator).
- **Android:** minSdk 24, compileSdk 36, namespace `com.adjust_kmp`. Adjust Android SDK v5.8.0 được expose dạng `api` nên tự động có mặt trong classpath của app.
- **Version hai platform:** cùng ở 5.8.0 (iOS pin qua `iosAdjustSdk`, Android qua `adjustAndroid`). Giữ hai số này bằng nhau khi bump — API bề mặt của Adjust không hoàn toàn đối xứng giữa hai platform, lệch minor là nguồn gốc của những khác biệt hành vi khó truy.
- Publish lên GitHub Packages với coordinates `way-sdk:adjust_kmp:<way-sdk-version>`.
