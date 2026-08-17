# wayAd — Public API

`wayAd` là module **core** quảng cáo của WaySDK (Kotlin Multiplatform, target Android + iOS): lớp trừu tượng thống nhất trên nhiều ad network cho 6 định dạng quảng cáo (banner, native, interstitial, rewarded, rewarded-interstitial, app open). Module chỉ chứa phần network-neutral: entry point `WayAdKit`, hệ thống scope theo network (`NetworkAdapterScope`), manager cho quảng cáo fullscreen, hệ thống preload theo placement, cơ chế waterfall nhiều ad unit (strategy request), state/layout Compose Multiplatform (`BannerAdLayout`, `BannerAdState`, `NativeAdState`, shimmer) và base View XML cho Android. **Core không chứa bất kỳ ad SDK nào** — mỗi network là một module implement riêng, theo đúng mô hình `wayPay` / `wayPay-store` / `wayPay-revenuecat` (nhưng đa slot: các network chạy song song qua `scopes`).

Phụ thuộc (theo `wayAd/build.gradle.kts`): `commonMain` expose qua `api` các module `:wayPay` (kiểm tra subscription), `:wayInstall` (`UserInstallKit`), `:adjust_kmp` (tracking Adjust). Android thêm Google UMP; iOS cinterop duy nhất là `UserMessagingPlatform.xcframework` — UMP là CMP (consent platform) dùng chung toàn app nên thuộc core, không thuộc riêng Admob. GoogleMobileAds cinterop nằm ở `:wayAd-admob`, AppLovinSDK cinterop nằm ở `:wayAd-applovin`.

## Chọn ad network — mỗi network một module

| Module | Artifact | Chứa gì |
|---|---|---|
| `:wayAd` | `way-sdk:way-ad` | Core network-neutral (bắt buộc) |
| `:wayAd-admob` | `way-sdk:way-ad-admob` | `AdmobNetworkAdapterScope`, adapter, `AdmobNativeLayout`, `FloatingNative`, `Admob*AdView`, factory Banner/Native, **toàn bộ** mediation stack của AdMob `com.google.ads.mediation:*` (gồm cả `applovin` — mediation đi theo network sở hữu nó) |
| `:wayAd-applovin` | `way-sdk:way-ad-applovin` | `AppLovinMaxNetworkAdapterScope`, adapter, `MaxNativeLayout`, `AppLovin*AdView`, mediation stack của MAX `com.applovin.mediation:*-adapter` |

| App dùng | Dependency Gradle | SPM package cần thêm trong Xcode |
|---|---|---|
| Chỉ AdMob | `way-ad` + `way-ad-admob` | GoogleMobileAds, GoogleUserMessagingPlatform, Adjust |
| AdMob + AppLovin MAX | thêm `way-ad-applovin` | thêm AppLovin-MAX-Swift-Package |

Nguyên tắc phân chia: **tách theo network adapter; mediation adapter đi theo network sở hữu nó.** `com.google.ads.mediation:applovin` là adapter để AdMob mediate AppLovin → thuộc `:wayAd-admob` (dù nó kéo theo `com.applovin:applovin-sdk` trên Android — giống như mọi mediation adapter khác đều kéo SDK của partner tương ứng). App chỉ dùng AdMob vẫn không có **MAX adapter code** và trên iOS không cần AppLovin-MAX-Swift-Package.

**Lưu ý version:** các module `way-ad-*` dùng API nội bộ (`@InternalWayAdApi`) của core, không có cam kết ổn định giữa các phiên bản — luôn khai báo **cùng một version** cho mọi artifact `way-ad*`.

Phiên bản SDK iOS được `verifyInteropVersions` (root `build.gradle.kts`) đối chiếu với `iosApp/.../Package.resolved`; app không khai báo AppLovin trong SPM thì pin AppLovin đơn giản bị bỏ qua, không báo lỗi.

> **QUAN TRỌNG — Tự chặn quảng cáo khi user có subscription:**
> wayAd tự động chặn load/show quảng cáo khi `com.waypay.AppPurchaseManager.getInstance().isSubscription == true`. Cơ chế này nằm ở **hai tầng**:
> 1. **Adapter** (`AdmobNetworkAdapter`, `AppLovinMaxNetworkAdapter`, cả Android lẫn iOS): mọi hàm `load*Ad` gọi `callback.onAdFailedToLoad(AdError("Subscription is active"))` và mọi hàm `show*Ad` gọi `callback.onAdFailedToShow(AdError("Subscription is active"))` rồi return sớm.
> 2. **Preload** (`BaseAdPreload`): `preloadAd` bỏ qua không load; `popBufferedAd` và `awaitAd` trả về `null` ngay cả khi buffer đang có quảng cáo.
>
> Hệ quả: app chỉ cần kết nối `AppPurchaseManager` (module wayPay) là toàn bộ quảng cáo tự tắt cho user premium — không cần check thủ công ở từng chỗ gọi. Với các manager fullscreen, luồng `show(keyPreload, ...)` khi có subscription sẽ đi vào nhánh `onFailToShow` / `onNextAction(false)`.

Ngoài ra, adapter cũng từ chối load khi `adUnitId` rỗng (`onAdFailedToLoad(AdError("... Ad unit id is blank"))`) — các factory tận dụng điều này bằng cách trả về request với `adUnitId = ""` khi `canShowAds = false`.

## Khởi tạo

Tham khảo cách dùng thực tế trong `composeApp/src/commonMain/kotlin/com/way_sdk/App.kt`:

```kotlin
import com.wayad.admob.AdmobNetworkAdapterScope           // cần :wayAd-admob
import com.wayad.admob.model.AdTestIds
import com.wayad.applovin.AppLovinMaxNetworkAdapterScope  // cần :wayAd-applovin
import com.wayad.common.WayAdKit
import com.wayad.common.model.WayAdConfig
import com.wayad.common.registerAdResume                  // overload 1 tham số nằm trong :wayAd-admob
import com.wayad.core.api.appopen.AppOpenAdRequest

// 1. Khởi tạo kit (chạy 1 lần khi app start)
WayAdKit.init(
    wayAdConfig = WayAdConfig(
        tokenAdjust = "dcr07sgkv37k",          // app token Adjust (bắt buộc)
        tokenAdImpression = "53vdaz",           // event token Adjust cho ad impression
        appLovinSdkKey = "<APPLOVIN_SDK_KEY>",  // bắt buộc nếu dùng AppLovinMaxNetworkAdapterScope
    ),
    isDebug = true,
    scopes = listOf(                            // BẮT BUỘC — core không còn default Admob
        AdmobNetworkAdapterScope,
        AppLovinMaxNetworkAdapterScope,
    ),
    onComplete = {},
)

// 2. Đăng ký app-open ad khi resume app (chỉ hoạt động trên Android)
WayAdKit.registerAdResume(AppOpenAdRequest(AdTestIds.APP_OPEN))

// 3. (khuyến nghị) Kết nối AppPurchaseManager của wayPay để wayAd tự chặn ads cho user premium
AppPurchaseManager.getInstance().connect(PurchaseConfig(subscriptionIds = setOf(...)))
```

Sau khi init, mọi thao tác load/show đi qua scope của từng network, ví dụ:

```kotlin
// Interstitial: preload rồi show theo key
AdmobNetworkAdapterScope.preloadAd("inter_home", InterstitialAdFactory.create(adUnitId = "..."))
AdmobNetworkAdapterScope.interstitialAdManager.show(
    keyPreload = "inter_home",
    activity = activity, // Android: Activity; iOS: có thể null
    onNextAction = { success -> /* điều hướng tiếp */ },
)

// Banner trong Compose
val state = rememberBannerAdState(
    config = BannerAdFactory.create(canShowAds = true, adUnitId = AdTestIds.BANNER, adPlacementId = null)
)
LaunchedEffect(Unit) { state.loadAd() }
BannerAdLayout(state)
```

---

## Public API

### `WayAdKit` (object, `com.wayad.common`)

Entry point của module. Giữ `WayAdConfig`, danh sách scope đã đăng ký, cờ debug, và trạng thái khởi tạo.

**Property**

| Property | Kiểu | Ý nghĩa |
|---|---|---|
| `scopes` | `List<NetworkAdapterScope>` | Danh sách scope đã đăng ký qua `init` (mặc định rỗng trước khi init). |
| `isKitInitialized` | `StateFlow<Boolean>` | Trạng thái đã init xong. **Giá trị khởi tạo là `true` trên Android, `false` trên iOS** (adapter chờ flow này bật `true` trước khi thật sự load ad). |

#### `fun init(wayAdConfig: WayAdConfig, isDebug: Boolean, scopes: List<NetworkAdapterScope>)`

Khởi tạo đồng bộ, **không** chạy luồng consent UMP. `scopes` **không có default và không được rỗng** (`require(scopes.isNotEmpty())` — core không tự fallback sang AdMob; `AdmobNetworkAdapterScope` nằm ở `:wayAd-admob`). Lưu config + cờ debug, đăng ký `scopes`; trên Android gọi `com.adjust_kmp.setupAdjust` ngay; gọi `adapter.initialize()` cho từng scope; set `isKitInitialized = true`; khởi tạo `UserInstallKit` (module wayInstall).

| Param | Kiểu | Mô tả |
|---|---|---|
| `wayAdConfig` | `WayAdConfig` | Config chung (token Adjust, SDK key AppLovin, timeout preload…). |
| `isDebug` | `Boolean` | Bật chế độ debug (Adjust sandbox, verbose log AppLovin…). |
| `scopes` | `List<NetworkAdapterScope>` | Các network sử dụng. **Bắt buộc, không rỗng** — core không có default (Admob nằm ở `:wayAd-admob`). |

**Trả về:** `Unit`.

#### `fun init(wayAdConfig: WayAdConfig, isDebug: Boolean = false, scopes: List<NetworkAdapterScope>, onComplete: () -> Unit = {})`

Overload bất đồng bộ, **có** luồng consent: gọi `requestConsentInfoUpdate` (Google UMP — hiện form consent nếu cần) trước, sau đó (riêng iOS thêm bước `com.adjust_kmp.setupAdjust` — bao gồm xin quyền ATT) rồi mới gọi overload `init` 3 tham số ở trên và cuối cùng `onComplete()`.

| Param | Kiểu | Mô tả |
|---|---|---|
| `wayAdConfig` | `WayAdConfig` | Như trên. |
| `isDebug` | `Boolean` | Như trên. Mặc định `false`. |
| `scopes` | `List<NetworkAdapterScope>` | Như trên. |
| `onComplete` | `() -> Unit` | Callback khi toàn bộ chuỗi consent → adjust → init hoàn tất. |

**Trả về:** `Unit`.

#### `fun isShowingAdsFullscreen(): Boolean`

**Trả về:** `true` nếu **bất kỳ** scope nào đang hiển thị quảng cáo fullscreen (interstitial, rewarded, app open, rewarded-interstitial).

#### `fun isShowingAdsFullscreenAsFlow(): Flow<Boolean>`

**Trả về:** `Flow<Boolean>` combine `hasAdsShowing` của tất cả manager fullscreen trong mọi scope; phát `true` khi có ít nhất một quảng cáo fullscreen đang hiển thị. Nếu chưa có scope nào, trả flow hằng `false`. (Lưu ý: là `Flow`, không phải `StateFlow`.)

#### `fun enableMessageForTester()`

Bật hiển thị Toast thông báo cho tester (chỉ có tác dụng khi `isDebug = true`; chỉ hiển thị trên Android, iOS chỉ log). **Trả về:** `Unit`.

#### `suspend fun awaitConsentInfoUpdate(activity: Any?): Boolean`

Bọc `requestConsentInfoUpdate` thành hàm suspend.

| Param | Kiểu | Mô tả |
|---|---|---|
| `activity` | `Any?` | Android: `Activity` (null thì lấy từ `LifecycleProvider.activity`). iOS: không dùng. |

**Trả về:** `Boolean` — kết quả luồng consent. Lưu ý hành vi **khác nhau giữa 2 platform khi consent đã có sẵn** (status `OBTAINED`/`NOT_REQUIRED`): Android trả `false`, iOS trả `true`. Android còn trả `false` khi activity null hoặc UMP báo lỗi.

#### `fun setCanLoadGlobalAds(enable: Boolean)` — **`@Deprecated` (WARNING)**

Alias cũ của [`WayAdToggle.setGlobalEnabled(enable)`](#wayadtoggle-object-comwayadcommon). Vẫn hoạt động, nhưng nên chuyển sang `WayAdToggle` để dùng được cả cờ riêng từng loại. **Trả về:** `Unit`.

> `setNetworkAdapter(adapter)` và các property preload/strategy cũ trên `WayAdKit`, cùng `PreloadManager.preloadAd`/`preloadAdIfEmpty`, `InterstitialAdManager.Companion.*` (và tương tự ở các manager khác) đã bị **xóa hẳn** khỏi core (không còn symbol, kể cả dạng deprecated) — dùng API tương ứng trên scope (`AdmobNetworkAdapterScope...`).

### `WayAdToggle` (object, `com.wayad.common`)

Kill-switch cho quảng cáo: một cờ **global** cho toàn SDK, cộng một cờ riêng cho **từng [`AdType`](#adtype-enum-comwayadcommonmodel)**. Tất cả mặc định `true`. Một loại chỉ được load khi **cả hai** cờ cùng bật (`canLoad`). Cờ được đọc tại **thời điểm load**, nên tắt giữa chừng là chặn luôn cả các lần load/refresh sau đó.

Tác động ở 2 tầng, giống cờ global cũ:

1. **Chặn load ở tầng adapter**: mọi `load*` trong cả 4 adapter (AdMob/AppLovin × Android/iOS) fail ngay với `AdError` — message là `"Can't load global ads"` khi tắt global, hoặc `"Can't load <loại> ads"` (ví dụ `"Can't load native ads"`) khi chỉ tắt riêng loại đó.
2. **Default `isVisible` ở tầng factory**: `BannerAdFactory.create(...)` dùng `canShowAds && WayAdToggle.canLoad(AdType.BANNER)`, `NativeAdFactory.create(...)` và builder `AdmobNativeXmlConfig` (Android) dùng `AdType.NATIVE` — chỉ áp tại thời điểm TẠO config; config đã tạo trước khi đổi cờ không tự cập nhật `isVisible`.

| Thành viên | Chữ ký | Mô tả |
|---|---|---|
| `isGlobalEnabled` | `val: Boolean` | Trạng thái cờ global. |
| `setGlobalEnabled` | `(enable: Boolean)` | Bật/tắt toàn bộ ads, bất kể cờ riêng từng loại. |
| `isTypeEnabled` | `(type: AdType): Boolean` | Cờ riêng của loại, **không** tính cờ global. |
| `setEnabled` | `(type: AdType, enable: Boolean)` | Bật/tắt riêng một loại. |
| `setEnabled` | `(types: Collection<AdType>, enable: Boolean)` | Bật/tắt nhiều loại một lượt. |
| `canLoad` | `(type: AdType): Boolean` | Kết quả cuối cùng: `isGlobalEnabled && isTypeEnabled(type)`. |
| `enableAll` | `()` | Đưa mọi cờ (global + từng loại) về `true`. |

```kotlin
import com.wayad.common.WayAdToggle
import com.wayad.common.model.AdType

WayAdToggle.setEnabled(AdType.APP_OPEN, false)                              // chỉ tắt app open
WayAdToggle.setEnabled(listOf(AdType.BANNER, AdType.NATIVE), false)         // tắt banner + native
WayAdToggle.setGlobalEnabled(false)                                         // kill-switch toàn bộ
WayAdToggle.canLoad(AdType.BANNER)                                          // false (global đang tắt)
WayAdToggle.enableAll()                                                     // reset về mặc định
```

Mọi cờ là atomic, đọc/ghi được từ bất kỳ thread nào; trạng thái chỉ tồn tại trong process (không persist qua lần mở app sau).

### `AdType` (enum, `com.wayad.common.model`)

`BANNER`, `NATIVE`, `INTERSTITIAL`, `REWARD`, `INTERSTITIAL_REWARD`, `APP_OPEN` — khoá cho cờ bật/tắt của `WayAdToggle`.

> AppLovin MAX chưa hỗ trợ rewarded-interstitial, nên `AdType.INTERSTITIAL_REWARD` chỉ có tác dụng với AdMob.

### Hàm expect top-level trên `WayAdKit` (`com.wayad.common`)

> `WayAdKit.setupAdjust(...)` đã chuyển sang module `adjust_kmp` thành hàm top-level `com.adjust_kmp.setupAdjust(tokenAdjust, isDebug, onComplete)` (xem `adjust_kmp/API.md`). `WayAdKit` gọi hàm này bên trong `init`, consumer không cần gọi trực tiếp.

#### `fun WayAdKit.requestConsentInfoUpdate(activity: Any?, onComplete: (Boolean) -> Unit)`

Chạy luồng consent Google UMP (debug geography EEA + test device id từ `WayAdConfig.testDeviceIds`, có danh sách mặc định). Hiện form consent nếu cần. UMP là CMP dùng chung toàn app (AppLovin MAX đọc chuỗi TCF do UMP ghi) nên nằm ở core và chạy bất kể app đăng ký network nào. Xem `awaitConsentInfoUpdate` về giá trị Boolean. **Trả về:** `Unit`.

#### `fun WayAdKit.registerAdResume(appOpenAdRequest: AppOpenAdRequest, scope: NetworkAdapterScope)`

Flow "ad resume" network-neutral, nằm ở **core**: preload app-open ad của `scope` truyền vào với key nội bộ `KEY_PRELOAD_AD_RESUME`, rồi observe `ProcessLifecycleOwner`: mỗi lần app resume sẽ show app-open ad nếu thỏa điều kiện (không phải lần mở đầu của session, không bị `disableByClick`, activity/route hiện tại không nằm trong danh sách disable). Sau mỗi impression tự preload lại. Gọi lần thứ hai trở đi chỉ log warning và bị bỏ qua (guard chống đăng ký đúp observer).

Mỗi network module tự đăng ký fullscreen activity của nó vào danh sách chặn trong `adapter.initialize()`: Admob đăng ký `AdActivity`, AppLovin đăng ký `AppLovinFullscreenActivity` — core không biết activity của network nào.

`:wayAd-admob` cung cấp overload tiện lợi giữ tương thích nguồn với chữ ký cũ (cùng package `com.wayad.common`): `fun WayAdKit.registerAdResume(appOpenAdRequest)` = gọi bản core với `AdmobNetworkAdapterScope`.

**iOS:** no-op (chỉ log "Ios not registerAdResume").

| Param | Kiểu | Mô tả |
|---|---|---|
| `appOpenAdRequest` | `AppOpenAdRequest` | Request app-open dùng cho ad resume (có thể là `AppOpenAdStrategyRequest`). |
| `scope` | `NetworkAdapterScope` | Network cung cấp app-open ad cho flow resume. |

**Trả về:** `Unit`.

### `NetworkAdapterScope` (open class, `com.wayad.common.model`)

Gom toàn bộ hạ tầng (preload, strategy loader, manager) cho **một** ad network. App không tự tạo mà dùng 2 object có sẵn: `AdmobNetworkAdapterScope` và `AppLovinMaxNetworkAdapterScope` (hoặc kế thừa để thêm network mới với `AdNetworkAdapter` tự viết).

**Property**

| Property | Kiểu | Ý nghĩa |
|---|---|---|
| `adapter` | `AdNetworkAdapter` | Adapter network của scope này. |
| `interstitialAdManager` | `InterstitialAdManager` | Manager interstitial (lazy). |
| `rewardAdManager` | `RewardAdManager` | Manager rewarded (lazy). |
| `appOpenAdManager` | `AppOpenAdManager` | Manager app open (lazy). |
| `interstitialRewardAdManager` | `InterstitialRewardAdManager` | Manager rewarded-interstitial (lazy). |

Ngoài ra scope còn expose 12 property `*AdPreload` / `*AdStrategyLoader` (public, gắn **`@InternalWayAdApi`** opt-in mức ERROR) — dành riêng cho các module network dùng chéo module, có thể đổi/xóa không báo trước; app **không** nên opt-in để dùng trực tiếp.

#### `fun preloadAd(keyPreload: String, adRequest: CoreAdRequest)`

Route request tới preload manager tương ứng theo kiểu runtime của `adRequest` (banner/native/interstitial/reward/interstitial-reward/app open) và bắt đầu load, đưa kết quả vào buffer theo `keyPreload`. Nếu đang có job preload active cho key này, job cũ bị cancel và load lại. Bỏ qua khi user có subscription.

| Param | Kiểu | Mô tả |
|---|---|---|
| `keyPreload` | `String` | Placement key định danh buffer. |
| `adRequest` | `CoreAdRequest` | Một trong các `*AdRequest` / `*AdStrategyRequest`. |

**Trả về:** `Unit`.

#### `fun preloadIfEmpty(keyPreload: String, adRequest: CoreAdRequest, canLoadAds: Boolean = true)`

Như `preloadAd` nhưng chỉ load khi buffer của `keyPreload` đang **rỗng** và `canLoadAds = true`.

| Param | Kiểu | Mô tả |
|---|---|---|
| `keyPreload` | `String` | Placement key. |
| `adRequest` | `CoreAdRequest` | Request. |
| `canLoadAds` | `Boolean` | Cho phép load hay không. |

**Trả về:** `Unit`.

### `AdmobNetworkAdapterScope` (object, `com.wayad.admob`)

Scope AdMob mặc định — `NetworkAdapterScope(AdmobNetworkAdapter())`. Thêm shortcut tới các factory:

| Property | Kiểu | Ý nghĩa |
|---|---|---|
| `bannerFactory` | `BannerAdFactory` | Factory banner. |
| `nativeFactory` | `NativeAdFactory` | Factory native. |
| `interstitialFactory` | `InterstitialAdFactory` | Factory interstitial. |
| `rewardFactory` | `RewardAdFactory` | Factory rewarded. |
| `appOpenFactory` | `AppOpenAdFactory` | Factory app open. |
| `interstitialRewardFactory` | `InterstitialRewardAdFactory` | Factory rewarded-interstitial. |

### `AppLovinMaxNetworkAdapterScope` (object, `com.wayad.applovin`)

Scope AppLovin MAX — `NetworkAdapterScope(AppLovinMaxNetworkAdapter())`. Yêu cầu `WayAdConfig.appLovinSdkKey` khi init (thiếu sẽ `onInitializeFailed`). **AppLovin MAX không hỗ trợ rewarded-interstitial** — load/show loại này luôn fail với thông báo tương ứng.

| Property | Kiểu | Ý nghĩa |
|---|---|---|
| `bannerFactory` | `AppLovinBannerFactory` | Factory banner AppLovin. |
| `nativeFactory` | `AppLovinNativeFactory` | Factory native AppLovin. |

#### `fun AppLovinMaxNetworkAdapterScope.showMediationDebugger()` (expect, `com.wayad.applovin`)

Mở AppLovin Mediation Debugger. Android cần đã init (`LifecycleProvider.context` khác null, ngược lại chỉ log warning); iOS gọi thẳng `ALSdk.shared()`. **Trả về:** `Unit`.

---

### Manager quảng cáo fullscreen (`com.wayad.common.manager`)

Bốn manager có cấu trúc giống nhau, lấy qua scope (`scope.interstitialAdManager`…). Constructor là `internal` — không tự khởi tạo được. Điểm chung:

- `hasAdsShowing: StateFlow<Boolean>` — trạng thái đang hiển thị của loại ad đó trong scope.
- `suspend fun load(request, onAdFailedToLoad, onAdLoaded): Result?` — load qua strategy loader (hỗ trợ waterfall khi request là `*StrategyRequest`). **Trả về kết quả hoặc `null` khi fail** (không throw); đồng thời gọi callback tương ứng (`onAdFailedToLoad` nhận message của exception).
- `fun show(result?, activity, ...)` — show kết quả đã load. Nếu `result == null`: gọi `onFailToShow("Ad is null")` + `onNextAction(false)` và return. Trong khi show, `hasAdsShowing = true`; reset về `false` khi đóng/fail.
- `suspend fun show(keyPreload, activity, showLoading, dismissLoading, ...)` — show từ buffer preload: nếu `canAwaitAd(keyPreload)` (đang load hoặc có sẵn) thì `showLoading()`, `awaitAd` với timeout `WayAdConfig.preloadAwaitTimeoutMs` (mặc định 30s), show rồi `dismissLoading()`; ngược lại `onFailToShow("Ad is not ready")` + `onNextAction(false)`.
- Ngữ nghĩa callback: `onNextAction(true)` được gọi khi ad đóng thành công (cùng với `onAdClose`), `onNextAction(false)` khi show fail — dùng nó làm điểm điều hướng tiếp.

#### `InterstitialAdManager`

| Hàm | Chữ ký | Ghi chú |
|---|---|---|
| `isInterstitialAdShowing` | `(): Boolean` | Giá trị hiện tại của `hasAdsShowing`. |
| `load` | `suspend (interstitialAdRequest: InterstitialAdRequest, onAdFailedToLoad: (String) -> Unit = {}, onAdLoaded: () -> Unit = {}): InterstitialAdResult?` | Null khi fail. |
| `show` | `(interstitialAdResult: InterstitialAdResult?, activity: Any?, onImpression: () -> Unit = {}, onNextAction: (Boolean) -> Unit = {}, onAdClose: () -> Unit = {}, onAdClick: () -> Unit = {}, onFailToShow: (String) -> Unit = {})` | `activity`: Android cần `Activity` (null thì lấy activity hiện tại từ `LifecycleProvider`); iOS có thể null. |
| `show` (preload) | `suspend (keyPreload: String, activity: Any?, showLoading: () -> Unit = {}, dismissLoading: () -> Unit = {}, onImpression, onNextAction, onAdClose, onAdClick, onFailToShow)` | Xem điểm chung ở trên. |

#### `RewardAdManager`

Như `InterstitialAdManager` với kiểu `RewardAdRequest`/`RewardAdResult`, thêm:

| Hàm | Chữ ký | Ghi chú |
|---|---|---|
| `isRewardShowing` | `(): Boolean` | |
| `show` / `show(keyPreload)` | thêm tham số cuối `onRewardEarned: (amount: Int) -> Unit = {}` | Gọi khi user nhận thưởng (`onUserEarnedReward`); chỉ truyền `amount`, không truyền `type`. |

#### `InterstitialRewardAdManager`

Như `InterstitialAdManager` với kiểu `InterstitialRewardAdRequest`/`InterstitialRewardAdResult`; `isInterstitialRewardAdShowing(): Boolean`. Lưu ý callback show **không** có `onRewardEarned` dù bên dưới là `InterstitialRewardAdShowCallback` (kế thừa `RewardAdShowCallback`). Chỉ hoạt động với AdMob (AppLovin không hỗ trợ).

#### `AppOpenAdManager`

Như `InterstitialAdManager` với kiểu `AppOpenAdRequest`/`AppOpenAdResult`; `isAppOpenShowing(): Boolean`. Thêm:

#### `fun preloadAdResume(appOpenAdRequest: AppOpenAdRequest)`

Preload app-open vào buffer key nội bộ `KEY_PRELOAD_AD_RESUME` (buffer mà `registerAdResume` dùng để show khi app resume). **Trả về:** `Unit`.

**`AppOpenAdManager.Companion` — điều khiển ad-resume toàn cục:**

| Hàm | Mô tả |
|---|---|
| `fun setupCurrentRoute(route: String?)` | Cập nhật route hiện tại (ví dụ route của NavHost) để đối chiếu với danh sách route bị chặn. |
| `fun disableByClick()` | Chặn **một lần** app-open ad sắp tới (dùng khi user vừa click ra ngoài app: mở CH Play, share…). Cờ tự bật lại sau lần resume kế tiếp. |
| `fun enableCanShowAdOpenNow()` | Bật lại cờ trên ngay lập tức. |
| `fun disableAdResumeWithActivity(nameClazzActivity: String)` | Không show ad-resume khi activity hiện tại có class name này. |
| `fun disableAdResumeWithRoute(routeQualifiedName: String?)` | Không show ad-resume khi route hiện tại trùng giá trị này (null thì bỏ qua). |

---

### Factory tạo request/config (`com.wayad.common.factory`, `com.wayad.applovin.factory`)

Các object factory tạo `*AdRequest` (1–3 ad unit, waterfall cao→thấp) hoặc `AdViewConfig` hoàn chỉnh. Quy tắc chung của các overload nhiều ad unit:

- `canShowAds = false` → request với `adUnitId = ""` (adapter sẽ fail "Ad unit id is blank" → coi như tắt ads).
- `canShowAdsHigh = false` → bỏ ad unit giá cao nhất khỏi waterfall; `canShowAdsMedium = false` → bỏ ad unit tầng giữa.
- Khi còn ≥ 2 ad unit → trả về `*AdStrategyRequest` (waterfall); còn 1 → request thường.

#### `BannerAdFactory` (object)

| Hàm | Chữ ký | Trả về |
|---|---|---|
| `create` | `(canShowAds: Boolean = true, adUnitId: String, adSize: BannerAdSize = ADAPTIVE, collapsePosition: CollapsePosition? = null)` | `BannerAdRequest` |
| `create` | `(canShowAds = true, adUnitId1: String, adUnitId2: String, canShowAdsHigh = true, adSize, collapsePosition)` | `BannerAdRequest` (waterfall 2 tầng) |
| `create` | `(canShowAds = true, adUnitId1, adUnitId2, adUnitId3, canShowAdsHigh = true, canShowAdsMedium = true, adSize, collapsePosition)` | `BannerAdRequest` (waterfall 3 tầng) |
| `create` | `(canShowAds: Boolean, adUnitId: String, isVisible: Boolean = canShowAds && WayAdToggle.canLoad(AdType.BANNER), autoRefresh: Boolean = true, timeToReload: Long = 20_000, adPlacementId: String? = null, reloadOnResume: Boolean = true)` | `AdmobBannerConfig` (config cho Compose/XML view; `adPlacementId != null` thì bật preload) |
| `create` | `(canShowAds, canShowAdsHigh, adUnitId1, adUnitId2, adPlacementId: String? = null, isVisible..., autoRefresh = true, timeToReload: Long = 20_000, reloadOnResume = true)` | `AdmobBannerConfig` waterfall 2 tầng |

#### `NativeAdFactory` (object)

Tương tự `BannerAdFactory` (không có adSize/collapse): 3 overload trả `NativeAdRequest`, 2 overload trả `AdmobNativeConfig` với chữ ký `(canShowAds, [canShowAdsHigh,] adUnitId…, adPlacementId: String, isVisible…, autoRefresh = true, timeToReload: Long = 20_000, reloadOnResume = true)`. Android bổ sung extension `NativeAdFactory.create(canShowAds, primaryAdUnitId, secondaryAdUnitId, canShowAdsHigh, adPlacementId, nativeLayoutId, isVisible, autoRefresh, timeToReload = 20_000, reloadWhenResume): AdmobNativeXmlConfig` cho view XML.

#### `InterstitialAdFactory` / `RewardAdFactory` / `AppOpenAdFactory` / `InterstitialRewardAdFactory` (object)

Mỗi factory có 3 overload `create` giống nhau (1 / 2 / 3 ad unit, cùng quy tắc `canShowAds*`), trả về lần lượt `InterstitialAdRequest`, `RewardAdRequest`, `AppOpenAdRequest`, `InterstitialRewardAdRequest` (hoặc biến thể `*StrategyRequest`).

#### `AppLovinBannerFactory` (object, `com.wayad.applovin.factory`)

| Hàm | Chữ ký | Trả về |
|---|---|---|
| `create` | `(canShowAds = true, adUnitId: String, adSize = ADAPTIVE, collapsePosition = null)` | `BannerAdRequest` |
| `create` | `(canShowAds = true, adUnitId, adPreloadConfig: AdPreloadConfig?, autoRefresh = true, isVisible = canShowAds, adSize, collapsePosition, timeToReload = 20_000, reloadOnResume = autoRefresh)` | `AppLovinBannerConfig` |

#### `AppLovinNativeFactory` (object)

| Hàm | Chữ ký | Trả về |
|---|---|---|
| `create` | `(canShowAds = true, adUnitId: String)` | `NativeAdRequest` |
| `create` | `(canShowAds = true, adUnitId, adPreloadConfig: AdPreloadConfig?, autoRefresh = true, isVisible = canShowAds, timeToReload = 20_000, reloadOnResume = autoRefresh)` | `AppLovinNativeConfig` |

---

### `PreloadManager` (object, `com.wayad.common`)

Helper preload theo `AdViewConfig`. 2 hàm cũ theo key (`preloadAd`, `preloadAdIfEmpty`) đã bị **xóa hẳn** — preload theo key dùng trực tiếp `NetworkAdapterScope.preloadAd/preloadIfEmpty`.

#### `fun <T : CoreAdRequest> preloadAdByConfig(adViewConfig: AdViewConfig<T>)`

Nếu `adViewConfig.adPreloadConfig` khác null và `enable = true` → gọi `adapterScope.preloadAd(adPlacementId, adRequest)` trên đúng scope của config; ngược lại chỉ log. **Trả về:** `Unit`.

#### `fun <T : CoreAdRequest> preloadAdByConfigIfEmpty(adViewConfig: AdViewConfig<T>)`

Như trên nhưng dùng `preloadIfEmpty` với `canLoadAds = adViewConfig.isVisible`. **Trả về:** `Unit`.

### `CoreAdPreload<T, R>` (interface, `com.wayad.core.preload_manager`) và `BaseAdPreload<T, R>` (abstract class)

Contract + hiện thực chung của hệ preload theo placement. Các lớp cụ thể (`BannerAdPreload`, `NativeAdPreload`…) là public nhưng gắn **`@InternalWayAdApi`** (opt-in mức ERROR) — chúng tồn tại để các module network (`:wayAd-admob`, `:wayAd-applovin`) dùng chéo module, **không phải API cho app**; app dùng gián tiếp qua `NetworkAdapterScope.preloadAd/preloadIfEmpty` và `manager.show(keyPreload, ...)`. Hành vi theo `BaseAdPreload`:

| Hàm | Chữ ký | Hành vi |
|---|---|---|
| `preloadAd` | `(placementId: String, adRequest: R)` | Bỏ qua nếu có subscription. Cancel job cũ đang active, load mới (hỗ trợ strategy request), đẩy kết quả vào buffer `ArrayDeque` theo placement, cập nhật state `LOADING → LOADED/FAILED/IDLE`. |
| `cancelPreload` | `(placementId)` | Cancel job, state về `IDLE`. |
| `isPreloading` | `(placementId): Boolean` | State hiện tại `== LOADING`. |
| `isAdAvailable` | `(placementId): Boolean` | Buffer khác rỗng. |
| `observePreloadState` | `(placementId): Flow<AdPreloadState>` | `BaseAdPreload` trả về `StateFlow`. |
| `getBufferedAds` | `(placementId): List<T>` | Snapshot buffer (rỗng nếu chưa có). |
| `popBufferedAd` | `(placementId): T?` | Lấy + xóa phần tử đầu buffer; **`null` khi buffer rỗng hoặc có subscription**. Buffer cạn → state `IDLE`. |
| `clearBufferedAds` | `(placementId)` | Xóa buffer, state `IDLE`. |
| `awaitAd` | `suspend (placementId, timeoutMillis: Long? = null): T?` | Trả ngay ad trong buffer nếu có; nếu đang `LOADING` thì chờ load xong (có timeout) rồi pop. **`null`** khi: có subscription, hết timeout, hoặc load fail/cancel. Không throw. |
| `preloading` | `suspend (placementId, timeout: Long? = null): T?` | Alias của `awaitAd`. |
| `canAwaitAd` | `(placementId): Boolean` | `isPreloading \|\| isAdAvailable`. |
| `preloadIfEmpty` | `(placementId, adRequest, canLoadAd: Boolean)` | Chỉ `preloadAd` khi buffer rỗng và `canLoadAd`. |

---

### Strategy / waterfall (`com.wayad.core.strategy`)

#### `AdStrategyLoader<AdRequest, AdResult>` (interface)

`suspend fun asyncLoadAd(request: AdRequest): Result<AdResult>` — contract loader waterfall. Các hiện thực (`BannerAdStrategyLoader`…) public nhưng gắn **`@InternalWayAdApi`** (dành cho các module network, không phải API cho app); app kích hoạt waterfall bằng cách truyền `*StrategyRequest` cho manager/preload/state.

#### `suspend fun <T : CoreAdRequest, R : CoreAdResult> List<T>.executeLoadAd(block: suspend (request: T) -> Result<R>): Result<R>` (top-level, `com.wayad.core.strategy.ads`)

Chạy `block` **tuần tự** trên từng request trong list, trả về `Result.success` đầu tiên; nếu tất cả fail, trả `Result.failure` của lần cuối (list rỗng → failure "No ad unit to load").

#### `BannerAdStrategyRequest` / `NativeAdStrategyRequest` / `InterstitialAdStrategyRequest` / `RewardAdStrategyRequest` / `AppOpenAdStrategyRequest` / `InterstitialRewardAdStrategyRequest` (data class)

Request nhiều ad unit (waterfall theo thứ tự trong `adUnitIds`, **toàn bộ** phần tử đều được thử — không giới hạn 3). Kế thừa request cùng loại (`adUnitId` = phần tử đầu) nên truyền được vào mọi API nhận request thường. Mỗi class có constructor phụ `(adUnitId1, adUnitId2?, adUnitId3?)`, `(adUnitId1, adUnitId2?)`, `(adUnitId1)` (banner thêm `adSize`, `collapsePosition`).

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `adUnitIds` | `NonEmptyList<String>` | Danh sách ad unit theo thứ tự ưu tiên. |
| `adUnitId1` | `String` | = `adUnitIds.head`. |
| `adUnitId2`, `adUnitId3` | `String?` | Phần tử 2/3 nếu có. |

#### `NonEmptyList<T>` (data class, `com.wayad.core.strategy.collections`)

List bất biến đảm bảo không rỗng, implement `List<T>` và `Comparable<List<T>>`.

| Thành viên | Chữ ký | Mô tả |
|---|---|---|
| constructor | `(head: T, tail: List<T> = emptyList())`, `(head: T, vararg tail: T?)` | Vararg tự lọc null. |
| `head` / `tail` | `T` / `List<T>` | Phần tử đầu / phần còn lại. |
| `asList` | `(): List<T>` | Chuyển về `List` thường. |
| `Companion.of` | `(head: T, vararg tail: T): NonEmptyList<T>` | Factory. |
| `Companion.fromList` | `(list: List<T>): NonEmptyList<T>?` | `null` nếu list rỗng. |

---

### Adapter SPI (`com.wayad.core.adapter`)

#### `AdNetworkAdapter` (interface)

Contract để tích hợp một ad network. Hàm chính: `getName(): String`, `initialize(callback: AdNetworkAdapterLoadCallback? = null)`, 6 hàm `load*Ad(request, callback)` (banner, native, interstitial, reward, interstitialReward, appOpen — kết quả trả qua `CoreAdLoadCallback`), 4 hàm `show*Ad(result, callback: AdFullScreenShowCallback, activity: Any? = null)` và `onAdReleased(adUnitId: String)` (default rỗng — hook dọn tài nguyên khi view bị release, iOS dùng để xóa delegate).

#### `AdNetworkAdapterLoadCallback` (interface)

`fun onInitializeSuccess()` / `fun onInitializeFailed(error: AdError)` — kết quả `initialize`.

#### `AdmobNetworkAdapter` (expect class, `com.wayad.admob`) / `AppLovinMaxNetworkAdapter` (expect class, `com.wayad.applovin`)

Hiện thực `AdNetworkAdapter` cho AdMob (`getName() = "AdmobNetworkAdapter"`) và AppLovin MAX (`getName() = "AppLovinMaxAdapter"`). Hành vi chung của cả hai, cả 2 platform:

- Mọi `load*` fail sớm khi `adUnitId` blank hoặc user có subscription (xem đầu tài liệu); trước khi load thật sự đều `WayAdKit.waitForInitialization()` (chờ `isKitInitialized == true`).
- Mọi `show*` fail (`onAdFailedToShow`) khi sai kiểu result, có subscription, hoặc (Android) không tìm được `Activity`.
- Doanh thu ad được tự động track sang Adjust (event token `tokenAdImpression` trong `WayAdConfig`).
- AppLovin: `loadInterstitialRewardAd`/`showInterstitialRewardAd` luôn fail với message "AppLovin Max does not support rewarded interstitial ads...".

---

### Compose UI

#### `BannerAdState` (class @Stable, `com.wayad.view.banner`)

State holder cho banner: tự chờ quảng cáo fullscreen đóng rồi mới load, ưu tiên lấy từ preload (theo `config.adPreloadConfig`) trước khi load qua strategy loader, tự refresh sau mỗi impression (`config.autoRefresh` + `config.timeToReload`, load lại qua debounce 3s), tự reload khi host resume (`config.reloadOnResume`, grace 500ms).

| Thành viên | Kiểu / Chữ ký | Mô tả |
|---|---|---|
| constructor | `(config: AdViewConfig<BannerAdRequest>, coroutineScope: CoroutineScope, keyDebug: String? = null)` | Thường dùng `rememberBannerAdState` thay vì tự tạo. |
| `isLoading` | `Boolean` (Compose state) | Đang load. |
| `hasAd` | `Boolean` | Đã có kết quả. |
| `error` | `AdError?` (Compose state) | Lỗi load gần nhất. |
| `isShow` | `Boolean` (Compose state) | Cờ hiển thị (khởi tạo = `config.isVisible`). |
| `adLoaded` | `SharedFlow<BannerAdResult>` | Phát khi load thành công. |
| `adFailedToLoad` | `SharedFlow<AdError>` | Phát khi load fail. |
| `adImpression` / `adClicked` | `SharedFlow<Unit>` | Sự kiện impression / click. |
| `adFailedToShow` | `SharedFlow<AdError>` | Show fail (kể cả khi gọi load lúc `isShow = false`). |
| `fun loadAd()` | | Load ngay (bỏ qua nếu state đã destroy; nếu `isShow = false` thì phát `adFailedToShow`). |
| `fun loadWithDebounce(debounceTimeMs: Long)` | | Chỉ load nếu đã qua `debounceTimeMs` kể từ impression gần nhất. |
| `fun setVisible(visible: Boolean)` | | Đổi hiển thị; `true` → clear kết quả cũ + load lại; `false` → cancel toàn bộ job và clear. |
| `fun onHostResumed()` / `fun onHostPaused()` | | Hook lifecycle (đã được `rememberBannerAdState` gắn sẵn). |
| `fun destroy()` | | Hủy state, cancel job, clear kết quả. |
| `fun cancelReloadJob()` | | Hủy job auto-refresh hiện tại. |

#### `@Composable fun rememberBannerAdState(config: AdViewConfig<BannerAdRequest>, keyDebug: String? = null): BannerAdState`

Tạo và remember `BannerAdState` theo nội dung config (đổi ad unit/cấu hình → tạo state mới), tự gắn lifecycle observer (resume/pause/destroy) và tự `destroy()` khi rời composition. **Không tự gọi `loadAd()`** — app chủ động gọi.

#### `@Composable expect fun BannerAdLayout(state: BannerAdState, modifier: Modifier = Modifier)`

Render banner: đang load → `BannerShimmer`; có kết quả → nhúng `AdView` (AdMob) hoặc `MaxAdView` (AppLovin) vào Compose (`AndroidView`/`UIKitView`). Android tự `resume()/pause()` AdView theo lifecycle; iOS tự set `rootViewController` và gọi `adapter.onAdReleased` khi release.

#### `@Composable fun BannerShimmer(modifier: Modifier = Modifier)`

Placeholder shimmer cho banner (dùng độc lập được).

#### `@Composable fun BannerAdEffect(bannerAdState: BannerAdState, onAdLoaded: (BannerAdResult) -> Unit = {}, onAdFailedToLoad: (AdError) -> Unit = {}, onAdImpression: () -> Unit = {}, onAdClicked: () -> Unit = {}, onAdFailedToShow: (AdError) -> Unit = {})`

Collect toàn bộ SharedFlow sự kiện của state thành callback — cách nhận sự kiện khuyến nghị trong Compose.

#### `fun BannerAdState.createLifecycleObserver(onDestroy: (() -> Unit)? = null): LifecycleEventObserver`

Tạo observer map `ON_RESUME/ON_PAUSE/ON_DESTROY` → `onHostResumed()/onHostPaused()/destroy()` (dùng khi tự quản lý state ngoài `rememberBannerAdState`).

#### `NativeAdState` (class @Stable, `com.wayad.view.nativead`)

Tương tự `BannerAdState` cho native ad (cùng bộ property/hàm: `isLoading`, `nativeResult: NativeAdResult?`, `error`, `isShow`, 5 SharedFlow, `loadAd`, `loadWithDebounce`, `setVisible`, `onHostResumed`, `onHostPaused`, `destroy`, `cancelReloadJob`). Thêm:

#### `fun enableUseCaseReloadWhenVideoEnd(enable: Boolean)`

Bật/tắt use-case (mặc định bật): trên **Android**, `AdmobNativeLayout` sẽ tự `loadWithDebounce(5_000)` khi video của native ad phát xong (nếu host đang RESUMED). **Trả về:** `Unit`.

#### `@Composable fun rememberNativeAdState(config: AdViewConfig<NativeAdRequest>, keyDebug: String? = null): NativeAdState`

Như `rememberBannerAdState` cho native.

#### `@Composable expect fun AdmobNativeLayout(state: NativeAdState, modifier: Modifier = Modifier, content: @Composable AdmobNativeLayoutScope.() -> Unit)`

Container render native ad **AdMob** với layout tự do qua `content` (scope cung cấp các thành phần ad). Đang load → `NativeShimmer`. Nếu state đang giữ `AppLovinNativeResult` → không render (log warning, dùng `MaxNativeLayout` thay thế).

#### `AdmobNativeLayoutScope` (expect class @Stable)

Scope của `content` trong `AdmobNativeLayout`; property chung `state: NativeAdState`. Thành phần con theo platform:

- **Android** (extension @Composable trên scope, tự bind vào `NativeAdView` thật): `MediaView`, `Headline(maxLines = 1, style)`, `Body(maxLines = 3, style)`, `CallToAction(style)` (fallback text "Install"), `Advertiser(style)`, `Price(style)`, `Store(style)`, `StarRating(style)`, `Icon()`. Mỗi thành phần tự hiện shimmer khi đang load và ẩn khi ad không có dữ liệu tương ứng.
- **iOS** (template dựng sẵn bằng UIKit): `SmallNativeAdTemplate(modifier, backgroundColor, colorBackgroundCta, colorTitleColor, colorHeadline, colorBody)` và `NativeAdFullscreenTemplate(...)` — không hỗ trợ compose từng thành phần lẻ như Android.

#### `@Composable fun NativeShimmer(modifier: Modifier = Modifier)`

Placeholder shimmer cho native ad.

#### `@Composable fun NativeAdEffect(nativeAdState: NativeAdState, onAdLoaded, onAdFailedToLoad, onAdImpression, onAdClicked, onAdFailedToShow)`

Như `BannerAdEffect` cho native.

#### `fun NativeAdState.createLifecycleObserver(onDestroy: (() -> Unit)? = null): LifecycleEventObserver`

Như bản banner.

#### `@Composable expect fun MaxNativeLayout(state: NativeAdState, spec: MaxNativeRenderSpec, modifier: Modifier = Modifier)` (`com.wayad.applovin.view`)

Render native ad **AppLovin MAX** theo `MaxNativeRenderSpec`. Nếu state giữ result không phải `AppLovinNativeResult` → không render (log warning). Mỗi platform có thêm overload riêng: Android `MaxNativeLayout(state, binder: MaxNativeAdViewBinder, modifier)`; iOS `MaxNativeLayout(state, template: String? = null, modifier)`.

#### `@Composable fun FloatingNative(...)` (2 overload, `com.wayad.view.floating`)

Overlay native ad nổi, kéo-thả được, có nút đóng "×":

- `FloatingNative(modifier, nativeAdState: NativeAdState?, initialAlignment: Alignment = BottomEnd, edgePadding: Dp = 16.dp, cornerRadius: Dp = 12.dp, reload: Long? = null, layoutAd: @Composable AdmobNativeLayoutScope.() -> Unit, content: @Composable () -> Unit)` — `content` là UI màn hình bên dưới; `layoutAd` là layout của ad. Nút đóng: nếu `reload == null` → `setVisible(false)` (ẩn hẳn); nếu có → chỉ ẩn tạm và tự hiện lại sau `reload` ms.
- `FloatingNative(modifier, nativeAdState: NativeAdState, initialAlignment, edgePadding, size: Dp = 140.dp, cornerRadius, backgroundColor: Color = Black, ctaColor: Color = 0xFFFF6A00, ctaTextColor: Color = White, reload: Long? = null, content)` — dùng template dựng sẵn `FloatingTemplateNativeAdContent`.

#### `@Composable expect fun AdmobNativeLayoutScope.FloatingTemplateNativeAdContent(modifier, size: Dp = 140.dp, cornerRadius: Dp = 12.dp, backgroundColor: Color = Black, ctaColor: Color = 0xFFFF6A00, ctaTextColor: Color = White)`

Template nội dung vuông nhỏ (media + CTA) cho floating native, có actual trên cả hai platform.

---

### View XML (chỉ Android, `com.wayad.view.*`, `com.wayad.applovin.view.xml`)

Dành cho app Android không dùng Compose. Các view tự load theo config, tự shimmer, tự refresh/reload-on-resume, chờ fullscreen ad đóng trước khi load — giống hành vi của `BannerAdState`/`NativeAdState`.

#### `LayoutAdView<T, R, C>` (abstract class)

`FrameLayout` cơ sở. Thành viên public: `currentAdResult: R?`, `config: C?`, `interceptAutoRefresh: Boolean` (cờ CHO PHÉP — default `true`; auto-refresh/reload-on-resume chỉ chạy khi `config.autoRefresh`/`config.reloadOnResume` VÀ cờ này cùng `true`. ⚠️ Lưu ý: một số path reload trong subclass — `AdmobBannerAdView.onHostResume`/auto-refresh sau impression, `AdmobNativeAdView` reload sau video, `AppLovinNativeAdView` reload — hiện chỉ kiểm tra `config`, KHÔNG kiểm tra cờ này), `fun setParamConfig(config: C)` (chỉ gán config và áp `config.isVisible` — **KHÔNG tự load**; sau khi set config phải gọi `loadAd()` để load lần đầu. Load tự động chỉ xảy ra ở path reload-on-resume — từ lần resume thứ 2 trở đi khi `config.reloadOnResume = true` — và auto-refresh sau impression), `fun loadAd()` (yêu cầu đã có config và `config.isVisible = true`, nếu không callback nhận `onAdFailedToLoad`), `abstract fun showAd()`, `fun setDebugTag(tagDebug: String)`.

#### `AdmobBannerAdView` (class)

Banner AdMob dạng XML — set `AdmobBannerConfig` qua `setParamConfig`. Thêm: `fun loadWithDebounce(debounceTimeMs: Long)`, `fun setLoadCallback(callback: BannerAdLoadCallback)`, `fun setShowCallback(callback: BannerAdShowCallback)`.

#### `AdmobNativeAdView` (open class)

Native AdMob dạng XML — dùng `AdmobNativeXmlConfig` (chứa `nativeLayoutId` là layout `NativeAdView`). Thêm: `fun showShimmer()`, `fun loadWithDebounce(debounceTimeMs: Long)`, `fun setLoadCallback(callback: NativeAdLoadCallback)`, `fun setShowCallback(callback: NativeAdShowCallback)`, `fun enableUseCaseReloadWhenVideoEnd(enable: Boolean)` (bật/tắt use case tự reload native khi video ad kết thúc — mặc định **bật**).

#### `FloatingNativeAdView` (class)

Kế thừa `AdmobNativeAdView`, thành floating view kéo-thả với nút đóng; hỗ trợ style qua XML attrs (`FloatingNativeAdView`) hoặc các setter runtime tương ứng: `fun setEnableDrag(enable: Boolean)`, `fun setEnableClose(enable: Boolean)` (áp ngay nút đóng nếu ad đang hiển thị), `fun setEnableAnimation(enable: Boolean)` (default `true`), `fun setReloadAfterCloseMs(ms: Long)` (số ms trước khi ad tự hiện lại sau khi user bấm đóng; `0` — default — là ẩn hẳn), `fun setEdgePaddingPx(px: Int)` (margin so với mép, default `0`), `fun setInitialAlignment(alignment: Int)` (vị trí neo ban đầu theo hằng alignment, áp ngay nếu view đã attach).

#### `FloatingNativeAdTemplate` (object)

Dựng `NativeAdView` template floating bằng code: `fun build(context: Context, options: Options = Options()): NativeAdView`, `fun applyTo(nativeAdView: NativeAdView, options: Options = Options())`; `Options(sizePx, cornerRadiusPx, backgroundColor, ctaColor, ctaTextColor, showAdBadge, showBottomGradient)`.

#### `AppLovinBannerAdView` / `AppLovinNativeAdView` (class / open class)

Bản AppLovin của 2 view trên, luôn load qua `AppLovinMaxNetworkAdapterScope`. `AppLovinBannerAdView` thêm: `fun setLoadCallback(callback: BannerAdLoadCallback)`, `fun setShowCallback(callback: BannerAdShowCallback)`. `AppLovinNativeAdView` thêm: `fun setBinder(binder: MaxNativeAdViewBinder)` (bắt buộc trước khi show), `fun setShimmerLayout(@LayoutRes layoutId: Int)`, `fun setLoadCallback(callback: NativeAdLoadCallback)` (không có show callback riêng). Cả hai thêm `fun destroy()` để chủ động giải phóng.

#### `AdmobLoadingAdDialog` (class)

Dialog loading đơn giản dùng khi chờ ad: `fun setMessage(message: String)`, `fun showLoading()`, `fun hideLoading()`. Companion:

#### `suspend fun <T> AdmobLoadingAdDialog.Companion.withLoadingAd(context: Context? = LifecycleProvider.context, message: String = "Loading Ad...", block: suspend () -> T): T`

Hiện dialog, delay 200ms, chạy `block`, luôn ẩn dialog ở `finally`; nếu `context == null` thì chạy `block` trực tiếp. **Trả về:** kết quả của `block`.

---

### Tiện ích khác

#### `BannerAdUtils` (object, `com.wayad.core.utils`)

`fun getAdSizeDimensions(adSize: BannerAdSize): Pair<Int, Int>` — trả về `(width, height)` dp danh nghĩa của từng `BannerAdSize` (ADAPTIVE/FLUID/INVALID trả 320×50; INLINE_ADAPTIVE trả 320×700).

#### `CallbackDispatcher<C>` (class, `com.wayad.core.api.callback`)

Giữ tối đa một callback kiểu `C`: `fun setCallback(callback: C?)`, `fun invokeCallback(action: C.() -> Unit)` (không làm gì nếu chưa set). Xuất hiện trong các `*Result` của banner/native để view gắn show-callback.

#### `fun <T> Continuation<T>.safeResume(value: T)` (`com.wayad.extension`)

Resume continuation an toàn (chỉ khi context còn active, nuốt exception).

#### `expect fun <R> synchronized(lock: Any, block: () -> R): R` (`com.wayad.extension`)

Khóa mutual-exclusion đa nền tảng (Android dùng `kotlin.synchronized`).

#### `enum TypePlatform { ANDROID, IOS }` + `expect fun getCurrentPlatform(): TypePlatform` (`com.wayad.common.utils`)

Nhận diện platform hiện tại.

> `AttPermissionRequester` / `AttState` (iOS only) đã chuyển sang module `adjust_kmp`, package `com.adjust_kmp.att` — vẫn dùng được từ `wayAd` vì `wayAd` expose `api(project(":adjust_kmp"))`, nhưng import cũ `com.wayad.common.utils.att.*` không còn.

#### `fun NativeAdState.getNativeAd()` (extension, mỗi platform)

Android trả `com.google.android.gms.ads.nativead.NativeAd?`, iOS trả `GADNativeAd?` — lấy đối tượng native ad gốc từ `nativeResult` (null nếu chưa load hoặc result không phải AdMob). Cùng package `com.wayad.admob.extension` còn có helper chuyển đổi `BannerAdSize.asAdmobAdSize()` cho từng platform.

---

## Public models

### `WayAdConfig` (data class, `com.wayad.common.model`)

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `tokenAdjust` | `String` | App token Adjust (bắt buộc). |
| `tokenAdImpression` | `String?` | Event token Adjust để track doanh thu ad impression. Mặc định `null`. |
| `testDeviceIds` | `List<String>` | Test device id cho UMP consent debug (rỗng → dùng danh sách mặc định của SDK). |
| `preloadAwaitTimeoutMs` | `Long` | Timeout chờ ad preload trong `manager.show(keyPreload, ...)`. Mặc định `30_000`. |
| `appLovinSdkKey` | `String?` | SDK key AppLovin — bắt buộc nếu đăng ký `AppLovinMaxNetworkAdapterScope`. |

### Request (`com.wayad.core.api.*`)

`interface CoreAdRequest { val adUnitId: String }`; `InScreenAdRequest` (banner, native) và `FullScreenAdRequest` (interstitial, reward, app open, interstitial-reward) là marker kế thừa nó.

| Model | Field | Ý nghĩa |
|---|---|---|
| `BannerAdRequest` (open class) | `adUnitId: String`, `adSize: BannerAdSize = ADAPTIVE`, `collapsePosition: CollapsePosition? = null` | Request banner; `collapsePosition != null` → banner collapsible (AdMob). |
| `NativeAdRequest` (open class) | `adUnitId: String` | Request native. |
| `InterstitialAdRequest` (open class) | `adUnitId: String` | Request interstitial. |
| `RewardAdRequest` (open class) | `adUnitId: String` | Request rewarded. |
| `InterstitialRewardAdRequest` (open class) | `adUnitId: String` | Request rewarded-interstitial (chỉ AdMob hỗ trợ). |
| `AppOpenAdRequest` (open class) | `adUnitId: String` | Request app open. |

### Result (`com.wayad.core.api.*`, `com.wayad.admob.model`, `com.wayad.applovin.model`)

`interface CoreAdResult { val adUnitId: String }`; mỗi loại ad có interface con: `BannerAdResult`, `NativeAdResult`, `InterstitialAdResult`, `RewardAdResult`, `InterstitialRewardAdResult`, `AppOpenAdResult`. Trong đó **`BannerAdResult` và `NativeAdResult` không rỗng** — cả hai bắt buộc `val callbackDispatcher: CallbackDispatcher<*AdShowCallback>` (state attach show-callback qua đây, không type-switch theo network); 4 interface còn lại rỗng. Riêng banner còn có `interface BannerAdRenderer { @Composable fun Render(modifier) }` (cùng file `BannerAdResult.kt`): banner result của một network **phải implement thêm** interface này thì `BannerAdLayout` mới render được — result không implement sẽ chỉ log warning và không hiển thị gì. Hiện thực cụ thể (generic `T` là đối tượng ad gốc của SDK network trên từng platform):

| Model | Field chính |
|---|---|
| `AdmobBannerResult<T>` | `adView: T`, `adUnitId`, `callbackDispatcher: CallbackDispatcher<BannerAdShowCallback>` |
| `AdmobNativeResult<T>` | `nativeAd: T`, `adUnitId`, `callbackDispatcher: CallbackDispatcher<NativeAdShowCallback>` |
| `AdmobInterstitialResult<T>` | `interstitialAd: T`, `adUnitId` |
| `AdmobInterstitialRewardResult<T>` | `interstitialRewardAd: T`, `adUnitId` |
| `AdmobRewardResult<T>` | `rewardedAd: T`, `adUnitId` |
| `AdmobAppOpenResult<T>` | `appOpenAd: T`, `adUnitId` |
| `AppLovinBannerResult<T>` | `adView: T`, `adUnitId`, `callbackDispatcher` |
| `AppLovinNativeResult<TLoader, TAd>` | `loader: TLoader`, `nativeAd: TAd`, `adUnitId`, `callbackDispatcher` |
| `AppLovinInterstitialResult<T>` | `interstitialAd: T`, `adUnitId`, `showCallbackDispatcher: CallbackDispatcher<AdFullScreenShowCallback>` |
| `AppLovinRewardResult<T>` | `rewardedAd: T`, `adUnitId`, `showCallbackDispatcher` |
| `AppLovinAppOpenResult<T>` | `appOpenAd: T`, `adUnitId`, `showCallbackDispatcher` |

### Callback (`com.wayad.core.api.callback` và các package con)

| Type | Thành viên | Ý nghĩa |
|---|---|---|
| `CoreAdLoadCallback<T>` (interface) | `onAdLoaded(result: T)`, `onAdFailedToLoad(error: AdError)` | Kết quả load. Các alias theo loại: `BannerAdLoadCallback`, `NativeAdLoadCallback`, `InterstitialAdLoadCallback`, `RewardAdLoadCallback`, `InterstitialRewardAdLoadCallback`, `AppOpenAdLoadCallback`. |
| `CoreAdShowCallback` (abstract class) | `onAdImpression()`, `onAdClicked()`, `onAdFailedToShow(error: AdError)` | Sự kiện hiển thị (mặc định rỗng). Alias: `BannerAdShowCallback`, `NativeAdShowCallback`. |
| `AdFullScreenShowCallback` (abstract class) | thêm `onAdClosed()` (mặc định gọi `onNextAction(true)`), `onNextAction(isSuccess: Boolean)`; `onAdFailedToShow` mặc định gọi `onNextAction(false)` | Cho ad fullscreen. Alias: `InterstitialAdShowCallback`, `AppOpenAdShowCallback`. |
| `RewardAdShowCallback` (abstract class) | thêm `onUserEarnedReward(amount: Int, type: String)` | Cho rewarded. `InterstitialRewardAdShowCallback` kế thừa nó. |

### `AdError` / `AdErrorException` (`com.wayad.core.api.error`)

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `AdError.error` | `String` | Message lỗi. |
| `AdError.code` | `Int?` | Mã lỗi từ SDK network (nếu có). |
| `AdErrorException.error` | `AdError` | Exception bọc `AdError` (message = `error.error`). |

### `BannerAdSize` (enum, `com.wayad.core.api.banner`)

| Giá trị | Ý nghĩa (kích thước danh nghĩa theo `BannerAdUtils`) |
|---|---|
| `BANNER` | 320×50. |
| `FULL_BANNER` | 468×60. |
| `LARGE_BANNER` | 320×100. |
| `LEADERBOARD` | 728×90. |
| `MEDIUM_RECTANGLE` | 300×250. |
| `WIDE_SKYSCRAPER` | 160×600. |
| `FLUID` | Kích thước linh hoạt theo container. |
| `ADAPTIVE` | Anchored adaptive banner theo bề rộng màn hình — **mặc định** (`getDefault()`). |
| `INLINE_ADAPTIVE` | Inline adaptive, cao tối đa `INLINE_HEIGHT_MAX = 700`. |
| `INVALID` | Không hợp lệ (fallback). |

### `CollapsePosition` (enum, `com.wayad.core.api.banner`)

| Giá trị | Ý nghĩa |
|---|---|
| `TOP` | Banner collapsible neo mép trên. |
| `BOTTOM` | Banner collapsible neo mép dưới. |

### `AdPreloadState` (enum, `com.wayad.core.preload_manager`)

| Giá trị | Ý nghĩa |
|---|---|
| `IDLE` | Chưa/không còn gì trong buffer, không load. |
| `LOADING` | Đang preload. |
| `LOADED` | Load xong, buffer có ad. |
| `FAILED` | Lần preload gần nhất thất bại. |

### `AdViewConfig<T : CoreAdRequest>` (interface, `com.wayad.model`) và `AdPreloadConfig`

Config cho các view/state tự quản lý (banner, native):

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `adRequest` | `T` | Request (thường/strategy). |
| `adPreloadConfig` | `AdPreloadConfig?` | `AdPreloadConfig(enable: Boolean, adPlacementId: String)` — nếu enable, state sẽ ưu tiên lấy ad từ buffer preload theo `adPlacementId` trước khi load mới. |
| `autoRefresh` | `Boolean` | Tự load lại sau mỗi impression. |
| `isVisible` | `Boolean` | Trạng thái hiển thị ban đầu. |
| `timeToReload` | `Long` | Khoảng chờ (ms) sau impression trước khi auto-refresh (mặc định các config: `20_000`). |
| `reloadOnResume` | `Boolean` | Reload khi host resume (mặc định = `autoRefresh`). |
| `adapterScope` | `NetworkAdapterScope` | Scope network mà config này thuộc về (quyết định adapter nào load). |

Hiện thực: `AdmobBannerConfig`, `AdmobNativeConfig` (scope AdMob); `AppLovinBannerConfig`, `AppLovinNativeConfig` (scope AppLovin); Android-only `AdmobNativeXmlConfig` (kế thừa `AdmobNativeConfig`, thêm `nativeLayoutId: Int` — layout resource cho `AdmobNativeAdView`).

### `MaxNativeRenderSpec` (expect class, `com.wayad.applovin.model`)

Chỉ định cách render native AppLovin, khác nhau theo platform:

- **Android:** `MaxNativeRenderSpec(binder: MaxNativeAdViewBinder)`; factory `Companion.fromLayout(@LayoutRes layoutResId, configure: MaxNativeAdViewBinder.Builder.() -> Unit)`.
- **iOS:** `MaxNativeRenderSpec(template: String?)` với preset: `Default` (null), `MediaBanner`, `SmallTemplate1`, `MediumTemplate1`, `VerticalMediaBanner`, và `Custom(template: String)`.

### `AdTestIds` (expect object, `com.wayad.admob.model`)

Ad unit id test chính thức của Google cho từng platform: `APP_OPEN`, `BANNER`, `BANNER_ADAPTIVE`, `INTERSTITIAL`, `INTERSTITIAL_VIDEO`, `NATIVE`, `NATIVE_VIDEO`, `REWARDED`, `REWARDED_INTERSTITIAL` (đều `String`).

---

## Lưu ý platform

- **Consent & khởi tạo:** overload `init` có `onComplete` chạy UMP consent trên cả 2 platform; riêng iOS còn chạy Adjust + ATT trong chuỗi này. `isKitInitialized` khởi tạo `true` trên Android nhưng `false` trên iOS — trên iOS **phải gọi `init` trước** thì adapter mới load ad (mọi `load*` đều chờ flow này). `awaitConsentInfoUpdate` trả giá trị khác nhau khi consent đã có sẵn (Android `false`, iOS `true`).
- **Ad resume (`registerAdResume`):** chỉ Android. iOS là no-op.
- **`activity: Any?` trong các hàm show:** Android cast sang `Activity`, null thì fallback `LifecycleProvider.activity` (vẫn null → `onAdFailedToShow`). iOS bỏ qua/tự tìm root view controller.
- **Rewarded-interstitial:** chỉ AdMob. AppLovin adapter fail cả load lẫn show trên cả 2 platform.
- **AppLovin native/banner view:** Android render qua `MaxNativeAdViewBinder` (layout XML); iOS render qua template string của MAX. `MaxNativeRenderSpec` che khác biệt này ở tầng Compose.
- **Native ad layout AdMob:** Android compose từng thành phần (`Headline`, `MediaView`, `CallToAction`…) bind vào `NativeAdView` thật; iOS chỉ có 2 template UIKit dựng sẵn (`SmallNativeAdTemplate`, `NativeAdFullscreenTemplate`) — `content` không thể tự do như Android. Android còn tự reload native khi video kết thúc (tắt bằng `enableUseCaseReloadWhenVideoEnd(false)`).
- **View XML** (`LayoutAdView`, `AdmobBannerAdView`, `AdmobNativeAdView`, `AppLovin*AdView`, `FloatingNativeAdView`, `AdmobLoadingAdDialog`, `AdmobNativeXmlConfig`) chỉ có trên Android.
- **iOS-only:** các UIView public `ProgrammaticNativeAdView`, `ProgrammaticFullscreenAdView`, `ProgrammaticFloatingAdView` (được các template dùng nội bộ, ít khi gọi trực tiếp).
- **Giải phóng tài nguyên iOS:** các layout Compose iOS gọi `adapter.onAdReleased(adUnitId)` khi view release để xóa delegate/loader — nếu tự nhúng view gốc, cần tự gọi tương tự.

## Log TAG

Mọi log của module dùng prefix `WayAd/` (vd `WayAd/Kit`, `WayAd/AdmobAdapter`, `WayAd/MaxBannerView`...). Bảng đầy đủ + quy ước chung: xem `wayCore/API.md`. `setDebugTag` trên XML view thay thế TAG mặc định — nên tự giữ prefix `WayAd/`.
