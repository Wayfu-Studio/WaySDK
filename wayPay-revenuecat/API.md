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

#### `fun platformApiKey(googleApiKey: String, appleApiKey: String): String`

Helper chọn API key theo platform hiện tại (RC yêu cầu key riêng cho Google/Apple).

| Param | Kiểu | Mô tả |
|---|---|---|
| `googleApiKey` | `String` | Key `goog_...` — trả về khi chạy trên Android. |
| `appleApiKey` | `String` | Key `appl_...` — trả về khi chạy trên iOS. |

**Trả về:** key khớp platform đang chạy.

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
- **`PurchaseConfig.tracker` vẫn được gọi** sau purchase thành công. ⚠️ Nếu đã bật integration Adjust server-side trên dashboard RC thì đừng truyền `AdjustPurchaseTracker` — double-count revenue.
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
