# WayPay — Public API

Module cross-platform (Android + iOS) cho in-app purchase / subscription.

- **Android:** Google Play Billing Library v7+.
- **iOS:** StoreKit 1 (iOS 12+, accessible từ Kotlin/Native qua cinterop).

> ⚠️ Implementation iOS dùng **StoreKit 1**, không phải StoreKit 2. SK2 là Swift-only,
> không có Obj-C bridge nên Kotlin/Native cinterop không access được. Xem mục
> [Ghi chú riêng từng nền tảng](#ghi-chú-riêng-từng-nền-tảng) cho chi tiết các SK2
> feature không có sẵn và workaround.

---

## Mục lục

1. [Bắt đầu nhanh (Quick start)](#bắt-đầu-nhanh)
2. [Public API — `AppPurchase`](#public-api--apppurchase)
3. [Data models](#data-models)
4. [Hooks (Verifier & Tracker)](#hooks)
5. [Recipe theo use case](#recipe-theo-use-case)
6. [Xử lý lỗi](#xử-lý-lỗi)
7. [Ghi chú riêng từng nền tảng](#ghi-chú-riêng-từng-nền-tảng)
8. [Migration từ API cũ](#migration-từ-api-cũ)

---

## Bắt đầu nhanh

```kotlin
val pay = AppPurchaseManager.getInstance()

// 1. Connect 1 lần khi app start
pay.connect(PurchaseConfig(
    subscriptionIds   = setOf("premium_monthly", "premium_yearly"),
    nonConsumableIds  = setOf("remove_ads"),
    consumableIds     = setOf("coin_100", "coin_500"),
))

// 2. Hiển thị danh sách
val subs   = pay.queryProducts(ProductType.SUBS)
val inApps = pay.queryProducts(ProductType.INAPP)

// 3. Mua
when (val r = pay.purchase("premium_monthly")) {
    // PurchaseRecord
    is Result.Success -> showThankYou(r.getOrThrow())
    is Result.Failure -> when (val e = r.exceptionOrNull()) {
        PurchaseError.UserCanceled -> { /* im lặng */ }
        else                       -> showError(e)
    }
}

// 4. Nút Restore (iOS bắt buộc)
pay.restorePurchases()

// 5. Lắng nghe renewal / external
launch {
    pay.purchaseEvents.collect { event -> handle(event) }
}

// 6. Quan sát trạng thái subscription để toggle UI / ads
launch {
    pay.isSubscriptionAsFlow.collect { isPremium ->
        if (isPremium) hideAds() else showAds()
    }
}
```

---

## Public API — `AppPurchase`

Lấy instance qua `AppPurchaseManager.getInstance()`. Là singleton, thread-safe.

### Properties (state)

| Property | Kiểu | Ý nghĩa |
|---|---|---|
| `isReady` | `StateFlow<Boolean>` | Billing service đã connect chưa. Android = `BillingClient.isReady`. iOS = `true` sau khi connect xong. |
| `isSubscription` | `Boolean` | Snapshot: user có ít nhất 1 sub active không. Tiện cho check đồng bộ (vd: gating tính năng, ẩn ads). |
| `isSubscriptionAsFlow` | `StateFlow<Boolean>` | Phiên Flow của `isSubscription` — dùng cho UI reactive. |
| `activeEntitlements` | `StateFlow<Set<String>>` | Tập productId user đang sở hữu (subscription active + non-consumable đã mua). KHÔNG chứa consumable. |
| `purchaseEvents` | `SharedFlow<PurchaseEvent>` | Stream tất cả event: mua mới (`USER_INITIATED`), restore (`RESTORE`), renewal & external (cả hai gộp vào `EXTERNAL`). Không replay — subscribe sớm. |

### Methods

#### `suspend fun connect(config: PurchaseConfig)`
Khởi tạo connection và đăng ký productId.

- **Idempotent** — gọi lại không re-connect, chỉ cập nhật config.
- Suspend cho đến khi billing service connect xong (Android) hoặc trả về ngay (iOS).
- Sau khi return, `isReady` phản ánh kết quả.

| Param | Mô tả |
|---|---|
| `config` | Xem [`PurchaseConfig`](#purchaseconfig). |

#### `fun disconnect()`
Đóng connection, release resource. Sau khi gọi, mọi hàm khác trả lỗi/empty.

#### `suspend fun queryProducts(type: ProductType): List<ProductInfo>`
Query detail sản phẩm từ store (tên, giá, mô tả).

| Param | Mô tả |
|---|---|
| `type` | `ProductType.INAPP` (consumable + non-consumable) hoặc `ProductType.SUBS`. |

**Trả về:** danh sách `ProductInfo`. Empty list nếu lỗi — **KHÔNG throw**.

#### `suspend fun purchase(productId: String): Result<PurchaseRecord>`
Mua sản phẩm. Module tự xác định loại từ config:
- Consumable → tự `consume` (Android) / `finish` (iOS).
- Non-consumable / Subscription → tự `acknowledge` / `finish`.

| Param | Mô tả |
|---|---|
| `productId` | ID đã đăng ký trong config (1 trong 3 set). Nếu chưa đăng ký → `Result.failure(NotRegistered)`. |

**Trả về:**
- `Result.success(PurchaseRecord)` khi mua + (nếu có) verify + finalize thành công.
- `Result.failure(PurchaseError)` nếu fail. Phân biệt user cancel vs lỗi qua `exceptionOrNull() as? PurchaseError`.

Kết quả đồng thời được emit qua `purchaseEvents`.

#### `suspend fun restorePurchases(): List<PurchaseRecord>`
Khôi phục purchase đã mua trước đó (cài lại app, đổi máy).

⚠️ **iOS bắt buộc có UI button gọi hàm này** (Apple Guideline 3.1.1). Không có sẽ bị reject.

Mỗi purchase được khôi phục sẽ:
- Verify (nếu có verifier).
- Acknowledge/finish nếu chưa.
- Thêm vào `activeEntitlements`.
- Phát qua `purchaseEvents` với `Source.RESTORE`.

**Trả về:** non-consumable & subscription đang active. Consumable không restore được.

#### `fun openSubscriptionSettings()`
Mở trang quản lý subscription. User có thể cancel / upgrade / downgrade từ đây.

- **Android:** deep link `https://play.google.com/store/account/subscriptions` (mở Play Store app).
- **iOS:** mở universal link `https://apps.apple.com/account/subscriptions` qua `UIApplication.openURL` (user rời khỏi app sang Settings). SK2 có `AppStore.showManageSubscriptions(in:)` cho in-app sheet, nhưng SK1 không có API tương đương.

> **Lưu ý:** đây KHÔNG phải `restorePurchases()` — hai chức năng khác hẳn:
> - `openSubscriptionSettings`: cho user **cancel/thay đổi** sub hiện có.
> - `restorePurchases`: **phục hồi** entitlement sang device/install mới.

---

## Data models

### `PurchaseConfig`

```kotlin
data class PurchaseConfig(
    val subscriptionIds: Set<String>   = emptySet(),
    val consumableIds:   Set<String>   = emptySet(),
    val nonConsumableIds:Set<String>   = emptySet(),
    val verifier:        PurchaseVerifier? = null,
    val tracker:         PurchaseTracker?  = null,
    val isOfferPersonalized: Boolean   = false,
)
```

| Field | Mô tả |
|---|---|
| `subscriptionIds` | Subscription IDs (gói tự gia hạn). |
| `consumableIds` | In-app tiêu hao: coin, gem... mua lại nhiều lần. |
| `nonConsumableIds` | In-app không tiêu hao: remove ads, unlock feature... 1 lần dùng mãi. |
| `verifier` | Hook verify backend, xem [`PurchaseVerifier`](#purchaseverifier). |
| `tracker` | Hook analytics, xem [`PurchaseTracker`](#purchasetracker). |
| `isOfferPersonalized` | EU regulation. Set `true` CHỈ KHI giá thực sự cá nhân hoá theo user. |

⚠️ **Không trùng ID giữa 3 set.** Nếu trùng, behavior undefined.

### `ProductType`

```kotlin
enum class ProductType { INAPP, SUBS }
```

- `INAPP` đại diện cho cả consumable & non-consumable. Việc phân biệt loại nào nằm ở `PurchaseConfig`.
- `SUBS` cho subscription.

### `ProductInfo`

```kotlin
data class ProductInfo(
    val ids: List<String>,
    val type: ProductType,
    val title: String,
    val description: String?,
    val price: Money,
    val entitlementActive: Boolean,
    val isSubscribed: Boolean? = null,    // chỉ SUBS
    val subscription: SubscriptionInfo? = null,  // null với INAPP
)
```

- `entitlementActive`: user đang sở hữu hay đang active sub.
- `subscription`: detail bổ sung (chu kỳ, trial, intro price...) cho SUBS.

### `Money`

```kotlin
data class Money(
    val amountMicros: Long,    // 1_000_000 = 1 unit
    val currencyCode: String,  // ISO-4217: "USD", "VND"...
    val formatted: String? = null,  // "$4.99"
)
```

### `SubscriptionInfo`

```kotlin
data class SubscriptionInfo(
    val periodISO8601: String? = null,        // "P1M", "P1Y"...
    val trialPeriodISO8601: String? = null,   // "P7D"
    val introPrice: Money? = null,
    val renewalState: RenewalState = RenewalState.UNKNOWN,
    val autoRenewing: Boolean? = null,
    val expiryTimeMs: Long? = null,
)
```

### `PurchaseRecord`

Đại diện 1 purchase đã hoàn tất.

```kotlin
data class PurchaseRecord(
    val productId: String,
    val productType: ProductType,
    val transactionId: String,
    val originalTransactionId: String? = null,
    val purchaseTimeMs: Long,
    val purchaseToken: String,
    val signature: String? = null,        // Android only
    val originalJson: String? = null,     // Android only
    val priceAmountMicros: Long? = null,
    val priceCurrencyCode: String? = null,
    val platform: Platform,
)
```

| Field | Mô tả |
|---|---|
| `transactionId` | Android: `orderId`. iOS: `Transaction.id`. Mỗi renewal có ID khác. |
| `originalTransactionId` | iOS: `Transaction.originalID` (giữ nguyên qua các renewal). Android = `transactionId`. |
| `purchaseToken` | Android: `Purchase.purchaseToken` để gửi Google Play Developer API. **iOS: App Store receipt blob đã base64 (toàn bộ transaction history của user)** — KHÔNG phải JWS per-transaction như SK2. Backend phải decode receipt rồi match `transactionId`. |
| `signature` / `originalJson` | Android only, để verify offline nếu cần. |
| `priceAmountMicros` / `priceCurrencyCode` | Giá tại thời điểm mua (cho tracker revenue). |
| `platform` | `ANDROID` hoặc `IOS`. |

### `PurchaseEvent`

```kotlin
sealed class PurchaseEvent {
    data class Success(val record: PurchaseRecord, val source: Source) : PurchaseEvent()
    data class Pending(val productId: String) : PurchaseEvent()
    data class Failed(val productId: String?, val error: PurchaseError) : PurchaseEvent()
    enum class Source { USER_INITIATED, RESTORE, RENEWAL, EXTERNAL }
}
```

`Source` cho biết event đến từ đâu:
- `USER_INITIATED`: caller gọi `purchase()` và productId khớp transaction trả về.
- `RESTORE`: caller gọi `restorePurchases()`.
- `EXTERNAL`: mọi event không khớp 2 case trên — auto-renewal, mua trên thiết bị khác, Family Sharing, pending → approved.
- `RENEWAL`: **reserved**. Hiện cả Android Billing v7 và StoreKit 1 đều không cung cấp signal để phân biệt renewal với event external khác, nên impl hiện tại classify renewal là `EXTERNAL`. Caller cần phân biệt thì check `record.originalTransactionId != record.transactionId` (iOS) hoặc đối chiếu purchase timestamp với cycle trước.

### `PurchaseError`

Sealed class extends `Throwable`. Dùng với `Result.failure`:

| Object | Khi nào |
|---|---|
| `UserCanceled` | User đóng dialog mua. **Không nên show error.** |
| `NetworkError` | Lỗi network. |
| `BillingUnavailable` | Service không khả dụng / parental control disable. |
| `AlreadyOwned` | User đã sở hữu sản phẩm. |
| `ItemUnavailable` | Product không tồn tại / chưa publish. |
| `PendingState` | Đang chờ approval. Show "Đang chờ duyệt". |
| `VerifyFailed` | Verifier trả false hoặc throw. |
| `NotConnected` | Chưa gọi `connect()` hoặc đã `disconnect()`. |
| `NotRegistered` | productId chưa khai báo trong config. |
| `ServiceError(code, msg)` | Lỗi từ billing service có code. |
| `Unknown(cause)` | Lỗi không phân loại. |

---

## Hooks

### `PurchaseVerifier`

Verify với backend của app (optional).

```kotlin
fun interface PurchaseVerifier {
    suspend fun verify(record: PurchaseRecord): Boolean
}
```

**Behavior:**
- Trả `true` → module tiếp tục acknowledge/finish.
- Trả `false` hoặc throw → KHÔNG finish. Google/Apple sẽ tự refund sau 3 ngày.

**Ví dụ:**

```kotlin
val verifier = PurchaseVerifier { record ->
    val response = httpClient.post("https://api.myapp.com/verify-purchase") {
        setBody(mapOf(
            "platform"      to record.platform.name,
            "productId"     to record.productId,
            "purchaseToken" to record.purchaseToken,
            "signature"     to record.signature,
        ))
    }
    response.status == HttpStatusCode.OK
}

pay.connect(
  PurchaseConfig(
    subscriptionIds = setOf("..."),
    verifier = verifier)
)
```

**Backend phải làm gì:**
- **Android:** gọi `purchases.products.get` / `purchases.subscriptions.get` của Google Play Developer API với `purchaseToken`.
- **iOS:** `purchaseToken` là **App Store receipt blob** (base64) chứa toàn bộ tx history. Backend phải:
  1. Gọi `/verifyReceipt` (legacy) hoặc App Store Server API `/inApps/v1/transactions/{transactionId}` (preferred) với receipt + tra `transactionId` để xác định tx hiện tại.
  2. Đọc `expires_date` (subscriptions), `revocation_date`, `cancellation_date` để confirm còn valid.
  3. (Không decode JWS như SK2 — SK1 không có JWS per-transaction.)

### `PurchaseTracker`

Analytics tracking sau khi verify + finalize thành công.

```kotlin
fun interface PurchaseTracker {
    suspend fun track(record: PurchaseRecord)
}
```

#### Built-in: `AdjustPurchaseTracker` (cross-platform)

Module cung cấp sẵn `AdjustPurchaseTracker` cho cả Android và iOS:

```kotlin
pay.connect(PurchaseConfig(
    subscriptionIds  = setOf("premium_monthly"),
    nonConsumableIds = setOf("remove_ads"),
    consumableIds    = setOf("coin_100"),
    tracker = AdjustPurchaseTracker(inAppPurchaseToken = "abc123"),
))
```

| Loại purchase | Android API | iOS API |
|---|---|---|
| Subscription | `Adjust.trackPlayStoreSubscription` | `Adjust.trackAppStoreSubscription` |
| In-app (consumable + non-consumable) | `Adjust.verifyAndTrackPlayStorePurchase` | `Adjust.verifyAndTrackAppStorePurchase` |

Subscription tracking dùng API chuyên biệt (KHÔNG cần token). In-app tracking dùng
`ADJEvent` + `inAppPurchaseToken` config trong Adjust dashboard.

Yêu cầu trong `PurchaseRecord`:
- `priceAmountMicros` + `priceCurrencyCode`: bắt buộc để revenue tracking. Module
  tự populate từ `ProductDetails`/`SKProduct` cache khi finalize.
- `signature` + `purchaseToken`: bắt buộc cho Android subscription (đã có sẵn từ
  `Purchase` của Billing Library).

Dependencies (cấu hình sẵn trong `wayPay/build.gradle.kts`):
- Android: `com.adjust.sdk:adjust-android`
- iOS: cocoapods `Adjust` (qua `cocoapods` block).

#### Custom tracker — Adjust + Firebase combined:

```kotlin
// androidMain
class AdjustFirebaseTracker(
    private val inAppToken: String,
) : PurchaseTracker {
    override suspend fun track(record: PurchaseRecord) {
        val amount = (record.priceAmountMicros ?: return) / 1_000_000.0
        val currency = record.priceCurrencyCode ?: return

        // Adjust
        when (record.productType) {
            ProductType.SUBS -> Adjust.trackPlayStoreSubscription(
                AdjustPlayStoreSubscription(
                    record.priceAmountMicros,
                    currency,
                    record.productId,
                    record.transactionId,
                    record.signature.orEmpty(),
                    record.purchaseToken,
                ).apply { purchaseTime = record.purchaseTimeMs }
            )
            ProductType.INAPP -> {
                val event = AdjustEvent(inAppToken)
                event.setRevenue(amount, currency)
                Adjust.verifyAndTrackPlayStorePurchase(event) { /* result */ }
            }
        }

        // Firebase — dùng tên chuẩn để GA4 tự nhận diện revenue.
        Firebase.analytics.logEvent(FirebaseAnalytics.Event.PURCHASE, mapOf(
            FirebaseAnalytics.Param.TRANSACTION_ID to record.transactionId,
            FirebaseAnalytics.Param.VALUE          to amount,
            FirebaseAnalytics.Param.CURRENCY       to currency,
            FirebaseAnalytics.Param.ITEM_ID        to record.productId,
        ))
    }
}
```

**Composite (nhiều tracker):**

```kotlin
class CompositeTracker(private val list: List<PurchaseTracker>) : PurchaseTracker {
    override suspend fun track(record: PurchaseRecord) {
        list.forEach { runCatching { it.track(record) } }
    }
}
```

> ⚠️ Lỗi từ tracker chỉ được log, KHÔNG fail purchase.

---

## Recipe theo use case

### Premium gating (subscription)

```kotlin
@Composable
fun MyScreen() {
    val isPremium by pay.isSubscriptionAsFlow.collectAsState()
    if (isPremium) PremiumContent() else FreeContent(showAds = true)
}
```

### Mua coin (consumable) + lưu state

```kotlin
suspend fun buy100Coins() {
    when (val r = pay.purchase("coin_100")) {
        is Result.Success -> {
            // Quan trọng: lưu state TRƯỚC khi return cho user, phòng crash.
            userRepo.addCoins(100)
            showSuccess()
        }
        is Result.Failure -> {
            when (val e = r.exceptionOrNull()) {
                PurchaseError.UserCanceled  -> { /* im lặng */ }
                PurchaseError.PendingState  -> showPending()
                else                        -> showError(e)
            }
        }
    }
}
```

### Restore button (iOS bắt buộc)

```kotlin
Button(onClick = {
    scope.launch {
        val restored = pay.restorePurchases()
        if (restored.isEmpty()) toast("Không có gói nào để khôi phục")
        else toast("Đã khôi phục ${restored.size} gói")
    }
}) { Text("Khôi phục mua hàng") }
```

### Lắng nghe renewal / external

Auto-renewal hiện được gộp vào `Source.EXTERNAL` (xem [PurchaseEvent](#purchaseevent)).
Caller cần phân biệt thì so sánh `transactionId` với `originalTransactionId`:

```kotlin
init {
    scope.launch {
        pay.purchaseEvents.collect { event ->
            when (event) {
                is PurchaseEvent.Success -> when (event.source) {
                    Source.EXTERNAL -> {
                        val r = event.record
                        val isRenewal = r.originalTransactionId != null &&
                                        r.originalTransactionId != r.transactionId
                        if (isRenewal) Log.d("Sub renewed: ${r.productId}")
                        else Log.d("External purchase: ${r.productId}")
                    }
                    else -> {}
                }
                is PurchaseEvent.Pending -> showPendingBanner(event.productId)
                is PurchaseEvent.Failed  -> {}
            }
        }
    }
}
```

---

## Xử lý lỗi

```kotlin
when (val r = pay.purchase(productId)) {
    is Result.Success -> grantEntitlement(r.getOrThrow())
    is Result.Failure -> when (val e = r.exceptionOrNull()) {
        PurchaseError.UserCanceled       -> {} // im lặng
        PurchaseError.NetworkError       -> showRetry()
        PurchaseError.BillingUnavailable -> showSorry()
        PurchaseError.AlreadyOwned       -> showAlreadyOwned()
        PurchaseError.PendingState       -> showPending()
        PurchaseError.VerifyFailed       -> showVerifyError()
        PurchaseError.NotConnected       -> reconnect()
        PurchaseError.NotRegistered      -> Log.e("Config bug: $productId chưa đăng ký")
        is PurchaseError.ServiceError    -> Log.e("Service ${e.code}: ${e.debugMessage}")
        else                             -> Log.e("Unknown", e)
    }
}
```

---

## Ghi chú riêng từng nền tảng

### Android

- Dùng Google Play Billing v7+ (`com.android.billingclient:billing-ktx`).
- Activity được lấy tự động qua `LifecycleProvider.activity` — đảm bảo có foreground Activity khi gọi `purchase()`.
- `enableAutoServiceReconnection()` đã bật — service tự reconnect khi disconnect.
- `purchaseToken` để verify với Google Play Developer API.

### iOS — ⚠️ Implementation dùng **StoreKit 1** (không phải StoreKit 2)

Module hiện tại implement iOS bằng **StoreKit 1** (`platform.StoreKit.*` qua Kotlin/Native
cinterop). Đây là lựa chọn kỹ thuật vì:

- StoreKit 2 là Swift-only API — `Product`, `Transaction`, `VerificationResult`... đều
  là Swift struct với generic associated values, KHÔNG có Obj-C bridge.
- Kotlin/Native cinterop chỉ access được Obj-C, không access được Swift-only.
- Để dùng SK2 từ KMP cần Swift bridge framework riêng (xem mục [Upgrade path lên SK2](#upgrade-path-lên-storekit-2) bên dưới).

StoreKit 1 vẫn được Apple support đầy đủ trên iOS 12+ (kể cả iOS 18 mới nhất), không
deprecated, và đủ cho 100% use case IAP/subscription thông thường.

#### Các API StoreKit 2 KHÔNG có sẵn ở implementation này

| SK2 feature | Có ở SK1? | Workaround |
|---|---|---|
| `Product.purchase()` async/await | ❌ Bằng callback observer | Wrapped trong suspend function qua `CompletableDeferred` |
| `Product.products(for:)` async | ❌ Bằng delegate | `SKProductsRequest` + `SKProductsRequestDelegate` wrap thành suspend |
| `Transaction.jwsRepresentation` (JWS per-tx) | ❌ | App Store receipt blob (toàn bộ tx) — backend decode bằng App Store Server API |
| `Transaction.updates` async iterator | ❌ | `SKPaymentTransactionObserver` callback → emit qua `purchaseEvents` |
| `Transaction.currentEntitlements` | ❌ | Persist local trong NSUserDefaults + `restoreCompletedTransactions` |
| `Transaction.subscriptionStatus` (auto-renew realtime) | ❌ | Hỏi backend qua App Store Server API |
| `Product.SubscriptionInfo.isEligibleForIntroOffer` | ❌ | Hỏi backend hoặc App Store Server API |
| `AppStore.showManageSubscriptions(in:)` (in-app sheet) | ❌ | Mở universal link `apps.apple.com/account/subscriptions` (rời khỏi app) |
| `AppStore.sync()` | ❌ | `SKPaymentQueue.restoreCompletedTransactions()` (prompt password) |
| Built-in JWS signature verify | ❌ | Backend verify qua App Store Server API |
| `Transaction.beginRefundRequest()` | ❌ | Không có cách native từ SK1 — phải redirect user sang Apple support |
| `RenewalState.AUTO_RENEW_ON/OFF` realtime | ❌ — luôn `UNKNOWN` | Cần App Store Server Notifications V2 vào backend |

#### Đặc điểm khác

- `purchaseToken` trong [`PurchaseRecord`](#purchaserecord) là **App Store receipt
  base64** (toàn bộ tx history), KHÔNG phải JWS per-transaction như SK2. Backend phải
  decode receipt + match `transactionId`.
- `signature` và `originalJson` luôn `null` trên iOS (đó là field Android).
- `RenewalState` cho subscription luôn `UNKNOWN` — SK1 không expose auto-renew state
  client-side. Lấy qua App Store Server API nếu cần.
- `expiryTimeMs` trong [`SubscriptionInfo`](#subscriptioninfo) (trả từ `queryProducts`)
  luôn `null` — SK1 không expose qua API, chỉ có trong receipt.
- `restorePurchases()` BẮT BUỘC có UI button (Apple App Review Guideline 3.1.1).
- Auto-renewal & external transaction event được phát qua `purchaseEvents` với
  `Source.EXTERNAL` (xem [Source](#purchaseevent) — `RENEWAL` chưa được phân biệt).

#### Entitlement persistence & expiry tracking (iOS)

Vì SK1 không có `Transaction.currentEntitlements` và `Transaction.subscriptionStatus`,
module tự track entitlement state local. Chi tiết hành vi:

- **Storage**: `NSUserDefaults` key `com.waypay.activeEntitlements`, format JSON
  `[{productId, expiryMs}, ...]`. User mở lại app vẫn nhớ trạng thái mà không cần gọi
  `restorePurchases()`.
- **Subscription expiry**: tính bằng `transactionDate + product.subscriptionPeriod`
  qua `NSCalendar.dateByAddingComponents` — tôn trọng độ dài thật của tháng (28–31
  ngày) và năm nhuận (366 ngày). Auto-renewal callback từ Apple sẽ push expiry tới khi
  có renewal mới.
- **Non-consumable** (mua một lần dùng mãi): expiry = `Long.MAX_VALUE`.
- **Consumable**: KHÔNG persist (bản chất one-shot, app tự quản state).
- **Pruning**: `applyPersistedEntitlements` (gọi mỗi `connect()` và sau mỗi purchase
  thành công) lọc bỏ entry đã hết hạn → state flow `activeEntitlements` và
  `isSubscription` luôn phản ánh đúng thực tế local.
- **Fallback 366 ngày**: khi compute expiry mà thiếu period info (SKProduct cache miss
  + fetch fail, hoặc transaction không có `transactionDate`), expiry được set =
  `now + 366 ngày`. Chọn dài để KHÔNG cắt nhầm yearly subscriber; canceled-monthly
  user sẽ thừa entitlement đến 366 ngày — trade-off chấp nhận được nếu không có
  backend verifier.
- **Migration từ format cũ** (`Set<String>` từ phiên bản v ≤ 2.1.0-alpha02):
  - Sub IDs → expiry = `now + 366 ngày` (cùng grace với fallback trên).
  - Non-sub IDs → `PERMANENT_EXPIRY`.
  - Sau khi migrate, format mới được ghi đè vào `NSUserDefaults`; không lặp lại.

> 💡 **Khi nào cần backend verifier**: nếu app cần (a) revoke entitlement sớm hơn khi
> user refund/cancel hoặc (b) biết chính xác expiry thay vì estimate 366d fallback, hãy
> implement [`PurchaseVerifier`](#purchaseverifier) gọi App Store Server API. Không có
> backend, module dùng best-effort offline tracking mô tả ở trên.

#### Upgrade path lên StoreKit 2

Nếu sau này muốn dùng SK2 (vd: cần JWS verify offline, real-time renewal state):

1. Tạo Swift framework wrap StoreKit 2 với interface `@objc`-compatible.
2. Thêm vào `wayPay/build.gradle.kts`:
   ```kotlin
   cocoapods {
       pod("WayPayStoreKit2Bridge") { /* local hoặc remote pod */ }
   }
   ```
3. Viết `AppPurchaseImpl.ios.kt` mới dùng `cocoapods.WayPayStoreKit2Bridge.*` thay
   `platform.StoreKit.*`.

Interface `AppPurchase` cross-platform không cần đổi — chỉ swap impl.

---

## Migration từ API cũ

| API cũ | API mới |
|---|---|
| `connect(adjustToken, listener, subs, inApps, nonConsumable)` | `connect(PurchaseConfig(...))`. Tracker pass qua `config.tracker`. |
| `registerPurchaseListener(listener)` | `purchaseEvents.collect { ... }`. Không leak listener nữa. |
| `registerSubscriptionProductIds(ids)` | `PurchaseConfig.subscriptionIds`. |
| `registerInAppProductIds(ids)` | `PurchaseConfig.consumableIds`. |
| `registerNonConsumableInAppProductIds(ids)` | `PurchaseConfig.nonConsumableIds`. |
| `purchase(productId, activity)` | `purchase(productId)` — không cần Activity. |
| `cancelPurchase(activity)` | `openSubscriptionSettings()` — rename cho đúng semantic. |
| `isSubscription` | Giữ nguyên. |
| `isSubscriptionAsFlow` | Giữ nguyên. |
| `isAvailable` | `isReady` (StateFlow). |
| `PurchaseListener.onPurchaseSuccess(List<String>)` | `PurchaseEvent.Success(PurchaseRecord, Source)`. Đầy đủ info hơn (transactionId, token, price...). |
| `PurchaseListener.onPurchaseFailure(Throwable)` | `PurchaseEvent.Failed(productId, PurchaseError)`. Phân biệt UserCanceled vs lỗi thật. |
| (không có) | `restorePurchases()` — mới, iOS bắt buộc. |
| Adjust + Firebase tracking hardcoded | Pass `PurchaseTracker` impl qua config. Xem [recipe](#purchasetracker). |

### Breaking changes

1. **`cancelPurchase()` đổi tên** → `openSubscriptionSettings()`.
2. **`PurchaseListener` bị xoá** → thay bằng `purchaseEvents: SharedFlow<PurchaseEvent>`.
3. **`adjustPurchaseInAppToken` không còn ở `connect()`** → pass qua `PurchaseConfig.tracker`.
4. **Adjust tracking không còn auto** — dùng built-in `AdjustPurchaseTracker(token)` qua `PurchaseConfig.tracker` (xem mục `PurchaseTracker` ở trên). Firebase tracking phải tự implement.
5. **`purchase()` không nhận Activity** — module tự lấy qua `LifecycleProvider`.
6. **`purchase()` trả `Result<PurchaseRecord>`** thay vì callback qua listener.
7. **`connect()` thành `suspend`** — không gọi được từ context đồng bộ. App developer phải gọi từ coroutine scope.

### Quan hệ với WayAdKit

`WayAdKit.init()` **KHÔNG** tự connect wayPay nữa (trước đây gọi với `PurchaseConfig()` rỗng, không có productIds, thực tế không track được gì).

App developer phải tự gọi:

```kotlin
// Application.onCreate (hoặc nơi tương đương)
applicationScope.launch {
    AppPurchaseManager.getInstance().connect(
        PurchaseConfig(
            subscriptionIds  = setOf("premium_monthly"),
            nonConsumableIds = setOf("remove_ads"),
            consumableIds    = setOf("coin_100"),
            tracker = AdjustPurchaseTracker(
                inAppPurchaseToken = "your_adjust_inapp_event_token"
            ),
        )
    )
}

// Song song, không cần đợi
WayAdKit.init(wayAdConfig, isDebug)
```

WayAdKit chỉ đọc `appPurchase.isSubscription` để gate ads — nếu wayPay chưa connect (vd: app không bán IAP), giá trị là `false` → ads load bình thường, không break.

> ⚠️ **Breaking change kèm theo:** `WayAdConfig.tokenAdPurchase` cũng đã bị xoá. Adjust IAP token giờ pass thẳng vào `AdjustPurchaseTracker(inAppPurchaseToken = ...)` ở app code, không qua `WayAdConfig` nữa.