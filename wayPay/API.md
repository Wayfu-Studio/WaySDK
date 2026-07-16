# wayPay — Public API

Module core cho in-app purchase / subscription của WaySDK, kiến trúc **pluggable backend**. Module này chỉ chứa contract (interface `AppPurchase`, models, hooks) và điểm truy cập `AppPurchaseManager` — **không chứa billing implementation**. App chọn 1 trong 2 backend lúc khởi động:

| Module | Backend | Install |
|---|---|---|
| `:wayPay-store` | Google Play Billing (Android) + StoreKit 1 (iOS), tự vận hành, không phí bên thứ ba | `WayPayStore.install()` |
| `:wayPay-revenuecat` | RevenueCat (purchases-kmp) — verify server-side, StoreKit 2, Offerings/paywall; tính ~1% MTR cộng dồn toàn tài khoản RC | `WayPayRevenueCat.install(apiKey)` |

wayAd và mọi module nội bộ SDK chỉ phụ thuộc module core này. Đổi backend = đổi 1 dòng install, miễn là app chỉ dùng API trong interface chung. Phụ thuộc: `api(:lifecycle_kmp)`, `api(:adjust_kmp)`.

Package: `com.waypay`, `com.waypay.model`, `com.waypay.hook`.

## Bắt đầu nhanh

```kotlin
// 0. Chọn backend 1 lần lúc app start, TRƯỚC khi init ads:
WayPayStore.install()   // hoặc WayPayRevenueCat.install(apiKey = ...)

val pay = AppPurchaseManager.getInstance()

// 1. Connect 1 lần khi app start
pay.connect(PurchaseConfig(
    subscriptionIds  = setOf("premium_monthly", "premium_yearly"),
    nonConsumableIds = setOf("remove_ads"),
    consumableIds    = setOf("coin_100"),
))

// 2. Hiển thị danh sách sản phẩm
val subs = pay.queryProducts(ProductType.SUBS)

// 3. Mua
when (val r = pay.purchase("premium_monthly")) {
    // xử lý Result<PurchaseRecord>
}

// 4. Nút Restore (iOS bắt buộc phải có)
pay.restorePurchases()

// 5. Quan sát trạng thái subscription để toggle UI / ads
launch {
    pay.isSubscriptionAsFlow.collect { isPremium ->
        if (isPremium) hideAds() else showAds()
    }
}
```

## Public API

### `object AppPurchaseManager`

Điểm truy cập `AppPurchase` cho toàn SDK theo mô hình install-based backend.

#### `fun getInstance(): AppPurchase`

Trả về instance dùng chung. **Luôn trả về cùng một proxy ổn định** — an toàn để cache qua `by lazy` bất kể thứ tự install/init. Trước khi backend được install: state đọc trả **snapshot từ phiên trước** nếu có, ngược lại giá trị mặc định an toàn (`isSubscription = false`, entitlements rỗng, `isReady = false`); `connect()` throw `IllegalStateException` với message hướng dẫn; `purchase()` trả `Result.failure(PurchaseError.NotConnected)`; `queryProducts()`/`restorePurchases()` trả empty list kèm log lỗi. Sau khi install: call được delegate sang backend; state entitlement được mirror từ backend **kể từ khi backend `isReady`** (trước đó giữ snapshot — giá trị `false` khởi tạo của backend chưa phải dữ liệu thật) và mỗi cập nhật đều được persist làm snapshot cho phiên sau.

**Trả về:** proxy `AppPurchase` (singleton suốt vòng đời process).

#### `fun install(backend: AppPurchase): AppPurchase`

Đăng ký backend thực. Được gọi từ `WayPayStore.install()` / `WayPayRevenueCat.install(...)` — không gọi trực tiếp trừ khi tự viết backend riêng.

Semantics **first-wins**: lần install đầu tiên thắng; các lần gọi sau bị bỏ qua (log error) và trả về backend đã có. Không hỗ trợ đổi backend lúc runtime.

| Param | Kiểu | Mô tả |
|---|---|---|
| `backend` | `AppPurchase` | Backend implementation cần đăng ký. |

**Trả về:** instance đang hoạt động sau lời gọi (backend vừa install, hoặc backend cũ nếu bị gọi lặp).

#### `val isInstalled: Boolean`

`true` nếu đã có backend được install.

---

### `interface AppPurchase`

API thống nhất quản lý in-app purchase / subscription cho cả Android và iOS. Vòng đời điển hình: `connect` (1 lần, idempotent) → `queryProducts` → `purchase` → subscribe `purchaseEvents` suốt vòng đời app → `disconnect` (optional). Caller không cần biết product là consumable / non-consumable / subscription khi gọi `purchase` — backend tự xác định từ `PurchaseConfig`. Không phụ thuộc Activity/Context ở public API.

#### Properties (state)

| Property | Kiểu | Ý nghĩa |
|---|---|---|
| `isReady` | `StateFlow<Boolean>` | Backend đã sẵn sàng và có dữ liệu entitlement authoritative chưa. Store backend Android = `BillingClient.isReady`; store iOS = `true` sau `connect()`; RC = `true` khi nhận được CustomerInfo đầu tiên (fetch fail thì vẫn `false` cho đến khi retry nền thành công). |
| `isSubscription` | `Boolean` | Snapshot đồng bộ: user có ít nhất 1 subscription active. Dùng cho check nhanh (gating tính năng, ẩn ads — wayAd đọc property này). |
| `isSubscriptionAsFlow` | `StateFlow<Boolean>` | Phiên bản Flow của `isSubscription`, dùng cho UI reactive. |
| `activeEntitlements` | `StateFlow<Set<String>>` | Tập productId user đang có quyền dùng: subscription active + non-consumable đã mua. KHÔNG chứa consumable (one-shot — app tự lưu state). Cập nhật khi connect xong, mua thành công, restore, hoặc renewal/external event. |
| `purchaseEvents` | `SharedFlow<PurchaseEvent>` | Stream mọi purchase event. Không replay — subscribe sớm (ngay sau `connect`). Phạm vi event khác nhau giữa 2 backend, xem docs từng backend. |

#### `suspend fun connect(config: PurchaseConfig)`

Khởi tạo connection và đăng ký productId. Idempotent — gọi lại không re-connect, chỉ cập nhật config.

Sau khi `connect()` return, state phản ánh **dữ liệu tốt nhất hiện có**, theo thứ tự ưu tiên: kết quả fetch mới (có mạng) → cache của backend → **snapshot cục bộ từ phiên trước** (proxy tự persist `isSubscription`/`activeEntitlements` vào SharedPreferences/NSUserDefaults mỗi khi backend cập nhật, và seed lại lúc khởi động — sống xuyên qua cả việc đổi backend, nên migration store→RC offline vẫn giữ đúng trạng thái premium). `isSubscription` chỉ là `false` mặc định khi install mới chưa từng có dữ liệu entitlement nào — kịch bản mà user chưa thể là premium. Theo dõi `isReady` nếu cần biết chính xác thời điểm backend đã có dữ liệu authoritative.

| Param | Kiểu | Mô tả |
|---|---|---|
| `config` | `PurchaseConfig` | Cấu hình productId, verifier, tracker. Xem [`PurchaseConfig`](#purchaseconfig). |

**Trả về:** `Unit`. Throw `IllegalStateException` nếu chưa install backend.

#### `fun disconnect()`

Đóng connection, release resource. Sau khi gọi, mọi hàm khác (trừ `connect`) trả lỗi/empty.

#### `suspend fun queryProducts(type: ProductType): List<ProductInfo>`

Query chi tiết sản phẩm từ store (tên, giá, mô tả).

| Param | Kiểu | Mô tả |
|---|---|---|
| `type` | `ProductType` | `INAPP` (consumable + non-consumable) hoặc `SUBS`. |

**Trả về:** danh sách `ProductInfo`. Empty list nếu lỗi — **không throw**.

#### `suspend fun purchase(productId: String): Result<PurchaseRecord>`

Mua sản phẩm. Backend tự xác định loại từ config: consumable → tự consume/finish (mua lại được); non-consumable/subscription → tự acknowledge/finish.

| Param | Kiểu | Mô tả |
|---|---|---|
| `productId` | `String` | ID đã đăng ký trong `PurchaseConfig` (1 trong 3 set). Chưa đăng ký → `Result.failure(PurchaseError.NotRegistered)`. |

**Trả về:** `Result.success(PurchaseRecord)` khi mua + verify (nếu có) + finalize thành công; `Result.failure` với cause là `PurchaseError` (phân biệt user cancel vs lỗi thật qua các sentinel object). Kết quả đồng thời được emit qua `purchaseEvents`.

#### `suspend fun restorePurchases(): List<PurchaseRecord>`

Khôi phục purchase đã mua (cài lại app, đổi máy). **iOS bắt buộc có UI button gọi hàm này** (App Review Guideline 3.1.1). Mỗi purchase khôi phục được verify (nếu có verifier), acknowledge/finish nếu chưa, thêm vào `activeEntitlements`, và phát qua `purchaseEvents` với source `RESTORE`.

**Trả về:** danh sách non-consumable & subscription đang active. Consumable không bao giờ được restore (store không lưu).

#### `fun openSubscriptionSettings()`

Mở trang quản lý subscription của OS (Google Play / App Store) để user cancel/đổi gói. Khác `restorePurchases`: hàm này để user THAY ĐỔI subscription hiện có, không phải phục hồi entitlement.

## Public models

Package `com.waypay.model`. Tất cả `@Serializable` trừ `PurchaseConfig`, `PurchaseEvent`, `PurchaseError`.

### `PurchaseConfig`

Cấu hình truyền vào `connect()`. Chia 3 set productId vì backend cần biết loại để quyết định cách finalize. Mỗi productId chỉ được thuộc 1 set — trùng thì behavior undefined.

| Field | Kiểu | Default | Ý nghĩa |
|---|---|---|---|
| `subscriptionIds` | `Set<String>` | `emptySet()` | Gói subscription tự gia hạn. |
| `consumableIds` | `Set<String>` | `emptySet()` | In-app tiêu hao (coin, gem) — mua lại được nhiều lần. |
| `nonConsumableIds` | `Set<String>` | `emptySet()` | In-app không tiêu hao (remove ads) — mua 1 lần dùng mãi. |
| `verifier` | `PurchaseVerifier?` | `null` | Hook verify backend. `null` = trust client. Backend RevenueCat BỎ QUA field này (RC verify server-side). |
| `tracker` | `PurchaseTracker?` | `null` | Hook analytics, gọi sau khi verify + finalize thành công. |
| `isOfferPersonalized` | `Boolean` | `false` | EU personalized-pricing flag. Android: pass vào `BillingFlowParams`; iOS: bỏ qua. |

### `ProductType`

| Giá trị | Ý nghĩa |
|---|---|
| `INAPP` | One-time purchase, gồm cả consumable và non-consumable. |
| `SUBS` | Subscription tự gia hạn. |

### `ProductInfo`

Thông tin sản phẩm dùng chung 2 platform, trả về từ `queryProducts`.

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `ids` | `List<String>` | productId/sku. Property `id: String?` trả phần tử đầu. |
| `type` | `ProductType` | Loại sản phẩm. |
| `title` | `String` | Tiêu đề hiển thị. |
| `description` | `String?` | Mô tả. |
| `price` | `Money` | Giá hiện tại. |
| `entitlementActive` | `Boolean` | User hiện có quyền dùng (INAPP đã sở hữu / SUBS còn hiệu lực). |
| `isSubscribed` | `Boolean?` | Chỉ SUBS: đang là thuê bao. INAPP = `null`. |
| `subscription` | `SubscriptionInfo?` | Thông tin thêm cho SUBS (chu kỳ, trial, giá intro, expiry). `null` với INAPP. |

`companion object` có `mockProductInfos(): List<ProductInfo>` — data mẫu cho preview/demo.

### `Money`

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `amountMicros` | `Long` | Giá theo micro-unit (1 đơn vị tiền = 1_000_000 micro). |
| `currencyCode` | `String` | Mã ISO-4217 ("USD", "VND"). |
| `formatted` | `String?` | Chuỗi đã format theo locale store ("$4.99"), có thể `null`. |

### `SubscriptionInfo`

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `periodISO8601` | `String?` | Chu kỳ cơ bản ISO-8601: `P1W` / `P1M` / `P1Y`... |
| `trialPeriodISO8601` | `String?` | Trial period nếu có (`P7D`...). |
| `introPrice` | `Money?` | Giá intro/ưu đãi nếu có. |
| `renewalState` | `RenewalState` | Trạng thái auto-renew. |
| `autoRenewing` | `Boolean?` | Dạng bool đơn giản của renewal; `null` nếu không rõ. |
| `expiryTimeMs` | `Long?` | Thời điểm hết hạn (epoch millis) nếu biết. |

### `RenewalState`

| Giá trị | Ý nghĩa |
|---|---|
| `UNKNOWN` | Không xác định được. |
| `AUTO_RENEW_ON` | Sẽ tự gia hạn. |
| `AUTO_RENEW_OFF` | User đã tắt gia hạn (sub chạy đến hết chu kỳ). |

### `Platform`

| Giá trị | Ý nghĩa |
|---|---|
| `ANDROID` / `IOS` | Nền tảng phát sinh purchase — server có thể route verify khác nhau. |

### `PurchaseRecord`

Dữ liệu một purchase. Trạng thái finalize phụ thuộc ngữ cảnh nhận được record:
- **Trả về từ `purchase`/`restorePurchases` hoặc trong `PurchaseEvent.Success` / `PurchaseTracker.track`**: đã verify (nếu có verifier) và đã ack/consume/finish.
- **Truyền vào `PurchaseVerifier.verify`**: payload **PRE-finalize** — backend verify được gọi TRƯỚC khi ack/consume/finish; kết quả verify quyết định purchase có được finalize hay không.

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `productId` | `String` | Product ID đã đăng ký trong config. |
| `productType` | `ProductType` | Loại sản phẩm. |
| `transactionId` | `String` | Transaction của LẦN này. Android: `Purchase.orderId` (fallback `""`); iOS: `SKPaymentTransaction.transactionIdentifier` (fallback `""`). Mỗi renewal có transactionId khác — dùng dedup analytics. |
| `originalTransactionId` | `String?` | Transaction đầu tiên của subscription (giữ nguyên qua renewal). iOS: `originalTransaction?.transactionIdentifier` (fallback = `transactionId` khi không có original); Android thường = `transactionId` hoặc `null`. |
| `purchaseTimeMs` | `Long` | Epoch millis lúc mua. |
| `purchaseToken` | `String` | Token verify server. Android: `purchaseToken` (Google Play Developer API); iOS: **app receipt base64** (StoreKit 1 — đọc từ `appStoreReceiptURL`, gửi cho App Store receipt validation / App Store Server API; KHÔNG phải JWS). Backend RevenueCat: transactionId (best-effort — RC giữ receipt). |
| `signature` | `String?` | Android only. iOS/RC = `null`. |
| `originalJson` | `String?` | Android only: JSON gốc từ Google. |
| `priceAmountMicros` | `Long?` | Giá lúc mua (micro units) — cần cho revenue tracking. |
| `priceCurrencyCode` | `String?` | Mã tiền tệ lúc mua. |
| `platform` | `Platform` | Nền tảng phát sinh. |

### `PurchaseEvent` (sealed class)

Event phát từ `purchaseEvents`.

| Case | Fields | Ý nghĩa |
|---|---|---|
| `Success` | `record: PurchaseRecord`, `source: Source` | Mua / khôi phục / renew thành công và đã finalize. |
| `Pending` | `productId: String` | Purchase chờ approval (Android: payment async; iOS: Ask to Buy). Sẽ có `Success` sau nếu approved. App nên hiện "đang chờ duyệt" thay vì error. |
| `Failed` | `productId: String?`, `error: PurchaseError` | Mua / khôi phục thất bại. |

`enum Source`: `USER_INITIATED` (caller gọi `purchase()`), `RESTORE` (caller gọi `restorePurchases()`), `RENEWAL` (reserved — **hiện chưa backend nào phát**; renewal trên iOS store backend về dưới `EXTERNAL`, Android chỉ phản ánh qua `activeEntitlements`), `EXTERNAL` (mua thiết bị khác, Family Sharing, pending→approved, renewal iOS).

### `PurchaseError` (sealed class, extends `Throwable`)

Lỗi từ luồng mua/khôi phục — tương thích `Result.failure`; caller dùng `result.exceptionOrNull() as? PurchaseError` để phân biệt. Hầu hết là object sentinel.

| Case | Ý nghĩa |
|---|---|
| `UserCanceled` | User đóng dialog mua — KHÔNG nên show error. |
| `NetworkError` | Lỗi mạng khi gọi billing service hoặc verifier. |
| `BillingUnavailable` | Billing service không khả dụng (Play Services lỗi / IAP bị disable qua parental control). |
| `AlreadyOwned` | Đã sở hữu (non-consumable / sub đang active). |
| `ItemUnavailable` | Sản phẩm không tồn tại hoặc chưa publish/approve trên console. |
| `PendingState` | Purchase đang chờ approval — hiện "chờ duyệt", không phải error. |
| `VerifyFailed` | Verifier trả `false`/throw → không finalize. Android: Google tự refund sau ~3 ngày (ack window). iOS SK1: KHÔNG auto-refund — transaction chưa finish sẽ được `SKPaymentQueue` replay ở lần khởi động sau cho tới khi finish/xử lý lại. |
| `NotConnected` | Chưa `connect()`, đã `disconnect()`, hoặc chưa install backend. |
| `NotRegistered` | productId chưa khai trong `PurchaseConfig`. |
| `ServiceError(code: Int, debugMessage: String?)` | Lỗi billing service có response code, khi không khớp case trên. |
| `Unknown(cause: Throwable?)` | Không phân loại được. |

## Hooks

Package `com.waypay.hook`.

### `fun interface PurchaseVerifier`

Hook verify purchase với backend của app (chống fake purchase, lưu entitlement server-side).

#### `suspend fun verify(record: PurchaseRecord): Boolean`

| Param | Kiểu | Mô tả |
|---|---|---|
| `record` | `PurchaseRecord` | Purchase cần verify — gửi `purchaseToken` + `productId` lên backend (Android: Google Play Developer API; iOS: `purchaseToken` là app receipt base64 — verify qua App Store receipt validation / App Store Server API). |

**Trả về:** `true` → backend module tiếp tục acknowledge/finish; `false` hoặc throw → KHÔNG finish, purchase fail với `VerifyFailed`. Hệ quả không-finish khác nhau theo platform: Android — Google tự refund sau ~3 ngày (ack window); iOS SK1 — không auto-refund, transaction được `SKPaymentQueue` replay ở lần khởi động sau. Truyền qua `PurchaseConfig.verifier`. Backend RevenueCat bỏ qua hook này.

### `fun interface PurchaseTracker`

Hook track purchase tới analytics provider. Chỉ được gọi sau khi verify + finalize thành công. Nếu throw: chỉ log, không ảnh hưởng kết quả `purchase()`.

#### `suspend fun track(record: PurchaseRecord)`

| Param | Kiểu | Mô tả |
|---|---|---|
| `record` | `PurchaseRecord` | Purchase đã finalize, có `priceAmountMicros`/`priceCurrencyCode` cho revenue. |

Track nhiều provider → viết 1 composite tracker gọi lần lượt trong `track()`.

### `expect class AdjustPurchaseTracker(purchaseEventToken: String) : PurchaseTracker`

Tracker built-in gửi purchase event tới Adjust: Android dùng `Adjust.verifyAndTrackPlayStorePurchase`, iOS dùng `Adjust.verifyAndTrackAppStorePurchase` (có verify với store trước khi report). Cả sub lẫn in-app dùng chung `purchaseEventToken`. Skip (best-effort) nếu record thiếu `priceAmountMicros`/`priceCurrencyCode`. Mỗi renewal có `transactionId` khác — Adjust tự dedup.

| Param constructor | Kiểu | Mô tả |
|---|---|---|
| `purchaseEventToken` | `String` | Adjust event token (tạo trong Adjust dashboard). |

⚠️ Với backend RevenueCat: nếu đã bật integration Adjust server-side trên dashboard RC thì đừng truyền tracker này — double-count revenue.

## Log TAG

Module log với TAG `WayPay/Manager` (proxy/manager) và `WayPay/AdjustTracker`. Toàn bộ quy ước TAG của WaySDK: xem `wayCore/API.md`.
