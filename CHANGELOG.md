# Changelog

Nhật ký thay đổi của WaySDK. Tất cả module publish chung một số version (`way-sdk-version` trong
`gradle.properties`) nên changelog cũng dùng chung một dòng thời gian, chia theo module bên trong mỗi mục.

Quan hệ với tài liệu API: mỗi module có `API.md` **mô tả trạng thái hiện tại** — public model, API,
function, class kèm tham số, mục tiêu và behaviour. File này thì ngược lại — chỉ ghi **sự thay đổi so với
phiên bản trước**: cái gì thêm, đổi, bỏ, và người dùng SDK phải làm gì để nâng cấp. Đọc `API.md` để biết SDK
hiện làm gì; đọc changelog để biết bản mới khác bản cũ chỗ nào.

Quy ước mục con: `Thêm` / `Đổi` / `Bỏ` / `Sửa lỗi` / `Cần làm khi nâng cấp` (chỉ ghi mục nào thực sự có).
Thay đổi phá vỡ tương thích đánh dấu **BREAKING**.

Changelog bắt đầu được ghi từ mục dưới đây; các thay đổi trước `2.3.0-alpha01` chỉ có trong lịch sử git.

---

## Chưa phát hành

Đồng bộ toàn bộ ad SDK giữa Android và iOS, và đưa version SDK iOS về một nguồn khai báo duy nhất.
**Không có thay đổi nào trong public API Kotlin** — app đang dùng SDK không phải sửa code, chỉ cần
re-resolve package phía Xcode.

### Đổi — version ad SDK

Trước bản này iOS chậm hơn Android một major của Google Mobile Ads (Android nhánh 25.x ↔ iOS nhánh 13.x,
Google phát hành theo cặp) và lệch 3 minor của Adjust. Sau bản này hai nền tảng đi cùng nhau:

| SDK | Android | iOS |
|---|---|---|
| Google Mobile Ads | `25.3.0` → **`25.4.0`** | `12.12.0` → **`13.7.0`** |
| User Messaging Platform (UMP) | `4.0.0` (không đổi) | `3.0.0` → **`3.1.0`** |
| AppLovin MAX | `13.6.2` → **`13.6.4`** | `13.6.3` → **`13.6.4`** |
| Adjust | `5.7.0` → **`5.8.0`** | `5.4.6` → **`5.8.0`** |

Ghi chú AdMob iOS 12 → 13 (major, [migration guide](https://developers.google.com/admob/ios/migration)):

- Deployment target tối thiểu lên iOS 13 — `iosApp` đang ở 15.6 nên không ảnh hưởng.
- Tên class Objective-C **không đổi** ở v13 (đợt đổi tên chỉ ảnh hưởng Swift, diễn ra ở v12), nên cinterop
  bridge của `:wayAd-admob` giữ nguyên: toàn bộ symbol đang dùng (`GADMobileAds`, `GADBannerView`,
  `GADInterstitialAd`, `GADRewardedAd`, `GADRewardedInterstitialAd`, `GADNativeAd`, `GADAppOpenAd`,
  `GADAdValue`, `GADFullScreenPresentingAdProtocol`, các hằng `GADAdSize*`) vẫn có trong header 13.7.0.
- `GADCurrentOrientationAnchoredAdaptiveBannerAdSizeWithWidth` bị **deprecate**, thay bằng
  `GADLargeAnchoredAdaptiveBannerAdSizeWithWidth` (kích thước banner mới, cho phép serve video demand).
  `:wayAd-admob` vẫn dùng API cũ — còn chạy, xếp vào việc cần làm sau.
- `neighboringContentURLStrings` từ v13 raise exception nếu phần tử không phải `String`. WaySDK không dùng.

Ghi chú Adjust: bản 5.8.0 là thay đổi cộng thêm so với 5.4.6/5.7.0 (Samsung MAPS, `disableDeviceIdsReading`,
`thirdPartySharingSettingsWithTimeout`, signature library 5.0.0), không bỏ API nào WaySDK đang gọi. Riêng
iOS 5.8.0 đã có `adidWithTimeout:completionHandler:` đối xứng với `Adjust.getAdidWithTimeout` bên Android —
`:wayPay-revenuecat` chưa chuyển sang dùng, xem lưu ý trong `wayPay-revenuecat/API.md`.

### Đổi — mediation adapter Android

Nguyên tắc mới, ghi thẳng trong `gradle/libs.versions.toml`: adapter AdMob (`com.google.ads.mediation:*`)
và adapter MAX (`com.applovin.mediation:*-adapter`) của **cùng một network phải bọc cùng version SDK
network**. Hai module `:wayAd-admob` và `:wayAd-applovin` có thể cùng nằm trong một app, khi đó Gradle gom
SDK bên dưới về version cao nhất — nếu hai adapter bọc hai version khác nhau thì một trong hai chạy với SDK
mà nó không được test cùng. Trước bản này ironSource (9.4.2 vs 9.4.3), Unity (4.18.0 vs 4.18.1) và AppLovin
đang lệch đúng kiểu đó.

| Network | AdMob adapter | MAX adapter | SDK network sau khi resolve |
|---|---|---|---|
| AppLovin | `13.6.2.0` → **`13.6.4.0`** | — (dùng `applovin-sdk` trực tiếp) | `applovin-sdk` 13.6.4 |
| Google | — | `25.3.0.0` → **`25.4.0.0`** | `play-services-ads` 25.4.0 |
| Chartboost | `9.12.0.0` → **`9.13.0.0`** | `9.12.1.0` → **`9.13.0.0`** | `chartboost-sdk` 9.13.0 |
| Facebook | `6.21.0.3` → **`6.22.0.0`** | `6.21.0.0` → **`6.22.0.0`** | `audience-network-sdk` 6.22.0 |
| Fyber | `8.4.5.0` → **`8.4.7.0`** | `8.4.5.0` → **`8.4.7.0`** | `marketplace-sdk` 8.4.7 |
| InMobi | `11.3.0.0` → **`11.4.0.0`** | `11.3.0.0` → **`11.4.0.0`** | `inmobi-ads-kotlin` 11.4.0 |
| ironSource | `9.4.2.0` → **`9.5.0.0`** | `9.4.3.0.0` → **`9.5.0.0.0`** | `mediation-sdk` 9.5.0 |
| LINE | `3.1.0.0` → **`3.1.1.1`** | `3000.1.1.0` (không đổi) | `fivead` 3.1.1 |
| Mintegral | `17.1.61.0` → **`17.1.71.0`** | `17.1.61.0` → **`17.1.71.0`** | `mbridge_android_sdk` 17.1.71 |
| Pangle | `8.0.0.5.0` → **`8.2.0.4.0`** | `8.1.0.3.0` → **`8.2.0.4.0`** | `pag-sdk` 8.2.0.4 |
| Unity | `4.18.0.0` → **`4.19.0.1`** | `4.18.1.0` → **`4.19.0.1`** | `unity-ads` 4.18.1 → **4.19.0** |
| Vungle | `7.7.4.0` → **`7.7.7.0`** | `7.7.4.0` → **`7.7.7.0`** | `vungle-ads` 7.7.7 |
| i-mobile | `2.3.2.3` → **`2.3.2.4`** | — (MAX không có adapter) | `adnw-sdk-android` 2.3.2 |
| maio | `2.0.8.2` → **`2.0.9.0`** | — (MAX không có adapter) | `android-sdk-v2` 2.0.9 |
| myTarget | `5.47.1.0` (**giữ**) | `5.47.1.0` (**giữ**) | `mytarget-sdk` 5.47.1 |

myTarget cố tình không lên bản mới nhất: AdMob đã có adapter `5.51.2.0` nhưng adapter MAX mới nhất vẫn bọc
myTarget 5.47.1, bump một phía sẽ kéo `mytarget-sdk` 5.51.2 vào và làm adapter MAX chạy sai version.
Bỏ qua quy tắc này chỉ khi app chắc chắn chỉ dùng một trong hai stack mediation.

### Đổi — hạ tầng cinterop iOS

- Version SDK iOS chuyển vào `gradle/libs.versions.toml` (`iosGoogleMobileAds`, `iosUserMessagingPlatform`,
  `iosApplovinSdk`, `iosAdjustSdk`). Trước đây mỗi lần bump phải sửa số ở 2 nơi tách rời — bảng `interopSdks`
  trong root `build.gradle.kts` và chuỗi hardcode trong `build.gradle.kts` của từng module; `interopSdks` và
  SPM thì có `verifyInteropVersions` canh, còn chuỗi hardcode ở module thì không ai canh, sai là fail ở bước
  cinterop với lỗi khó đọc. Giờ cả ba nơi đọc chung một hằng.
- `:downloadInterop*` **BREAKING (chỉ với người build SDK từ source)**: task giờ bày `.xcframework` ra layout
  cố định `native-frameworks/<name>-<version>/<name>.xcframework` thay vì giải nén nguyên archive. Mỗi nhà
  cung cấp đóng gói một kiểu (Adjust bọc trong `AdjustSdk-iOS-tvOS-Dynamic-xcframework/`, AppLovin trong
  `applovin-ios-sdk-<version>/`, Google để ở gốc) — chuẩn hoá ở một chỗ để module không phải biết cấu trúc
  archive. Task báo lỗi rõ ràng nếu không tìm thấy `.xcframework` trong archive.
- Nguồn tải GoogleMobileAds và UserMessagingPlatform đổi từ bundle CocoaPods (`dl.google.com/dl/cpdc/...`)
  sang artifact SPM (`dl.google.com/googleadmobadssdk/...`) — đúng binary mà Xcode link qua SPM, và URL tra
  được từ `binaryTarget` trong `Package.swift` của đúng tag (bundle CocoaPods có hash không tra được theo
  version). URL của Adjust và AppLovin nội suy thẳng từ version.
- Thêm `check` lúc configure: URL tải phải chứa đúng chuỗi version, bắt trường hợp bump version trong catalog
  mà quên sửa URL — nếu không sẽ tải nhầm binary cũ vào thư mục mang tên version mới.
- Thư mục `native-frameworks/` (gitignore) nên xoá sạch một lần sau khi lấy bản này; các thư mục version cũ
  không còn được tham chiếu.

### Đổi — ad unit MAX trong app demo

Chỉ ảnh hưởng `composeApp`/`iosApp`, **không đụng tới module SDK nào**.

MAX cấp ad unit theo **từng app trên từng platform**, nhưng `MaxAdTestIds` trước đây là một `object` ở
`commonMain` dùng chung cho cả hai nền tảng — và bộ ID trong đó vốn thuộc app **Android** `com.test.sdk`.
Hệ quả: MAX trên Android fail 100% (`Ad Unit ID … is invalid or disabled`) vì `applicationId` là
`com.way_sdk`, còn iOS thì chạy nửa vời — server vẫn trả ad nhưng sai loại (creative MREC cho ad unit NATIVE,
`ALTaskRenderNativeAd: No oRtb response provided`) hoặc adapter timeout ở banner.

`MaxAdTestIds` giờ là `expect object` với `actual` riêng cho từng platform, kèm định danh app demo đổi cho
khớp entry trong MAX dashboard:

| | Trước | Sau |
|---|---|---|
| `composeApp` `applicationId` | `com.way_sdk` | `com.test.sdk` |
| `iosApp` `PRODUCT_BUNDLE_IDENTIFIER` | `com.way.sdk` | `food.cooking.sizzle.stack.card` |

Sau khi khớp, MAX load được trên cả hai nền tảng (banner, interstitial, rewarded).

Đổi định danh kéo theo Firebase: phải thêm client tương ứng vào project `waysdk-18380` —
`composeApp/google-services.json` cho Android (thiếu là **fail build** ngay ở plugin `google-services`) và
`iosApp/iosApp/GoogleService-Info.plist` cho iOS (thiếu thì vẫn build được, chỉ log lệch bundle). Cả hai file
đã được cập nhật.

`MaxAdTestIds.NATIVE` phía iOS vẫn là placeholder `FILL_ME_IOS_MAX_NATIVE_AD_UNIT_ID` — app iOS trong
dashboard chưa có ad unit native, nên màn native MAX trên iOS còn fail.

### Sửa lỗi — dữ liệu gửi Adjust

Áp cho cả bốn `AdjustExt.kt` (admob/applovin × android/ios). Lỗi có sẵn từ trước, không do đợt nâng version.

- **Field rỗng làm Adjust bỏ luôn parameter.** Mọi callback/partner parameter trước đây dùng `?: ""`, chỉ
  chặn được `null` chứ không chặn chuỗi rỗng. Khi gặp giá trị rỗng, Adjust log `Callback parameter value is
  empty` rồi **loại bỏ parameter đó** khỏi event. Quan sát thực tế: AdMob native trên iOS trả `adSourceName`
  rỗng nên event `ad_impression` của native mất hẳn field `adNetwork`, trong khi banner/interstitial/rewarded
  vẫn có. Giờ parameter chỉ được gửi khi giá trị không rỗng.

  Không thêm giá trị thay thế: khi `adSourceName` rỗng thì field `adNetwork` **vắng mặt** thay vì điền
  `adSourceID` hay tên class adapter. Sự vắng mặt mang thông tin thật, còn điền giá trị từ nguồn khác sẽ làm
  field mất đi ngữ nghĩa "tên ad source". Nguồn vẫn truy được qua `adSourceId` đi kèm.

- **BREAKING (dữ liệu): thống nhất field doanh thu của event `ad_impression` thành `revenue`, số tiền thật,
  format cố định 6 chữ số thập phân.** Trước đó tồn tại ba cách biểu diễn khác nhau cho cùng một đại lượng:

  | | Trước | Sau |
  |---|---|---|
  | AdMob Android | `revenue_micros` = `"500"` | `revenue` = `"0.000500"` |
  | AdMob iOS | `revenue_micros` = `"5.0E-4"` | `revenue` = `"0.000500"` |
  | MAX (cả 2 platform) | `revenue` = `"5.0E-4"` | `revenue` = `"0.000500"` |

  Hai vấn đề được xử lý cùng lúc. Thứ nhất, **đơn vị lệch nhau**: iOS đọc `GADAdValue.value` (đơn vị tiền tệ)
  còn Android đọc `AdValue.valueMicros` (micros), nhưng gửi dưới cùng một tên field. Thứ hai, **`Double.toString()`
  sinh ký hiệu khoa học** với mọi giá trị nhỏ hơn 0.001 — đúng khoảng doanh thu của một impression: `0.0005`
  thành `"5.0E-4"`, `0.000001` thành `"1.0E-6"`. Nhánh MAX ở cả hai nền tảng đang dính lỗi này, chỉ chưa lộ vì
  test ad luôn trả `0.0`.

  `formatAdRevenue()` (internal, mỗi module một bản) dựng chuỗi từ số nguyên micros thay vì qua `toString()`,
  nên không phụ thuộc locale và không bao giờ ra ký hiệu khoa học.

  **Số liệu trước và sau mốc deploy khác nhau về cả tên field lẫn định dạng** — phải báo team analytics.

  *Đã cân nhắc và loại bỏ:* dùng thẳng `event.setRevenue(amount, currency)` — đây là API doanh thu chuẩn của
  Adjust cho event và sẽ xoá sạch vấn đề định dạng. Nhưng mỗi impression đã được báo cáo qua
  `Adjust.trackAdRevenue()` rồi, set thêm revenue lên event nữa là Adjust nhận **cùng một khoản tiền hai lần**.
  Vì vậy doanh thu chỉ đi qua đúng một đường là `ad_revenue`, còn trong event nó chỉ là dữ liệu tham chiếu
  dạng callback parameter.

  Ngoài ra khi `adValue` là `null`, iOS trước đây gửi chuỗi literal `"null"`; giờ bỏ hẳn field.

- **`ad_revenue` không còn set giá trị rỗng.** `setRevenue`/`setAdRevenueNetwork`/`setAdRevenueUnit`/
  `setAdRevenuePlacement` chỉ được gọi khi có giá trị. Riêng phần số tiền của `ad_revenue` vốn đã đúng ở cả
  hai nền tảng (iOS truyền `value`, Android chia `valueMicros / 1_000_000`, khớp hướng dẫn tích hợp của Adjust
  cho từng platform) — thay đổi này không đụng tới nó.

### Đổi — tài liệu

- `wayAd-admob/API.md`, `wayAd-applovin/API.md`: version SPM cần pin ở Xcode giờ trỏ về
  `gradle/libs.versions.toml` thay vì `interopSdks`; thêm ghi chú quy tắc pin adapter mediation.
- `adjust_kmp/API.md`: cập nhật version hai platform (đã hết lệch), mô tả lại layout `native-frameworks`.
- `wayPay-revenuecat/API.md`: ghi rõ `adidWithTimeout` đã có ở Adjust iOS 5.8.0 nhưng WaySDK chưa dùng.
- Thêm file này.

### Cần làm khi nâng cấp

1. **Xcode**: không cần resolve SPM bằng tay. `project.pbxproj` và `Package.resolved` đã cập nhật sẵn
   (GoogleMobileAds 13.7.0, UMP 3.1.0, AppLovin 13.6.4, Adjust 5.8.0, kéo theo `adjust_signature_sdk` 5.0.0)
   và đã được `xcodebuild -resolvePackageDependencies` xác nhận là đúng graph.

   Nhưng **build lần đầu phải Clean Build Folder** (⇧⌘K, hoặc xoá `~/Library/Developer/Xcode/DerivedData/iosApp-*`).
   DerivedData cũ còn giữ package graph của Adjust 5.4.6 — vốn phụ thuộc `adjust_signature_sdk` 3.61.0 thay vì
   5.0.0 — nên build sẽ fail bằng lỗi khó đoán:

   ```
   error: Could not compute dependency graph: failed to find blueprint corresponding to
   PIF GUID: SWBTargetGUID(rawValue: "PACKAGE-PRODUCT:AdjustSignature")
   ```

   Đây là cache cũ, không phải sai cấu hình: với DerivedData sạch thì `iosApp` build thành công.
2. **App tích hợp SDK**: pin SPM đúng bốn version trên, nếu không `verifyInteropVersions` sẽ fail build với
   thông báo chỉ rõ bên nào lệch.
3. **Android**: không phải làm gì. Nếu app tự khai báo adapter mediation riêng ngoài WaySDK thì rà lại theo
   bảng ở trên để không kéo hai version của cùng một SDK network.

### Việc còn lại (không nằm trong bản này)

- **Ba lỗi số liệu ad đã xác định nhưng chưa sửa** (đều có từ trước, không do đợt nâng version này):
  - *AdMob banner iOS đếm hai impression cho một lần hiển thị.* GMA iOS trả về **hai ad response khác nhau**
    (`responseIdentifier` khác nhau, cách ~0.4 giây) rồi ghi hai impression và hai `ad_revenue`. Android
    không dính. Đã loại trừ ba nguyên nhân: callback lặp cùng creative, gán lại `rootViewController`, và
    preload khi view chưa vào hierarchy — bỏ hẳn preload vẫn còn lỗi. Ứng viên còn lại: banner bị đổi kích
    thước lúc Compose layout làm GMA request lại. Bước tiếp theo nên là **thêm log** ở
    `bannerViewDidReceiveAd` (hiện chỉ log lần đầu nên lần nhận ad thứ hai vô hình) kèm `adSize`/`frame`,
    chứ không sửa tiếp khi chưa biết nguyên nhân.
  - *MAX banner ghi impression và ad revenue lúc preload, trước khi hiển thị 3–11 giây*, còn lần hiển thị
    thật thì không được đếm. Nguyên nhân: `MAAdView` render ngay khi load dù chưa vào view hierarchy, và
    WaySDK lấy impression từ `didPayRevenueForAd` (không dùng `didDisplayAd` được vì AppLovin không gọi
    callback đó cho banner/MREC). Đã thử `stopAutoRefresh()` trước `loadAd()` — **không** dùng được: Android
    không đổi gì, iOS thì banner không load nổi (`ALAppLovinMediationAdapter is timing out`, `code=-5101`
    sau đúng 10.01s) vì creative không bao giờ render.
  - *MAX SDK auto-refresh chạy song song với vòng reload của `BannerAdState`*, làm mỗi chu kỳ reload ghi hai
    impression. `BannerAdState.startAutoRefreshCountdown` đã tự lo reload sau mỗi impression, nên hai cơ chế
    chồng nhau.
- Chuyển banner adaptive của `:wayAd-admob` sang API `GADLargeAnchoredAdaptiveBannerAdSizeWithWidth` của
  AdMob v13.
- Chuyển `:wayPay-revenuecat` (iOS) sang `Adjust.adidWithTimeout` để bỏ phần callback đọng.
- myTarget đang giữ lại ở 5.47.1 chờ adapter MAX bắt kịp AdMob.
- Ad unit **native** cho app iOS trong MAX dashboard.
- Mediation adapter MAX cho iOS: hiện `iosApp` chỉ link `AppLovinSDK`, không có adapter bên thứ ba nào (Android
  có 12). Đã thử thêm 11 adapter qua SPM rồi gỡ ra: log waterfall cho thấy mọi ad unit đang test chỉ có **một**
  entry và là `APPLOVIN_NETWORK` (`Loading ad 1 of 1 from APPLOVIN_NETWORK`), nên không adapter nào được gọi
  tới. Chỉ đánh giá lại được khi app iOS có ad unit thật với waterfall nhiều network. Riêng adapter Meta hiện
  **không dùng được qua SPM**: manifest khai `platforms: [.iOS(.v13)]` trong khi phụ thuộc `FBAudienceNetwork`
  yêu cầu iOS 15.0, SwiftPM từ chối resolve — lỗi nằm trong package của AppLovin.
