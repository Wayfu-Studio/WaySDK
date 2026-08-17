# wayAd-applovin — Public API

Backend **AppLovin MAX** cho `:wayAd`. Khi không khai báo module này: framework Kotlin không sinh symbol MAX nào và app iOS bỏ được `AppLovin-MAX-Swift-Package` khỏi Xcode. **Lưu ý Android:** APK vẫn có thể chứa `com.applovin:applovin-sdk` dù không dùng module này — `:wayAd-admob` kéo nó vào qua mediation adapter `com.google.ads.mediation:applovin` (AdMob mediate AppLovin trong waterfall); cái module này quyết định là **MAX adapter code** (`com.wayad.applovin.*`, `com.applovin.mediation:*-adapter`) có mặt hay không.

Phụ thuộc: `api(:wayAd)`, Android thêm `com.applovin:applovin-sdk` + toàn bộ mediation adapter của MAX (`com.applovin.mediation:*-adapter`: Google, Facebook, Vungle, Unity, ironSource, Mintegral, InMobi, Pangle, Chartboost, Fyber, Line, myTarget); iOS cinterop `AppLovinSDK.xcframework`. Package: `com.wayad.applovin`. Lưu ý: `com.google.ads.mediation:applovin` (adapter để **AdMob** mediate AppLovin) KHÔNG nằm ở đây — nó thuộc stack mediation của Admob và nằm ở `:wayAd-admob`.

Artifact: `way-sdk:way-ad-applovin`.

## Cài đặt

```kotlin
// build.gradle.kts của app
implementation("way-sdk:way-ad-applovin:<version>")   // đã api(:wayAd), không cần khai báo lại
```

Xcode: thêm SPM package `https://github.com/AppLovin/AppLovin-MAX-Swift-Package.git` với version khớp `iosApplovinSdk` trong `gradle/libs.versions.toml` (hiện **13.6.4**) — `verifyInteropVersions` sẽ fail build nếu lệch.

## Khởi tạo

`WayAdConfig.appLovinSdkKey` là **bắt buộc** khi đăng ký scope này (thiếu → `onInitializeFailed`).

```kotlin
WayAdKit.init(
    wayAdConfig = WayAdConfig(
        tokenAdjust = "...",
        appLovinSdkKey = "<APPLOVIN_SDK_KEY>",
    ),
    isDebug = true,
    scopes = listOf(
        AdmobNetworkAdapterScope,          // bỏ nếu app chỉ chạy AppLovin
        AppLovinMaxNetworkAdapterScope,
    ),
    onComplete = {},
)
```

## Public API

Chi tiết từng symbol xem `wayAd/API.md` — các mục liên quan module này:

| Symbol | Mục trong `wayAd/API.md` |
|---|---|
| `AppLovinMaxNetworkAdapterScope`, `showMediationDebugger()` | Scope theo network |
| `AppLovinMaxNetworkAdapter` | Adapter |
| `AppLovinBannerFactory`, `AppLovinNativeFactory` | Factory tạo request/config |
| `AppLovinBannerConfig`, `AppLovinNativeConfig`, `MaxNativeRenderSpec` | Config |
| `AppLovinBannerResult`, `AppLovinNativeResult`, `AppLovinInterstitialResult`, `AppLovinRewardResult`, `AppLovinAppOpenResult` | Result |
| `MaxNativeLayout` | UI Compose |
| `AppLovinBannerAdView`, `AppLovinNativeAdView` (Android) | View XML |

Banner AppLovin render qua `BannerAdLayout` dùng chung của `:wayAd`: `AppLovinBannerResult` hiện thực `BannerAdRenderer` nên `:wayAd` dispatch tới nó mà không cần biết AppLovin.

## Giới hạn

- **Rewarded-interstitial:** AppLovin MAX không hỗ trợ — `loadInterstitialRewardAd`/`showInterstitialRewardAd` luôn fail trên cả 2 platform.
- **Native view:** Android render qua `MaxNativeAdViewBinder` (layout XML), iOS qua template string của MAX; `MaxNativeRenderSpec` che khác biệt ở tầng Compose.
- `AdmobNativeLayout` của `:wayAd` không render được `AppLovinNativeResult` (log warning) — dùng `MaxNativeLayout`.
