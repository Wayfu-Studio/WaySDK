# WaySDK

Bộ SDK Kotlin Multiplatform (Android + iOS) dùng chung cho các app của Wayfu Studio: quảng cáo (`wayAd`), thanh toán (`wayPay`), attribution cài đặt (`wayInstall`) và các module nền tảng đi kèm.

Hai loại tài liệu, đừng lẫn: `API.md` của từng module mô tả **trạng thái hiện tại** của public API (model, function, class, tham số, behaviour); [CHANGELOG.md](CHANGELOG.md) ghi **thay đổi giữa các phiên bản** và việc cần làm khi nâng cấp.

## Mô hình nhánh

Repo vận hành theo 3 nhánh `inhouse/*`, tách theo **phiên bản Google Mobile Ads SDK** mà app đích sử dụng:

| Nhánh | Vai trò | Google Ads SDK (Android) |
|---|---|---|
| `develop` | **Nơi viết logic chung nhất.** Mọi thay đổi về core (`wayAd` core, `wayPay`, `wayInstall`, module nền tảng, tài liệu API) đều viết ở đây trước. | — (không gắn với SDK cụ thể) |
| `inhouse/v2/main` | Triển khai với **AdMob thường** (legacy). | `com.google.android.gms:play-services-ads` |
| `inhouse/v3/main` | Triển khai với **AdMob Next-Gen**. | `com.google.android.libraries.ads.mobile.sdk:ads-mobile-sdk` |

Quy tắc làm việc:

- Sửa logic chung → làm trên `inhouse/develop`, sau đó merge sang `inhouse/v2/main` **và** `inhouse/v3/main`.
- Sửa code đặc thù cho một phiên bản SDK (chủ yếu nằm trong `wayAd-admob`) → làm trực tiếp trên nhánh `v2` hoặc `v3` tương ứng.
- Không merge chéo giữa `v2` và `v3`: hai nhánh khác nhau ở adapter Admob (`wayAd-admob`) — API của hai SDK Google không tương thích với nhau.

Điểm khác nhau chính giữa `v2` và `v3` khi merge từ `develop`:

- `wayAd-admob` trên `v3` viết lại toàn bộ adapter theo API Next-Gen (`com.google.android.libraries.ads.mobile.sdk.*`), bao gồm cả `AdActivity`, callback và renderer banner.
- Trên `v3`, các module chứa mediation adapter (`wayAd-admob`, `wayAd-applovin`) phải `exclude` artifact `play-services-ads` khỏi classpath — Next-Gen SDK tự đóng gói các class `com.google.android.gms.ads` tương thích, để cả hai cùng tồn tại sẽ bị lỗi duplicate class.
- `WayAdConfig` trên `v3` cần thêm `androidAppAdsId` để truyền vào `MobileAds.initialize` kiểu Next-Gen.

## Cấu trúc module

Mỗi module có tài liệu API riêng trong file `API.md` của module đó:

- `wayAd` — core quảng cáo, không phụ thuộc ad network cụ thể ([API](wayAd/API.md))
  - `wayAd-admob` — adapter AdMob ([API](wayAd-admob/API.md))
  - `wayAd-applovin` — adapter AppLovin MAX ([API](wayAd-applovin/API.md))
- `wayPay` — core thanh toán/subscription ([API](wayPay/API.md))
  - `wayPay-store` — backend Google Play Billing / StoreKit ([API](wayPay-store/API.md))
  - `wayPay-revenuecat` — backend RevenueCat ([API](wayPay-revenuecat/API.md))
- `wayInstall` — attribution nguồn cài đặt
- `wayCore`, `wayLayout`, `lifecycle_kmp`, `adjust_kmp` — module nền tảng dùng chung
- `composeApp` + `iosApp` — app demo để test SDK trên hai nền tảng

## Version SDK quảng cáo

Tất cả version của ad SDK và mediation adapter — cả Android lẫn iOS — khai báo trong `gradle/libs.versions.toml`. Hai quy tắc đọc kỹ trước khi bump, giải thích đầy đủ trong [CHANGELOG.md](CHANGELOG.md):

- **Google Mobile Ads phát hành theo cặp giữa hai platform** (Android 25.x ↔ iOS 13.x). Bump một bên thì bump luôn bên kia, tương tự với Adjust và AppLovin.
- **Adapter AdMob và adapter MAX của cùng một network phải bọc cùng version SDK network**, vì `:wayAd-admob` và `:wayAd-applovin` có thể cùng nằm trong một app.

Version SDK iOS (`iosGoogleMobileAds`, `iosUserMessagingPlatform`, `iosApplovinSdk`, `iosAdjustSdk`) được dùng chung cho cinterop bridge và SPM pin trong `iosApp`; task `:verifyInteropVersions` fail build nếu hai bên lệch. Chạy `./gradlew downloadInteropFrameworks` để tải XCFramework về `native-frameworks/` (gitignore).
