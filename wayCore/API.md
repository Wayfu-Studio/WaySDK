# wayCore — Public API

`wayCore` là module nền tảng (Kotlin Multiplatform, targets Android + iOS) của WaySDK. Module cung cấp hệ thống logging thống nhất (`WayLog` / `WayLogger` / `LogLevel`, package `com.waycore.logger`) và kiểu trừu tượng hoá host của ứng dụng (`AppHost`, package `com.waycore.host`) để các module khác trong WaySDK tham chiếu tới màn hình/host hiện tại mà không phụ thuộc trực tiếp vào API từng nền tảng.

Các module khác của WaySDK phụ thuộc vào wayCore để ghi log; ứng dụng chủ có thể cài (install) logger tuỳ chỉnh của mình để nhận toàn bộ log của SDK.

## Public API

### `WayLog` (object)

Facade logging tĩnh, thread-safe (các trường được đánh dấu `@Volatile`). Mặc định chuyển tiếp log tới logger nền tảng (Android: `android.util.Log`; iOS: `NSLog`). Mỗi lời gọi log chỉ được chuyển tiếp khi mức log của lời gọi có `priority >= minLevel.priority`; nếu không, lời gọi bị bỏ qua.

**Property:**

| Property | Kiểu | Ý nghĩa |
|---|---|---|
| `minLevel` | `var LogLevel` | Ngưỡng lọc log, mặc định `LogLevel.DEBUG`. Chỉ các lời gọi có mức ≥ `minLevel` mới được chuyển tới logger. Đặt `LogLevel.NONE` để tắt toàn bộ log (priority của `NONE` là `Int.MAX_VALUE` nên không mức nào vượt qua được). |

#### `fun install(logger: WayLogger)`

Cài logger tuỳ chỉnh, thay thế logger nền tảng mặc định. Mọi lời gọi `WayLog.d/i/w/e` sau đó (đã qua bộ lọc `minLevel`) sẽ được chuyển tới logger này. Đây là điểm mở rộng để ứng dụng chủ chuyển log của SDK sang hệ thống logging riêng (Timber, os_log, ghi file, …).

| Param | Kiểu | Mô tả |
|---|---|---|
| `logger` | `WayLogger` | Cài đặt logger tuỳ chỉnh nhận toàn bộ log của SDK. |

**Trả về:** `Unit`

```kotlin
WayLog.install(object : WayLogger {
    override fun d(tag: String, message: String) { /* ... */ }
    override fun i(tag: String, message: String) { /* ... */ }
    override fun w(tag: String, message: String, throwable: Throwable?) { /* ... */ }
    override fun e(tag: String, message: String, throwable: Throwable?) { /* ... */ }
})
WayLog.minLevel = LogLevel.WARN
```

#### `fun reset()`

Gỡ logger tuỳ chỉnh, khôi phục về logger nền tảng mặc định (Android: Logcat; iOS: `NSLog`). Không thay đổi `minLevel`.

**Trả về:** `Unit`

#### `fun d(tag: String, message: String)`

Ghi log mức DEBUG. Chỉ chuyển tới logger khi `minLevel` ≤ DEBUG.

| Param | Kiểu | Mô tả |
|---|---|---|
| `tag` | `String` | Nhãn phân loại nguồn log. |
| `message` | `String` | Nội dung log. |

**Trả về:** `Unit`

#### `fun i(tag: String, message: String)`

Ghi log mức INFO. Chỉ chuyển tới logger khi `minLevel` ≤ INFO.

| Param | Kiểu | Mô tả |
|---|---|---|
| `tag` | `String` | Nhãn phân loại nguồn log. |
| `message` | `String` | Nội dung log. |

**Trả về:** `Unit`

#### `fun w(tag: String, message: String, throwable: Throwable? = null)`

Ghi log mức WARN, kèm exception tuỳ chọn. Chỉ chuyển tới logger khi `minLevel` ≤ WARN.

| Param | Kiểu | Mô tả |
|---|---|---|
| `tag` | `String` | Nhãn phân loại nguồn log. |
| `message` | `String` | Nội dung log. |
| `throwable` | `Throwable?` (mặc định `null`) | Exception đính kèm, có thể bỏ qua. |

**Trả về:** `Unit`

#### `fun e(tag: String, message: String, throwable: Throwable? = null)`

Ghi log mức ERROR, kèm exception tuỳ chọn. Chỉ chuyển tới logger khi `minLevel` ≤ ERROR.

| Param | Kiểu | Mô tả |
|---|---|---|
| `tag` | `String` | Nhãn phân loại nguồn log. |
| `message` | `String` | Nội dung log. |
| `throwable` | `Throwable?` (mặc định `null`) | Exception đính kèm, có thể bỏ qua. |

**Trả về:** `Unit`

### `WayLogger` (interface)

Contract cho một backend logging. Ứng dụng chủ implement interface này và truyền vào `WayLog.install(...)` để tự xử lý log của SDK. Lưu ý: `WayLog` đã lọc theo `minLevel` trước khi gọi, nên implementation không cần tự lọc mức log.

#### `fun d(tag: String, message: String)`
Xử lý log mức DEBUG.

#### `fun i(tag: String, message: String)`
Xử lý log mức INFO.

#### `fun w(tag: String, message: String, throwable: Throwable? = null)`
Xử lý log mức WARN, `throwable` có thể `null`.

#### `fun e(tag: String, message: String, throwable: Throwable? = null)`
Xử lý log mức ERROR, `throwable` có thể `null`.

(Tất cả trả về `Unit`; các hàm đều đồng bộ, không `suspend`.)

### `AppHost` (expect class)

Kiểu trừu tượng đại diện cho "host" giao diện của ứng dụng trên từng nền tảng, dùng làm tham số chung khi các module WaySDK cần tham chiếu tới màn hình hiện tại (ví dụ để hiển thị UI hệ thống). Không có API riêng — nó là typealias sang kiểu nền tảng:

| Platform | Kiểu thực tế |
|---|---|
| Android | `android.app.Activity` |
| iOS | `platform.UIKit.UIViewController` |

## Public models

### `LogLevel` (enum)

Mức log, dùng cho `WayLog.minLevel`. Mỗi giá trị mang một `priority: Int` (public); số càng lớn mức càng nghiêm trọng.

| Giá trị | `priority` | Ý nghĩa |
|---|---|---|
| `VERBOSE` | 2 | Mức thấp nhất. Lưu ý: `WayLog` hiện không có hàm ghi log VERBOSE — giá trị này chỉ hữu ích làm ngưỡng `minLevel` (tương đương cho phép tất cả). |
| `DEBUG` | 3 | Log gỡ lỗi (`WayLog.d`). Là `minLevel` mặc định. |
| `INFO` | 4 | Log thông tin (`WayLog.i`). |
| `WARN` | 5 | Cảnh báo (`WayLog.w`). |
| `ERROR` | 6 | Lỗi (`WayLog.e`). |
| `NONE` | `Int.MAX_VALUE` | Tắt toàn bộ log khi đặt làm `minLevel`. |

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `priority` | `Int` | Độ ưu tiên dùng để so sánh với `minLevel` (giá trị trùng với hằng số mức log của Android `android.util.Log`). |

## Lưu ý platform

- **Android**: logger mặc định ghi ra Logcat qua `android.util.Log.d/i/w/e`, giữ nguyên `tag`; `throwable` (nếu có) được truyền thẳng cho Logcat.
- **iOS**: logger mặc định ghi qua `NSLog` với định dạng `[WayLog/<D|I|W|E>][<tag>] <message>`; nếu có `throwable`, stack trace (`stackTraceToString()`) được nối vào sau message trên dòng mới. Ký tự `%` trong chuỗi được escape thành `%%` để tránh bị `NSLog` hiểu nhầm là format specifier.
- Các logger mặc định (`AndroidLogger`, `IosLogger`, `NoOpLogger`) đều là `internal` — muốn thay đổi hành vi log chỉ có một con đường public là `WayLog.install(...)`.
- Toàn bộ API là đồng bộ; module không dùng `suspend` hay `Flow`.

## Quy ước log TAG toàn WaySDK

Mọi log của SDK (qua `WayLog`) dùng TAG thống nhất định dạng **`Way<Module>/<Component>`** — filter toàn SDK bằng prefix `Way`, filter theo module bằng `WayPay/`, `WayAd/`...

```bash
# Android — toàn bộ log WaySDK:
adb logcat | grep -E "Way(Pay|Ad|Install|Lifecycle|Core)/"
# Chỉ billing:
adb logcat | grep "WayPay/"
```
iOS: filter Console/Xcode bằng chuỗi `Way`.

| Module | TAG |
|---|---|
| `:wayPay` (core) | `WayPay/Manager`, `WayPay/AdjustTracker` |
| `:wayPay-store` | `WayPay/Store` |
| `:wayPay-revenuecat` | `WayPay/RevenueCat` |
| `:wayAd` | `WayAd/Kit`, `WayAd/AdmobAdapter`, `WayAd/MaxAdapter`, `WayAd/StrategyLoader`, `WayAd/Preload`, `WayAd/AppOpenManager`, `WayAd/Adjust`, `WayAd/BannerLayout`, `WayAd/BannerEffect`, `WayAd/AdmobNativeLayout`, `WayAd/MaxNativeLayout`, `WayAd/NativeAdState key=<keyDebug>`, `WayAd/AdmobBannerView`, `WayAd/AdmobNativeView`, `WayAd/MaxBannerView`, `WayAd/MaxNativeView` |
| `:wayInstall` | `WayInstall/Kit`, `WayInstall/Cache` |
| `:lifecycle_kmp` | `WayLifecycle/NetworkMonitor` |

Lưu ý: các XML view của wayAd có `setDebugTag(tagDebug)` thay thế hoàn toàn TAG mặc định — nếu dùng, nên tự giữ prefix `WayAd/` để không lọt filter.
