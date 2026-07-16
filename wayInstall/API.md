# wayInstall — Public API

Module `wayInstall` (artifact `way-sdk:way-install`, iOS framework `wayInstallKit`) là module Kotlin Multiplatform chịu trách nhiệm xác định **nguồn cài đặt (install attribution)** của user: paid hay organic, đến từ network nào. Module gom tín hiệu từ 4 provider — **Adjust**, **Google Play Install Referrer**, **Firebase**, **Apple Search Ads (AdServices)** — cache lại từng nguồn, hợp nhất thành một "verdict" duy nhất, và tự động log kết quả lên **GA4 (Firebase Analytics)** qua event `source_user` / `source_user_updated` cùng các user property `way_attribution`, `way_is_organic`, `way_primary_provider`.

Phụ thuộc module (theo `build.gradle.kts`): `api(":lifecycle_kmp")` (lấy Android `Context` qua `LifecycleProvider`), `api(":adjust_kmp")` (Adjust SDK), và các thư viện: kotlinx-coroutines, ktor client, GitLive firebase-analytics, multiplatform-settings, Play `installreferrer` (Android). Cache lưu bền vững qua `SharedPreferences` (Android) / `NSUserDefaults` (iOS) với tên store `"UserInstallKit"`.

## Khởi tạo

Gọi `init()` càng sớm càng tốt (sau khi Firebase và Adjust đã được khởi tạo, và `lifecycle_kmp` đã có `Context` trên Android):

```kotlin
UserInstallKit.init(
    config = UserInstallConfig(
        adjustAttributionTimeoutMs = 10_000L,
        installReferrerTimeoutMs = 5_000L,
        appleAdServicesTimeoutMs = 5_000L,
        attributionLogTimeoutMs = 60_000L,
        attributionUpgradeWindowMs = 60_000L,
        isDebug = BuildConfig.DEBUG,
    )
) {
    // onComplete: cả 4 provider đã warm-up xong (fetch hoặc lấy từ cache)
}

// Sau đó, ở bất kỳ đâu:
UserInstallKit.resolveAttribution { verdict ->
    if (verdict.attribution == Attribution.PAID) { /* ... */ }
}
```

`init()` làm 3 việc: (1) lưu `config`; (2) chạy **song song** warm-up cả 4 provider trên `Dispatchers.IO` — provider nào đã có cache thì bỏ qua fetch, provider nào fetch thành công thì ghi cache; (3) chạy nền **attribution logger** tự log verdict lên GA4 (xem "Tự động log GA4" bên dưới). `onComplete` được gọi sau khi cả 4 warm-up kết thúc (không chờ logger). Gọi lại `init()` an toàn: chỉ fetch lại những provider chưa có cache.

Lưu ý: các hàm `fromXxx`/`resolveAttribution` vẫn chạy được khi chưa gọi `init()` (dùng `UserInstallConfig()` mặc định), nhưng khi đó việc tự động log GA4 sẽ không chạy.

## Public API

### `UserInstallKit` (object)

Entry point duy nhất của module. Mọi công việc nền chạy trên `CoroutineScope(SupervisorJob() + Dispatchers.IO)` nội bộ.

#### `fun init(config: UserInstallConfig = UserInstallConfig(), onComplete: () -> Unit = {})`

Khởi tạo module như mô tả ở trên.

| Param | Kiểu | Mô tả |
|---|---|---|
| `config` | `UserInstallConfig` | Cấu hình timeout và debug log. |
| `onComplete` | `() -> Unit` | Gọi (trên thread IO) khi cả 4 provider warm-up xong. |

**Trả về:** `Unit` (bất đồng bộ, không block).

#### `fun fromAdjust(forceRefresh: Boolean = false, onResult: (InstallSource) -> Unit)`
#### `fun fromInstallReferrer(forceRefresh: Boolean = false, onResult: (InstallSource) -> Unit)`
#### `fun fromFirebase(forceRefresh: Boolean = false, onResult: (InstallSource) -> Unit)`
#### `fun fromAppleSearchAds(forceRefresh: Boolean = false, onResult: (InstallSource) -> Unit)`

Lấy `InstallSource` thô của từng provider riêng lẻ. Cả 4 hàm cùng hành vi:

- **Cache-first:** nếu `forceRefresh = false` và đã có cache → `onResult` được gọi **đồng bộ ngay trên thread gọi** với giá trị cache. Ngược lại fetch trên thread IO rồi callback trên IO.
- **Lỗi/null:** nếu fetch ném exception hoặc trả `null` (timeout, thiếu context, API không khả dụng) → callback nhận `InstallSource.Unknown.copy(provider = <provider>)` và kết quả **không được cache** (lần gọi sau sẽ fetch lại). Fetch thành công thì được ghi cache bền vững — kể cả kết quả "không có tín hiệu" mà provider trả về hợp lệ.
- Callback luôn được gọi đúng một lần cho mỗi lần gọi hàm.

| Param | Kiểu | Mô tả |
|---|---|---|
| `forceRefresh` | `Boolean` | `true` = bỏ qua cache, fetch lại từ provider. |
| `onResult` | `(InstallSource) -> Unit` | Nhận kết quả (không bao giờ null; lỗi → `Unknown`). |

#### `fun resolveAttribution(forceRefresh: Boolean = false, onResult: (AttributionVerdict) -> Unit)`

Hợp nhất tín hiệu của các provider **đã sẵn sàng** thành một `AttributionVerdict`. Nếu `forceRefresh = true`, fetch lại song song cả 4 provider (kết quả thành công được ghi đè cache) rồi mới resolve; nếu `false`, resolve ngay từ cache hiện có — provider chưa ready sẽ không tham gia. Callback chạy trên thread IO. Hàm thuần đọc trạng thái nên gọi lại nhiều lần idempotent (với `forceRefresh=false`).

Luật hợp nhất (theo `InstallSourceResolver`):

- Chỉ 3 provider **trust** tham gia quyết định: `ADJUST`, `INSTALL_REFERRER`, `APPLE_SEARCH_ADS`. `FIREBASE` không bao giờ quyết định verdict.
- Tín hiệu có `confidence == LOW` bị bỏ qua.
- Có ≥1 tín hiệu paid → `PAID`; nếu ≥2 tín hiệu paid → `confidence = HIGH`, `reason = "paid_corroborated"`, ngược lại lấy confidence của tín hiệu chính, `reason = "paid_<provider>"`.
- Không có paid nhưng có tín hiệu organic → `ORGANIC`; ≥2 nguồn → `HIGH`/`"organic_corroborated"`, 1 nguồn → `MEDIUM`/`"organic_<provider>"`.
- Không có tín hiệu nào → `UNKNOWN`, `confidence = LOW`, `reason` cho biết đang chờ provider nào (`"no_trust_provider_ready"`, `"waiting_for_trust:<danh sách>"`, hoặc `"all_trust_low_signal"`).
- Provider chính (`primaryProvider`, và `network` khi paid) chọn theo thứ tự ưu tiên: Adjust → Install Referrer → Apple Search Ads → Firebase.

| Param | Kiểu | Mô tả |
|---|---|---|
| `forceRefresh` | `Boolean` | `true` = fetch lại toàn bộ provider trước khi resolve. |
| `onResult` | `(AttributionVerdict) -> Unit` | Nhận verdict cuối cùng. |

#### `suspend fun resolveAttributionAwait(forceRefresh: Boolean = false): AttributionVerdict`

Bản `suspend` của `resolveAttribution` (bọc bằng `suspendCancellableCoroutine`, hỗ trợ cancel).

**Trả về:** `AttributionVerdict` như trên.

#### `fun isOrganic(forceRefresh: Boolean = false, onResult: (Boolean) -> Unit)`
#### `suspend fun isOrganicAwait(forceRefresh: Boolean = false): Boolean`

Tiện ích: gọi `resolveAttribution` rồi trả `verdict.attribution == Attribution.ORGANIC`. Lưu ý: verdict `UNKNOWN` cũng trả `false` (không phải organic ≠ chắc chắn paid).

**Trả về:** `true` chỉ khi verdict là `ORGANIC`.

#### `fun clearCache()`

Xoá toàn bộ cache `InstallSource` của mọi provider, xoá verdict đã log gần nhất (khiến logger có thể log lại `source_user` ở lần `init()` sau), và reset tập provider ready về rỗng.

### Tự động log GA4 (hành vi nền của `init`)

Logger poll mỗi **1,5 giây**:

- **Lần log đầu (`source_user`):** log khi verdict đạt `HIGH`, hoặc `MEDIUM` khi cả 3 trust provider đã ready, hoặc khi hết `attributionLogTimeoutMs` (log verdict hiện có kể cả `UNKNOWN`). Verdict đã log được lưu bền vững → **các lần mở app sau không log lại** `source_user`.
- **Nâng cấp (`source_user_updated`):** trong `attributionUpgradeWindowMs`, nếu xuất hiện verdict có confidence **cao hơn** VÀ khác attribution hoặc khác network → log event update. Đạt `HIGH` thì logger dừng. ⚠️ Window này tính theo **từng session**: timestamp lần log đầu không được persist (chỉ verdict được lưu), nên mỗi lần mở app sau khi đã commit, logger mở một cửa sổ upgrade mới dài `attributionUpgradeWindowMs` — event update có thể phát ở session sau, không chỉ trong window đầu tiên.
- Mỗi event kèm params: `attribution`, `is_organic`, `confidence`, `primary_provider`, `network`, `providers_ready`, `reason`; đồng thời set user property `way_attribution`, `way_is_organic`, `way_primary_provider`.

### `fun createSettings(name: String): Settings` (top-level, `com.wayinstall.utils`)

Hàm `expect/actual` public tạo store key-value `com.russhwolf.settings.Settings` theo tên: Android dùng `SharedPreferencesSettings` (cần `LifecycleProvider` đã có Application), iOS dùng `NSUserDefaultsSettings`. Chủ yếu phục vụ nội bộ SDK; app thường không cần gọi trực tiếp.

## Public models

### `UserInstallConfig` (data class)

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `adjustAttributionTimeoutMs` | `Long` (mặc định `10_000`) | Timeout chờ Adjust attribution. |
| `installReferrerTimeoutMs` | `Long` (mặc định `5_000`) | Timeout kết nối Play Install Referrer (Android). |
| `appleAdServicesTimeoutMs` | `Long` (mặc định `5_000`) | Timeout gọi API Apple AdServices (iOS). |
| `attributionLogTimeoutMs` | `Long` (mặc định `60_000`) | Hạn chót để logger commit event `source_user` đầu tiên. |
| `attributionUpgradeWindowMs` | `Long` (mặc định `60_000`) | Cửa sổ sau lần log đầu cho phép log `source_user_updated`. |
| `isDebug` | `Boolean` (mặc định `false`) | Bật debug log (Logcat/NSLog). |

### `InstallSource` (data class, `@Serializable`)

Tín hiệu attribution thô của một provider.

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `provider` | `Provider` | Provider sinh ra tín hiệu. |
| `network` | `String?` | Tên network quảng cáo (vd `"Google Ads"`, `"Apple Search Ads"`); `null` nếu organic/không rõ. |
| `campaign` | `String?` | Tên/ID campaign. |
| `adgroup` | `String?` | Tên/ID adgroup (Install Referrer dùng `utm_content`). |
| `creative` | `String?` | Tên/ID creative. |
| `isOrganic` | `Boolean` (mặc định `true`) | `false` khi có tín hiệu paid. |
| `confidence` | `Confidence` (mặc định `LOW`) | Độ tin của tín hiệu này. |
| `rawData` | `Map<String, String>` | Dữ liệu gốc từ provider (referrer string, utm params, tracker token, response Apple...). |

`InstallSource.Unknown`: hằng companion — `provider = UNKNOWN`, `isOrganic = true`, `confidence = LOW`. Các hàm `fromXxx` trả `Unknown.copy(provider = ...)` khi fetch thất bại.

### `AttributionVerdict` (data class, `@Serializable`)

Kết quả hợp nhất cuối cùng.

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `attribution` | `Attribution` | `PAID` / `ORGANIC` / `UNKNOWN`. |
| `confidence` | `Confidence` | Độ tin của verdict. |
| `primaryProvider` | `Provider` | Provider quyết định verdict (theo thứ tự ưu tiên). |
| `network` | `String?` | Network của tín hiệu paid chính; luôn `null` khi organic/unknown. |
| `readyProviders` | `Set<Provider>` | Các provider đã fetch xong (kể cả fetch fail) tại thời điểm resolve. |
| `reason` | `String` | Lý do máy đọc được: `paid_corroborated`, `paid_adjust`, `organic_corroborated`, `organic_install_referrer`, `no_trust_provider_ready`, `waiting_for_trust:...`, `all_trust_low_signal`, ... |

### `Attribution` (enum)

`PAID`, `ORGANIC`, `UNKNOWN`.

### `Confidence` (enum)

`HIGH`, `MEDIUM`, `LOW` (LOW bị resolver bỏ qua khi hợp nhất).

### `Provider` (enum)

`ADJUST`, `INSTALL_REFERRER`, `FIREBASE`, `APPLE_SEARCH_ADS`, `UNKNOWN`.

## Lưu ý platform

**Android**

- `fromAdjust`: gọi `Adjust.getAttributionWithTimeout(context, timeout)`. `network` rỗng hoặc `"Unattributed"`/`"Organic"` → organic (`"Organic"` tường minh → `MEDIUM`, còn lại `LOW`); có network thực → paid, `confidence = HIGH`. Cần `LifecycleProvider.context` (từ `lifecycle_kmp`), thiếu context → trả `Unknown`.
- `fromInstallReferrer`: dùng Play Install Referrer client, parse chuỗi referrer thành `utm_source/utm_medium/utm_campaign/utm_content/gclid` (giá trị `"(not set)"` bị loại). Có `gclid` → `network = "Google Ads"`; `utm_medium=organic` → organic `MEDIUM`; không có tín hiệu nào → organic `LOW`; có tín hiệu paid → `confidence = MEDIUM`. `FEATURE_NOT_SUPPORTED`/`SERVICE_UNAVAILABLE` → trả (và cache) bản `Unknown` của provider; response code khác hoặc timeout → fail (`Unknown`, không cache).
- `fromFirebase`: chỉ lấy `appInstanceId` của Firebase Analytics vào `rawData` — luôn `isOrganic = true`, `confidence = LOW`; mang tính bổ trợ, không ảnh hưởng verdict.
- `fromAppleSearchAds`: **stub**, luôn trả `Unknown(provider = APPLE_SEARCH_ADS)`.

**iOS**

- `fromAdjust`: gọi `Adjust.attributionWithCompletionHandler` bọc `withTimeout`; logic phân loại giống Android; `rawData` gồm cả tracker token/name, click label, cost type/amount/currency.
- `fromAppleSearchAds`: lấy token qua `AAAttribution.attributionTokenWithError` (yêu cầu iOS ≥ 14.3; không có token → fail) rồi POST token tới `https://api-adservices.apple.com/api/v1/`. Response `attribution=true` → paid, `network = "Apple Search Ads"`, kèm `campaignId/adGroupId/adId/keywordId`; `attribution=false` → organic. **Lưu ý theo code hiện tại:** cả hai nhánh đều không set `confidence` (giữ mặc định `LOW`), nên tín hiệu Apple Search Ads — kể cả paid — đang bị resolver lọc bỏ và không ảnh hưởng verdict; nó chỉ xuất hiện qua `fromAppleSearchAds` và `readyProviders`.
- `fromInstallReferrer`: **stub**, luôn trả `Unknown(provider = INSTALL_REFERRER)` — tức trust provider này luôn "ready nhưng LOW" trên iOS.
- `fromFirebase`: **stub**, trả `InstallSource(FIREBASE, isOrganic = true)` — không lấy appInstanceId như Android.

**Chung:** cache là vĩnh viễn theo cài đặt app (SharedPreferences/NSUserDefaults) — attribution chỉ fetch thật một lần trong đời cài đặt trừ khi dùng `forceRefresh` hoặc `clearCache()`. Trên thực tế verdict có ý nghĩa được quyết định bởi Adjust + Install Referrer (Android) và Adjust (iOS).

## Log TAG

Module log với TAG `WayInstall/Kit` và `WayInstall/Cache`. Toàn bộ quy ước TAG của WaySDK: xem `wayCore/API.md`.
