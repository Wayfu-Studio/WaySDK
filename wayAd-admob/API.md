# wayAd-admob — Public API

Backend **Google AdMob** cho `:wayAd`. Tách khỏi core theo đúng mô hình `wayPay`/`wayPay-store`: `:wayAd` là module chung network-neutral, mỗi ad network là một module implement, đăng ký song song qua `WayAdKit.init(scopes = ...)`.

Phụ thuộc: `api(:wayAd)`, Android thêm `com.google.android.gms:play-services-ads` (**api** — end-user gọi được API AdMob trực tiếp, version do catalog của SDK pin, giống cách `:wayAd-applovin` expose `api(applovin-sdk)`) + **toàn bộ** stack mediation của AdMob (`com.google.ads.mediation:*`: AppLovin, Chartboost, Fyber, i-mobile, InMobi, ironSource, Vungle, Line, maio, Facebook, Mintegral, myTarget, Pangle, Unity — mỗi adapter kéo theo SDK của partner tương ứng, riêng `applovin` kéo `com.applovin:applovin-sdk`) + `:wayLayout`; iOS cinterop `GoogleMobileAds.xcframework`. Consent Google UMP **không** nằm ở đây — UMP là CMP dùng chung toàn app nên thuộc core `:wayAd`. Package chính: `com.wayad.admob` (một số API giữ package `com.wayad.common`/`com.wayad.view.*` để tương thích nguồn với code cũ).

Artifact: `way-sdk:way-ad-admob`. Luôn dùng **cùng version** với `way-ad` (module này dùng `@InternalWayAdApi` của core, không có cam kết ổn định chéo version).

## Cài đặt

```kotlin
// build.gradle.kts của app
implementation("way-sdk:way-ad-admob:<version>")   // đã api(:wayAd), không cần khai báo lại
```

Xcode: thêm SPM package GoogleMobileAds + GoogleUserMessagingPlatform với version khớp `iosGoogleMobileAds` / `iosUserMessagingPlatform` trong `gradle/libs.versions.toml` (hiện GoogleMobileAds **13.7.0**, UMP **3.1.0**) — `verifyInteropVersions` sẽ fail build nếu lệch.

Android: `play-services-ads` **25.4.0** (dòng 25.x, tương đương major 13.x của SDK iOS). Toàn bộ adapter `com.google.ads.mediation:*` được pin sao cho SDK network bên dưới trùng version với adapter MAX tương ứng ở `:wayAd-applovin` — xem chú thích trong `gradle/libs.versions.toml` trước khi bump lẻ một adapter.

## Khởi tạo

```kotlin
import com.wayad.admob.AdmobNetworkAdapterScope
import com.wayad.common.registerAdResume   // overload 1 tham số nằm trong module này

WayAdKit.init(
    wayAdConfig = WayAdConfig(tokenAdjust = "..."),
    isDebug = true,
    scopes = listOf(AdmobNetworkAdapterScope),
    onComplete = {},
)
WayAdKit.registerAdResume(AppOpenAdRequest(AdTestIds.APP_OPEN))  // Android only
```

## Những gì nằm trong module này

- `AdmobNetworkAdapterScope` (`com.wayad.admob`) — scope + `AdmobNetworkAdapter` (expect/actual Android/iOS), kèm shortcut 6 factory (`bannerFactory`, `nativeFactory`, …).
- `BannerAdFactory`, `NativeAdFactory` (`com.wayad.common.factory`) — các overload trả về `AdmobBannerConfig`/`AdmobNativeConfig`. (4 factory network-neutral còn lại — Interstitial/Reward/AppOpen/InterstitialReward — vẫn ở core.)
- Model: `AdmobBannerResult`/`AdmobNativeResult`/… (`com.wayad.admob.model`), `AdTestIds`.
- UI Compose: `AdmobNativeLayout` + `AdmobNativeLayoutScope`, `FloatingNative`, `FloatingTemplateNativeAdContent` (`com.wayad.view.*`).
- View XML Android: `AdmobBannerAdView`, `AdmobNativeAdView`, `AdmobLoadingAdDialog` (`com.wayad.view.xml`).
- **Ad resume**: overload tiện lợi `WayAdKit.registerAdResume(appOpenAdRequest)` (package `com.wayad.common`) = gọi bản core `registerAdResume(appOpenAdRequest, scope)` với `AdmobNetworkAdapterScope`. Flow ad-resume nằm ở core (network-neutral); `AdActivity` của AdMob được adapter tự đăng ký chặn trong `initialize()`, đối xứng với cách AppLovin đăng ký `AppLovinFullscreenActivity`.

## Điểm khác so với thời còn nằm trong `:wayAd`

- `WayAdKit.init` không còn default `scopes = listOf(AdmobNetworkAdapterScope)` — phải truyền tường minh (và phải thêm module này để có `AdmobNetworkAdapterScope`).
- Các thành viên deprecated mức ERROR (companion `AppOpenAdManager.show/load/...`, `PreloadManager.preloadAd/preloadAdIfEmpty`, `WayAdKit.setNetworkAdapter`) đã bị xóa hẳn khỏi core.
