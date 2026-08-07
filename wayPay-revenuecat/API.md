# wayPay-revenuecat — Public API

Backend RevenueCat cho `:wayPay`, chạy trên [purchases-kmp](https://github.com/RevenueCat/purchases-kmp) (hiện 3.2.1). Được gì so với `:wayPay-store`: verify receipt server-side, StoreKit 2 trên iOS, entitlement cross-device, Offerings/paywall/experiments, dashboard doanh thu. Đổi lại: phí RevenueCat ~1% MTR khi tổng doanh thu tracked của **toàn tài khoản RC** (cộng dồn mọi app/project) vượt ngưỡng free — cân nhắc `:wayPay-store` cho app doanh thu lớn.

Module `api` purchases-kmp — app dùng thẳng toàn bộ RC API (`Purchases.sharedInstance`, Offerings, paywall, subscriber attributes, `logIn`/`logOut`...) không cần khai báo lại dependency.

Phụ thuộc: `api(:wayPay)`, `api(purchases-kmp-core)`. Package: `com.waypay.revenuecat`.

## Nguyên tắc thiết kế: state là projection của RevenueCat

Mọi state chung (`activeEntitlements`, `isSubscription`, `isSubscriptionAsFlow`) được nuôi từ `CustomerInfo` listener của RC — **không** từ kết quả các hàm facade. Vì vậy purchase đi qua BẤT KỲ cửa nào — facade `AppPurchase`, RC API trực tiếp, RevenueCatUI paywall, renewal từ thiết bị khác — đều hội tụ về cùng state, và cơ chế auto-chặn ads của wayAd (đọc `isSubscription`) luôn đúng. App được tự do dùng tính năng RC ngoài facade mà không sợ lệch state.

## Khởi tạo

```kotlin
// Gọi 1 lần lúc app khởi động, TRƯỚC khi init ads / trước khi connect:
WayPayRevenueCat.install(
    apiKey = WayPayRevenueCat.platformApiKey(
        googleApiKey = "goog_...",
        appleApiKey  = "appl_...",
    ),
    appUserId = myStableUserId,   // null → RC anonymous ID
)
AppPurchaseManager.getInstance().connect(PurchaseConfig(subscriptionIds = setOf(...)))

// Nếu dùng integration server-side (Adjust, AppsFlyer...) — gọi càng sớm càng tốt, xem mục dưới:
WayPayRevenueCat.attachAttribution(AttributionSource.adjust())
```

## Public API

### `object WayPayRevenueCat`

Entry point duy nhất của module. Implementation (`RevenueCatPurchase`) là `internal`.

#### `fun install(apiKey: String, appUserId: String? = null, configure: PurchasesConfiguration.Builder.() -> Unit = {}): AppPurchase`

Configure RevenueCat (nếu `Purchases.isConfigured` chưa true) và đăng ký backend vào `AppPurchaseManager`. Idempotent — gọi lặp trả instance cũ. Nếu backend KHÁC đã install trước, backend đó thắng (first-wins).

| Param | Kiểu | Mô tả |
|---|---|---|
| `apiKey` | `String` | RC public API key theo ĐÚNG platform đang chạy (`goog_...` / `appl_...`). Dùng `platformApiKey` để chọn tự động. |
| `appUserId` | `String?` | ID user ổn định của app — entitlement đi theo user cross-device. `null` → RC tự cấp anonymous ID (theo device/install). Nên chốt convention ngay từ đầu; đổi sau rắc rối với user cũ. |
| `configure` | `PurchasesConfiguration.Builder.() -> Unit` | Tuỳ biến thêm config RC (`verificationMode`, `storeKitVersion`, `diagnosticsEnabled`...) trước khi build. |

**Trả về:** instance `AppPurchase` đang hoạt động.

#### `fun attachAttribution(vararg sources: AttributionSource, collectDeviceIdentifiers: Boolean = true, retry: AttributionRetry = AttributionRetry())`

Gắn ID attribution lên RC để bật **integration server-side** — RC tự forward purchase/renewal/refund về Adjust, AppsFlyer... mà app không phải track gì thêm. Xem [Cơ chế S2S](#cơ-chế-s2s-attachattribution) bên dưới.

| Param | Kiểu | Mô tả |
|---|---|---|
| `sources` | `vararg AttributionSource` | Danh sách nền tảng muốn bật — mỗi cái resolve + retry độc lập, cái lấy được ID sớm không phải chờ cái chậm. |
| `collectDeviceIdentifiers` | `Boolean` | `true` → gọi `Purchases.collectDeviceIdentifiers()`: Android thu `$gpsAdId`/`$ip`/`$deviceVersion`, iOS thu `$idfa`/`$idfv`/`$ip`. (KDoc của `purchases-kmp` còn ghi `$androidId` — đã stale, `purchases-android` 10.12.0 không còn collect Android ID.) |
| `retry` | `AttributionRetry` | Chính sách retry khi ID chưa sẵn sàng. Mặc định 10 lần, backoff 1s→30s → tổng thời gian chờ ~151s; riêng Adjust cộng thêm timeout đọc 5s mỗi lần nên cửa sổ thực tế ~3m20s. |

Throw `IllegalStateException` nếu chưa `install`. Gọi lặp là an toàn: source đã resolve xong hoặc đang chạy sẽ không khởi động lại — gọi thêm chỉ để bổ sung nền tảng mới.

#### `fun platformApiKey(googleApiKey: String, appleApiKey: String): String`

Helper chọn API key theo platform hiện tại (RC yêu cầu key riêng cho Google/Apple).

| Param | Kiểu | Mô tả |
|---|---|---|
| `googleApiKey` | `String` | Key `goog_...` — trả về khi chạy trên Android. |
| `appleApiKey` | `String` | Key `appl_...` — trả về khi chạy trên iOS. |

**Trả về:** key khớp platform đang chạy.

## Cơ chế S2S: attachAttribution

RC không tự biết user của app là ai bên Adjust/AppsFlyer. Mỗi integration của RC được kích hoạt bằng một **subscriber attribute** chứa ID của nền tảng đó (`$adjustId`, `$appsflyerId`...); có attribute thì RC mới forward purchase server-side. `attachAttribution` lo toàn bộ vòng đời của các attribute này.

### `class AttributionSource` (package `com.waypay.revenuecat.attribution`)

Constructor `internal` — tạo bằng factory. `adjust()` tự đọc `adid` từ Adjust SDK (WaySDK đã có sẵn dependency `:adjust_kmp`); các nền tảng còn lại nhận lambda `suspend () -> String?` vì SDK của chúng nằm ở phía app.

| Factory | Attribute RC | Nguồn ID phía app |
|---|---|---|
| `adjust(readTimeoutMs: Long = 5_000)` | `$adjustId` | **Tự động** — `Adjust.getAdidWithTimeout` (Android) / `Adjust.adidWithCompletionHandler` (iOS). `readTimeoutMs` là timeout cho mỗi lần đọc. |
| `appsFlyer { }` | `$appsflyerId` | `AppsFlyerLib.getInstance().getAppsFlyerUID(context)` |
| `firebase { }` | `$firebaseAppInstanceId` | `Firebase.analytics.appInstanceId` |
| `facebook { }` | `$fbAnonId` | `AppEventsLogger.getAnonymousAppDeviceGUID(context)` |
| `airbridge { }` | `$airbridgeDeviceId` | Airbridge SDK |
| `mparticle { }` | `$mparticleId` | mParticle SDK |
| `cleverTap { }` | `$clevertapId` | CleverTap SDK |
| `mixpanel { }` | `$mixpanelDistinctId` | Mixpanel SDK |
| `oneSignal { }` | `$onesignalUserId` | OneSignal v11+ |
| `postHog { }` | `$posthogUserId` | PostHog SDK |
| `airship { }` | `$airshipChannelId` | Airship SDK |
| `custom(key, name = key) { }` | `key` tuỳ ý | Dùng khi RC thêm integration mới hoặc cần gắn ID nội bộ. |

### `data class AttributionRetry(maxAttempts: Int = 10, initialDelayMs: Long = 1_000, maxDelayMs: Long = 30_000)`

Delay nhân đôi sau mỗi lần resolve trả về null/rỗng, chặn trên tại `maxDelayMs` — mặc định là 1+2+4+8+16+30×4 = 151s chờ, chưa kể thời gian mỗi lần đọc. Hết `maxAttempts` mà vẫn không có ID → log lỗi TAG `WayPay/RevenueCat` và bỏ source đó (không throw, không ảnh hưởng source khác).

Throw `IllegalArgumentException` nếu `maxAttempts <= 0`, `initialDelayMs < 0` hoặc `maxDelayMs < initialDelayMs`. `AttributionSource.adjust(readTimeoutMs)` cũng yêu cầu `readTimeoutMs > 0`.

### Ví dụ nhiều nền tảng

```kotlin
WayPayRevenueCat.install(apiKey = ..., appUserId = ...)

setupAdjust(tokenAdjust = token, isDebug = isDebug) { }   // không cần chờ callback

WayPayRevenueCat.attachAttribution(
    AttributionSource.adjust(),
    AttributionSource.appsFlyer { AppsFlyerLib.getInstance().getAppsFlyerUID(context) },
    AttributionSource.firebase { Firebase.analytics.appInstanceId.await() },
    collectDeviceIdentifiers = true,
)
```

### Hành vi

- **Tự retry trong một cửa sổ giới hạn** — `adid` chỉ tồn tại sau khi Adjust init xong và nhận response session đầu tiên, nên gọi `attachAttribution` ngay sau `install()` là được, không cần chờ callback của `setupAdjust`. Lần đầu mở app mà offline: vòng retry bắt kịp **nếu** mạng quay lại trong cửa sổ ~3m20s của config mặc định; quá cửa sổ thì source dừng và log lỗi — nâng `maxAttempts`/`maxDelayMs` nếu app cần bám lâu hơn.
- **Tự gắn lại khi đổi user** — subscriber attribute thuộc về từng app user id. Module theo dõi `appUserID` qua listener `CustomerInfo` và tự re-apply toàn bộ ID đã resolve sau `Purchases.awaitLogIn(...)` / `awaitLogOut()`; app không phải gọi lại.
- **Gọi càng sớm càng tốt** — RC đính attribute vào purchase tại thời điểm mua. ID gắn sau khi user đã mua thì purchase đó đã forward đi mà không có địa chỉ nền tảng → mất attribution cho chính giao dịch đó. Đặt `attachAttribution` lúc khởi động, trước khi mở paywall.
- **Idempotent** — gọi lặp không resolve lại source đã xong hoặc đang chạy.

### Lưu ý platform

- **iOS**: `$idfa` sẽ null nếu ATT chưa `AUTHORIZED` (`setupAdjust` đã tự xin quyền — xem `adjust_kmp/API.md`). `adid` không phụ thuộc ATT nên integration Adjust vẫn chạy.
- **Đọc `adid`**: Android dùng `Adjust.getAdidWithTimeout` — Adjust tự đặt timer và gỡ callback khi hết hạn, không đọng. iOS 5.4.6 chỉ có `adidWithCompletionHandler` (không có bản timeout) nên timeout do coroutine giữ; callback quá hạn vẫn nằm trong `cachedAdidReadCallbacksArray` của Adjust cho tới khi `adid` xuất hiện. Vì vậy nâng `maxAttempts` lên cao thì cân nhắc nâng `AttributionSource.adjust(readTimeoutMs)` thay vì tăng số lần đọc.
- **Android**: app target API 33+ phải khai báo permission `com.google.android.gms.permission.AD_ID`, nếu không `$gpsAdId` trả về chuỗi toàn số 0. App hướng tới trẻ em thì đặt `collectDeviceIdentifiers = false` theo [data safety của Google Play](https://rev.cat/google-plays-data-safety).
- ⚠️ Bật integration server-side rồi thì **đừng** truyền `AdjustPurchaseTracker` vào `PurchaseConfig.tracker` — double-count revenue.

## Mapping contract `AppPurchase` → RevenueCat

| Hàm facade | Thực thi bằng | Ghi chú |
|---|---|---|
| `connect(config)` | set delegate + `awaitCustomerInfo()` (cache-first) | Suspend đến khi fetch CustomerInfo đầu tiên hoàn tất → thông thường sau `connect()`, `isSubscription` dùng được ngay. Từ lần mở app thứ 2, RC có cache disk → return gần như tức thì kể cả offline. **Degrade case**: nếu fetch đầu tiên thất bại (vd lần đầu mở app offline), `connect()` vẫn return nhưng `isReady` giữ `false`; backend tự retry nền (backoff 5s→60s) đến khi nhận được CustomerInfo đầu tiên — lúc đó `isReady` mới thành `true`. Trong lúc chờ, state ở tầng `AppPurchaseManager` giữ **snapshot từ phiên trước** (nếu có) nên premium user không bị coi nhầm là free. Throw `IllegalStateException` nếu chưa `install`. |
| `queryProducts(type)` | `awaitGetProducts(ids)` | `ids` lấy từ `PurchaseConfig` theo `type`. Lỗi → empty list, không throw. |
| `purchase(productId)` | `awaitGetProducts` → `awaitPurchase(storeProduct)` | Tôn trọng `isOfferPersonalized`. Map lỗi RC → `PurchaseError` (bảng dưới). |
| `restorePurchases()` | `awaitRestore()` | Trả record cho sub active + non-consumable đã mua. |
| `openSubscriptionSettings()` | mở `managementUrl` từ CustomerInfo | Fallback: trang subscription mặc định của store. |
| `disconnect()` | gỡ delegate, `isReady = false` | Không "đóng" RC thật — RC là singleton process-wide. |

### Mapping lỗi RC → `PurchaseError`

| RC | `PurchaseError` |
|---|---|
| `userCancelled` / `PurchaseCancelledError` | `UserCanceled` |
| `NetworkError`, `OfflineConnectionError` | `NetworkError` |
| `ProductAlreadyPurchasedError` | `AlreadyOwned` |
| `ProductNotAvailableForPurchaseError` | `ItemUnavailable` |
| `PaymentPendingError` | `PendingState` (kèm event `Pending`) |
| `PurchaseNotAllowedError`, `ConfigurationError`, `StoreProblemError` | `BillingUnavailable` |
| còn lại | `ServiceError(code, message)` |

## Khác biệt hành vi so với `:wayPay-store`

Đọc kỹ trước khi đổi backend:

- **`PurchaseConfig.verifier` bị BỎ QUA** (log warning) — RC verify server-side. Backend app cần biết purchase → dùng [webhook RC](https://www.revenuecat.com/docs/integrations/webhooks).
- **`PurchaseRecord.purchaseToken` không phải receipt thô** — chỉ là transactionId (RC giữ receipt). `signature`/`originalJson` = `null`. Riêng record từ `restorePurchases()`: non-consumable có `transactionId` lấy từ `nonSubscriptionTransactions`; subscription để rỗng (`""`) vì RC không expose per-product transaction trong `CustomerInfo`.
- **`purchaseEvents` chỉ phát cho hành động qua facade** (`USER_INITIATED`, `RESTORE`, `Pending`, `Failed`). Renewal và external purchase KHÔNG phát event — chúng phản ánh qua `activeEntitlements`/`isSubscriptionAsFlow`; cần chi tiết renewal thì dùng webhook RC hoặc đọc `CustomerInfo` trực tiếp.
- **`PurchaseConfig.tracker` vẫn được gọi** sau purchase thành công. ⚠️ Nếu đã bật integration Adjust server-side trên dashboard RC (xem [`attachAttribution`](#cơ-chế-s2s-attachattribution)) thì đừng truyền `AdjustPurchaseTracker` — double-count revenue.
- **`isSubscription`** = `CustomerInfo.activeSubscriptions` không rỗng. **`activeEntitlements`** = `activeSubscriptions` + (`nonConsumableIds` ∩ `allPurchasedProductIdentifiers`); trước khi `connect(config)` chỉ gồm `activeSubscriptions`.
- **`ProductInfo.subscription`**: `renewalState` luôn `UNKNOWN`, `autoRenewing`/`expiryTimeMs` = `null` (đọc từ `CustomerInfo`/RC API nếu cần chi tiết); period/trial/intro map từ `StoreProduct`.

## Dùng RC ngoài facade

```kotlin
// Offerings + paywall — không đi qua AppPurchase, state chung vẫn tự đúng:
val offerings = Purchases.sharedInstance.awaitOfferings()
val pkg = offerings.current?.availablePackages?.first()
Purchases.sharedInstance.awaitPurchase(pkg)   // isSubscription tự cập nhật qua listener

// Identity:
Purchases.sharedInstance.awaitLogIn(userId)
```

Lưu ý lock-in: phần code dùng RC-native API (Offerings, paywall...) không swap được sang `:wayPay-store` — chỉ phần đi qua interface chung swap được.

## Lưu ý platform

- **Android**: RC tự init với application context qua androidx-startup — không cần truyền Context.
- **iOS**: cần link `PurchasesHybridCommon` XCFramework của RC vào app qua SPM (theo hướng dẫn purchases-kmp). Không còn giới hạn StoreKit 1 — RC native SDK dùng StoreKit 2.
- **Migration user cũ từ `:wayPay-store`**: RC tự sync purchase từ store khi app chạy lần đầu với RC; test kỹ kịch bản "user đang có sub active, update app sang bản dùng RC" để entitlement nhận diện ngay, không hở khoảng premium.

## Log TAG

Module log với TAG `WayPay/RevenueCat`. Toàn bộ quy ước TAG của WaySDK: xem `wayCore/API.md`.
