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

Với SUBS, xem thêm [Tính giá theo chu kỳ](#tính-giá-theo-chu-kỳ) — extension quy giá các gói về cùng một chu kỳ để so sánh.

### `Money`

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `amountMicros` | `Long` | Giá theo micro-unit (1 đơn vị tiền = 1_000_000 micro). |
| `currencyCode` | `String` | Mã ISO-4217 ("USD", "VND"). |
| `formatted` | `String?` | Chuỗi đã format theo locale store ("$4.99"), có thể `null`. |

### `SubscriptionInfo`

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `periodISO8601` | `String?` | Chu kỳ cơ bản, chuỗi ISO-8601 **thô từ store**: `P1W` / `P4W` / `P1M` / `P1Y`... `null` nếu store không trả chu kỳ. |
| `trialPeriodISO8601` | `String?` | Free trial period nếu có (`P3D`, `P7D`...). |
| `introPrice` | `Money?` | Giá intro/ưu đãi nếu có. |
| `renewalState` | `RenewalState` | Trạng thái auto-renew. |
| `autoRenewing` | `Boolean?` | Dạng bool đơn giản của renewal; `null` nếu không rõ. |
| `expiryTimeMs` | `Long?` | Thời điểm hết hạn (epoch millis) nếu biết. |

Hai field ISO ở trên là dữ liệu thô: mỗi store trả một dạng khác nhau cho **cùng một gói** — Google Play `P1W`/`P4W`, StoreKit có thể `P7D`, RevenueCat quy từ `value + unit`. **Đừng so sánh chúng bằng chuỗi.** Dùng 3 property dẫn xuất dưới đây; chúng là computed property nên không xuất hiện trong JSON khi serialize `SubscriptionInfo` (wire format giữ nguyên).

| Property | Kiểu | Ý nghĩa |
|---|---|---|
| `period` | `SubscriptionPeriod?` | `periodISO8601` đã parse + chuẩn hoá. `null` nếu store không trả chu kỳ hoặc chuỗi sai format. |
| `trialPeriod` | `SubscriptionPeriod?` | `trialPeriodISO8601` đã parse. `null` nếu gói không có free trial. |
| `billingCycle` | `BillingCycle?` | Phân loại chu kỳ cho UI; bằng `period?.cycle`. |

```kotlin
// Phân biệt gói tuần / tháng / năm — cho kết quả giống nhau trên cả 3 backend
val label = when (product.subscription?.billingCycle) {
    BillingCycle.WEEKLY -> "Gói tuần"
    BillingCycle.MONTHLY -> "Gói tháng"
    BillingCycle.YEARLY -> "Gói năm"
    else -> product.subscription?.period?.iso8601.orEmpty()
}

// Sắp xếp gói từ ngắn đến dài
val sorted = subs.sortedBy { it.subscription?.period?.approximateDays ?: 0 }
```

### `SubscriptionPeriod`

Chu kỳ thanh toán đã parse từ ISO-8601. `parse()` chuẩn hoá các dạng **tương đương chính xác** về một biểu diễn duy nhất, nhờ vậy 2 backend trả 2 chuỗi khác nhau vẫn so được bằng `==`:

| Store trả | Sau `parse()` |
|---|---|
| `P1W` / `P7D` | `SubscriptionPeriod(1, WEEK)` |
| `P2W` / `P14D` | `SubscriptionPeriod(2, WEEK)` |
| `P1Y` / `P12M` | `SubscriptionPeriod(1, YEAR)` |
| `P30D` | `SubscriptionPeriod(30, DAY)` — 30 không chia hết cho 7 nên giữ đơn vị ngày |

| Field / Property | Kiểu | Ý nghĩa |
|---|---|---|
| `value` | `Int` | Số đơn vị. |
| `unit` | `PeriodUnit` | Đơn vị thời gian. |
| `iso8601` | `String` | Chuỗi ISO-8601 đã chuẩn hoá (`P1W`) — có thể khác `periodISO8601` gốc nếu store trả dạng tương đương. |
| `approximateDays` | `Int` | Độ dài xấp xỉ theo ngày; dùng để **so sánh / sắp xếp** các gói. |
| `cycle` | `BillingCycle` | Phân loại cho UI. |

⚠️ `approximateDays` quy ước MONTH = 30 và YEAR = 365 ngày. Chỉ dùng để so sánh độ dài chu kỳ, **không** dùng để tính ngày hết hạn — cần expiry thật thì đọc `SubscriptionInfo.expiryTimeMs`.

#### `fun parse(iso8601: String?): SubscriptionPeriod?`

Parse phần ngày của duration ISO-8601 (`P[nY][nM][nW][nD]`). Trả `null` khi: input `null`/rỗng, thiếu `P`, thiếu số hoặc thiếu designator (`P1`), designator lạ (`P1X`), designator trùng (`P1M1M`), có phần giờ (`PT12H`, `P1DT12H`), số quá lớn, hoặc tổng độ dài `<= 0` (`P0D`).

Parser cố ý **lenient** ở vài điểm, vì mục tiêu là nuốt được dữ liệu store trả về chứ không phải validate input người dùng: chấp nhận chữ thường và khoảng trắng thừa (` p1m `), không ép đúng thứ tự component của ISO (`P1M1Y` vẫn parse), và cho phép `W` trộn với `Y/M/D` (ISO strict thì không). Dạng nhiều component (`P1Y6M` — không store nào phát) được gộp về tổng ngày xấp xỉ với `unit = DAY`. Nếu dùng `parse()` để validate chuỗi từ nguồn ngoài store, hãy tự siết thêm ở tầng gọi.

### `PeriodUnit`

| Giá trị | `approximateDays` | `isoSuffix` |
|---|---|---|
| `DAY` | 1 | `D` |
| `WEEK` | 7 | `W` |
| `MONTH` | 30 | `M` |
| `YEAR` | 365 | `Y` |

### `BillingCycle`

Phân loại chu kỳ theo cách người dùng hiểu, suy ra từ `SubscriptionPeriod.approximateDays`. Cố ý **gom các biến thể tương đương về cùng một nhóm** để UI không phải xử lý từng dạng chuỗi — cần chính xác tuyệt đối thì đọc `SubscriptionPeriod.value` + `.unit`.

| Giá trị | Khoảng ngày | Khớp với |
|---|---|---|
| `DAILY` | 1–2 | `P1D` |
| `WEEKLY` | 6–8 | `P1W`, `P7D` |
| `TWO_WEEKS` | 13–16 | `P2W`, `P14D` |
| `MONTHLY` | 26–32 | `P1M`, `P4W`, `P30D` |
| `TWO_MONTHS` | 55–63 | `P2M`, `P8W` |
| `THREE_MONTHS` | 85–95 | `P3M`, `P13W` |
| `FOUR_MONTHS` | 115–125 | `P4M` |
| `SIX_MONTHS` | 175–190 | `P6M`, `P26W` |
| `YEARLY` | 355–375 | `P1Y`, `P12M`, `P52W` |
| `OTHER` | còn lại | `P10D`, `P5M`... |

`companion object` có `ofApproximateDays(days: Int): BillingCycle` — map thẳng số ngày sang nhóm, dùng khi độ dài chu kỳ đến từ nguồn khác `SubscriptionPeriod`.

Chu kỳ các store thực sự bán: **Google Play** 1 tuần / 4 tuần / 1–2–3–4–6 tháng / 1 năm; **App Store** 1 tuần / 1–2–3–6 tháng / 1 năm (không có 4 tuần và 4 tháng). Không store nào bán gói 2 tuần — `TWO_WEEKS` chỉ xảy ra với backend tự triển khai.

### Tính giá theo chu kỳ

Extension của `ProductInfo` (package `com.waypay.model`) để quy giá các gói về **cùng một chu kỳ** rồi so sánh — dùng cho paywall kiểu "chỉ $0.58/tuần", "tiết kiệm 71%".

#### `fun ProductInfo.pricePer(unit: PeriodUnit = PeriodUnit.WEEK): Money?`
#### `fun ProductInfo.pricePer(period: SubscriptionPeriod): Money?`

Quy giá 1 chu kỳ thanh toán sang chu kỳ đích, mẫu số chung là `approximateDays`. **Mặc định là 1 tuần** — đơn vị nhỏ nhất cả 2 store đều bán, nên gọi `pricePer()` trên mọi gói đều ra "giá mỗi tuần" so được với nhau.

Một hàm trả lời được cả 2 chiều: quy **xuống** để so gói, quy **lên** để biết tổng chi phí.

| Gói | Gọi | Kết quả |
|---|---|---|
| Năm `$29.99` (`P1Y`) | `pricePer()` | `$0.575` mỗi tuần |
| Năm `$29.99` (`P1Y`) | `pricePer(PeriodUnit.MONTH)` | `$2.46` mỗi tháng |
| Tuần `$1.99` (`P1W`) | `pricePer(PeriodUnit.YEAR)` | `$103.76` — tổng phải trả trong 1 năm |

Trả `null` khi `type != SUBS` (kiểm cả khi product INAPP lỡ mang theo `subscription`), store không trả chu kỳ (`subscription?.period == null`), hoặc phép nhân **tràn `Long`**. Phân số được rút gọn trước khi nhân nên mọi mức giá thực tế đều nằm trong tầm; `null` ở đây nghĩa là input phi lý, không phải kết quả sai âm thầm.

Cơ sở tính là `ProductInfo.price` — **giá gia hạn cơ bản**, không phải `introPrice` hay giá sau trial. `Money.currencyCode` giữ nguyên; `Money.formatted` được format lại theo **locale thiết bị** (ký hiệu, vị trí, dấu phân cách nhóm) nhưng **số chữ số thập phân lấy theo chính đồng tiền đó**, không theo locale: `VND`/`JPY` → 0 chữ số, `USD` → 2. Đồng tiền không định nghĩa scale (`XXX`) → 2. Làm tròn ở đúng scale đó, half-even, tính trên số nguyên micros nên không có sai số dấu phẩy động.

| micros | `USD` | `VND` |
|---|---|---|
| `57_342_465_753` | `$57,342.47` | `₫57,342` |
| `499_999` | `$0.50` | `₫0` |

Vì locale thiết bị ≠ locale của store nên chuỗi có thể lệch nhẹ so với `price.formatted` gốc. Trả `null` nếu `currencyCode` không phải mã ISO-4217 hợp lệ.

#### `fun ProductInfo.savingsPercentAgainst(baseline: ProductInfo): Int?`

Phần trăm rẻ hơn của gói này so với `baseline`, sau khi quy cả hai về cùng độ dài. Gói năm so gói tuần ở ví dụ trên → `71`. Số **âm** nghĩa là đắt hơn baseline (gói tuần so gói năm → `-246`).

Trả `null` khi một trong hai không phải `SUBS`, thiếu chu kỳ, **khác `currencyCode`**, tràn `Long`, hoặc kết quả không lọt vào range `Int` (gói đắt hơn baseline tới mức phi lý — không wrap thành số sai).

```kotlin
val subs = pay.queryProducts(ProductType.SUBS)
val cheapestPerWeek = subs.minByOrNull { it.pricePer()?.amountMicros ?: Long.MAX_VALUE }
val baseline = subs.firstOrNull { it.subscription?.billingCycle == BillingCycle.WEEKLY }

subs.forEach { p ->
    val perWeek = p.pricePer()?.formatted            // "$0.58"
    val save = baseline?.let { p.savingsPercentAgainst(it) }?.takeIf { it > 0 }
    render(p.price.formatted, perWeek, save?.let { "Tiết kiệm $it%" })
}
```

⚠️ Kết quả kế thừa quy ước MONTH = 30 / YEAR = 365 ngày của `approximateDays`, nên là **con số marketing để so sánh**, không phải số tiền store sẽ thực sự charge. Số tiền charge luôn là `ProductInfo.price`.

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
