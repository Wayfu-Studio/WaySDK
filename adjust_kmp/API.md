# adjust_kmp — Public API

`adjust_kmp` là module Kotlin Multiplatform đóng vai trò **lớp nền Adjust** cho toàn bộ WaySDK: nó không tự wrap lại API của Adjust SDK thành facade riêng, mà (1) cung cấp helper chuyển đổi tiền tệ dùng chung khi track revenue, và (2) là **chủ sở hữu duy nhất của cinterop bridge `cocoapods.Adjust`** trên iOS — mọi module khác (`wayInstall`, `wayAd`, `wayPay`) đều `api(project(":adjust_kmp"))` để dùng trực tiếp các type Adjust gốc trên từng platform.

Cụ thể, public surface của module gồm 3 phần:

- **Kotlin chung (`commonMain`)**: file `AdjustMoney.kt` — helper quy đổi số tiền dạng micros về đơn vị tiền tệ mà Adjust yêu cầu khi set revenue.
- **iOS**: package cinterop `cocoapods.Adjust` (generate từ `src/nativeInterop/cinterop/Adjust.def`, module Objective-C `AdjustSdk`), expose nguyên bộ API của Adjust iOS SDK: `Adjust`, `ADJConfig`, `ADJEvent`, `ADJAdRevenue`, `ADJAttribution`, các hằng `ADJEnvironmentProduction` / `ADJEnvironmentSandbox`, `ADJLogLevelVerbose`, ...
- **Android**: re-export (`api`) hai thư viện `com.adjust.sdk:adjust-android` và `com.adjust.sdk:adjust-android-webbridge` (v5.7.0) — consumer dùng thẳng `com.adjust.sdk.*` (`Adjust`, `AdjustConfig`, `AdjustEvent`, `AdjustAdRevenue`, ...).

## Khởi tạo

Module **không có entry point / hàm init riêng**. Việc khởi tạo Adjust (tạo config, gọi `Adjust.initSdk(...)`) do các module tiêu thụ thực hiện trực tiếp trên type platform mà `adjust_kmp` expose (trong WaySDK hiện tại là `wayInstall`). Ví dụ pattern trên iOS:

```kotlin
import cocoapods.Adjust.ADJConfig
import cocoapods.Adjust.ADJEnvironmentProduction
import cocoapods.Adjust.Adjust

val config = ADJConfig(appToken = token, environment = ADJEnvironmentProduction)
Adjust.initSdk(config)
```

Và pattern track revenue dùng helper của module (chung cho cả hai platform):

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

Đây là API "mượn" — module không định nghĩa type nào, nhưng là nơi duy nhất trong WaySDK khai báo cinterop nên mọi import `cocoapods.Adjust.*` ở các module khác đều đi qua module này. Các type Adjust iOS SDK 5.4.6 mà WaySDK đang dùng qua bridge:

| Symbol | Vai trò |
|---|---|
| `Adjust` | Class tĩnh chính: `initSdk`, `trackEvent`, `trackAdRevenue`, đọc attribution, ... |
| `ADJConfig` | Config khởi tạo (app token, environment, log level). |
| `ADJEvent` | Event track thường / kèm revenue (`setRevenue`). |
| `ADJAdRevenue` | Track doanh thu quảng cáo (ad revenue). |
| `ADJAttribution` | Dữ liệu attribution trả về từ Adjust. |
| `ADJEnvironmentProduction`, `ADJEnvironmentSandbox`, `ADJLogLevelVerbose` | Hằng cấu hình môi trường / log. |

### Re-export Android: `com.adjust.sdk.*`

`androidMain` khai báo `api(adjust-android)` và `api(adjust-android-webbridge)` (v5.7.0), nên consumer của `adjust_kmp` dùng trực tiếp Adjust Android SDK (`Adjust`, `AdjustConfig`, `AdjustEvent`, `AdjustAdRevenue`, webbridge cho WebView) mà không cần tự khai báo dependency.

## Public models

Module **không định nghĩa model riêng** nào. Mọi model (event, config, attribution, ad revenue) là type gốc của Adjust SDK trên từng platform, được expose transitively như mô tả ở trên.

## Lưu ý platform

- **iOS — cinterop trực tiếp qua XCFramework (không dùng CocoaPods plugin):**
  - `Adjust.def`: `language = Objective-C`, `modules = AdjustSdk`, `package = cocoapods.Adjust` (giữ tên package `cocoapods.*` để tương thích code cũ, nhưng cơ chế là cinterop trực tiếp).
  - Cinterop compile với `-fmodules -F<slice>` trỏ vào `native-frameworks/AdjustSdk-5.4.6/.../AdjustSdk.xcframework`, slice `ios-arm64` cho `iosArm64` và `ios-arm64_x86_64-simulator` cho `iosSimulatorArm64`. XCFramework được task `:downloadInteropFrameworks` (root project) tải từ GitHub release `adjust/ios_sdk v5.4.6` (bản **Dynamic**); mọi task `cinterop*` phụ thuộc task này.
  - Cinterop chỉ cung cấp header lúc compile — **app iOS cuối phải tự link binary `AdjustSdk.xcframework` (qua SPM)**. Root project có task `verifyInteropVersions` đối chiếu version trong `Package.resolved` của `iosApp` (identity `ios_sdk` → `AdjustSdk`) với version cinterop `5.4.6` để bảo đảm hai bên không lệch nhau.
  - Framework Kotlin output: `adjust_kmpKit`, static. Target hỗ trợ: `iosArm64`, `iosSimulatorArm64` (không có x86_64 riêng ngoài slice simulator).
- **Android:** minSdk 24, compileSdk 36, namespace `com.adjust_kmp`. Adjust Android SDK v5.7.0 được expose dạng `api` nên tự động có mặt trong classpath của app.
- **Lệch version giữa hai platform:** iOS đang pin Adjust 5.4.6, Android 5.7.0 — cùng dòng major 5 nhưng không đồng bộ số minor.
- Publish lên GitHub Packages với coordinates `way-sdk:adjust_kmp:<way-sdk-version>`.
