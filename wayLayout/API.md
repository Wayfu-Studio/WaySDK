# wayLayout — Public API

`wayLayout` là module **thư viện resource thuần Android** (Android Library, namespace `com.waylayout`, artifact `way-sdk:way-layout`) của WaySDK. Module **không chứa bất kỳ file Kotlin/Java nào trong `src`** — toàn bộ public API là **Android resources**: layout XML cho quảng cáo native/shimmer placeholder, drawable, styleable attributes, style và color. Đây KHÔNG phải module Compose Multiplatform; các layout dùng hệ View XML truyền thống.

Vị trí trong WaySDK: `wayLayout` là "kho giao diện" cho module `:wayAd` — `wayAd` phụ thuộc qua `api(project(":wayLayout"))`, dùng các layout shimmer làm placeholder khi tải banner (`AdmobBannerAdView`, `AppLovinBannerAdView`), dùng bộ ID `ad_*` làm hợp đồng bind native ad (`AdmobNativeAdView`), và đọc styleable `FloatingNativeAdView` trong view floating ad. Phụ thuộc bên ngoài (theo `build.gradle.kts`): `androidx.core:core-ktx`, `androidx.appcompat`, `com.google.android.material`, **Google Play Services Ads** (cung cấp `NativeAdView`/`MediaView` dùng trong layout) và **Facebook Shimmer** (cung cấp `ShimmerFrameLayout`). `minSdk = 26`, `compileSdk = 36`, publish lên GitHub Packages (`https://maven.pkg.github.com/hidenobi/WaySDK`).

## Public API

### `R.layout.shimmer_banner_small`
Placeholder skeleton (hiệu ứng shimmer) cho banner **nhỏ** trong lúc quảng cáo đang tải. Root là `com.facebook.shimmer.ShimmerFrameLayout` (id `@+id/shimmer_container_banner`, `shimmer_auto_start="true"`, base color `#ccc`, nền `@drawable/bg_card_ads`). Bên trong là khung xám mô phỏng một native banner cao **56dp**: icon vuông 50dp bên trái, 2 dòng text giữa, 1 nút bên phải, kèm nhãn "Ad" (style `AppTheme.Ads`) ở góc trên-trái.

Nơi dùng thực tế trong `wayAd`: `AdmobBannerAdView` và `AppLovinBannerAdView` inflate layout này khi chiều cao banner **< 250** (dp).

### `R.layout.shimmer_banner_large`
Placeholder skeleton shimmer cho banner **lớn**, cùng cấu trúc root `ShimmerFrameLayout` (id `shimmer_container_banner`, auto-start, nền `bg_card_ads`) nhưng cao cố định **250dp**: nhãn "Ad" trên cùng, vùng ảnh lớn 160dp ở giữa, hàng dưới gồm icon 50dp + 2 dòng text + nút — mô phỏng một native/banner medium-rectangle.

Nơi dùng thực tế: `AdmobBannerAdView` / `AppLovinBannerAdView` chọn layout này khi chiều cao banner **>= 250** (dp).

### `R.layout.layout_native_ad_small`
Template native ad dạng ngang, full-width, nền trắng. Root là `com.google.android.gms.ads.nativead.NativeAdView`. Cấu trúc: cột trái gồm icon app 36dp (`ad_app_icon`) + nhãn "Ad" nền vàng (`ad_attribution`) + `MediaView` 72dp (`ad_media`); cột giữa gồm headline tối đa 2 dòng (`ad_headline`), body 2 dòng (`ad_body`), `RatingBar` 5 sao bước 0.5 (`ad_stars`), store/price (`ad_store`, `ad_price`), advertiser (`ad_advertiser`); ngoài cùng bên phải là nút CTA cao 40dp (`ad_call_to_action`).

Layout này định nghĩa **hợp đồng ID `ad_*`** mà `AdmobNativeAdView` (wayAd) dùng `findViewById` để bind dữ liệu `NativeAd` vào view.

### `R.layout.layout_native_ad_floating`
Template native ad **nổi (floating)** kích thước cố định **140dp x 140dp**, bo góc (nền `bg_floating_native_container`, `clipToOutline="true"`). Root là `NativeAdView`. Gồm: `MediaView` phủ toàn bộ (`ad_media`); một `TextView` headline **ẩn** (`ad_headline`, `visibility="invisible"`, kích thước 0dp — tồn tại chỉ để thoả yêu cầu đăng ký headline view của SDK Ads); dải gradient đen mờ ở đáy (`bg_floating_native_bottom_gradient`) chứa nút CTA cam bo góc (`ad_call_to_action`); và badge "Ad" cam nhỏ ở góc trên-trái.

Trong `wayAd`, `FloatingNativeAdTemplate` dựng lại cấu trúc tương đương bằng code và tái sử dụng chính các ID/drawable của layout này (`R.id.ad_media`, `R.id.ad_headline`, `R.id.ad_call_to_action`, `R.drawable.bg_floating_native_bottom_gradient`).

### Styleable `FloatingNativeAdView` (prefix `fnav_`)
Bộ XML attributes cấu hình cho custom view `com.wayad.view.floating.FloatingNativeAdView` (khai báo attr ở module này, view đọc attr nằm ở `wayAd`). Hành vi mô tả dưới đây lấy từ code đọc attr thực tế trong `FloatingNativeAdView.kt`:

| Attr | Kiểu (format) | Mô tả (default khi không khai báo) |
|---|---|---|
| `fnav_size` | `dimension` | Đặt cả width và height của view thành cùng một giá trị (view vuông). Chỉ áp dụng nếu > 0; default -1 (giữ layout params sẵn có). |
| `fnav_cornerRadius` | `dimension` | Nếu >= 0: gán background `GradientDrawable` trong suốt bo góc + `clipToOutline = true`. Default -1 (không bo). |
| `fnav_edgePadding` | `dimension` | Margin 4 phía áp vào `FrameLayout.LayoutParams` khi căn vị trí ban đầu (chỉ áp khi > 0 và có `fnav_initialAlignment`). Default 0. |
| `fnav_reloadAfterClose` | `integer` | Số **millisecond** chờ trước khi ad tự hiện lại sau khi user bấm nút đóng. 0 (default) = đóng hẳn (`visibility = GONE`); > 0 = ẩn (alpha 0, không nhận click) rồi tự khôi phục sau khoảng thời gian này. |
| `fnav_enableDrag` | `boolean` | Cho phép kéo-thả view (intercept touch, dịch chuyển bằng translation, vượt `touchSlop` mới tính là drag). Default `true`. |
| `fnav_enableClose` | `boolean` | Hiện nút đóng "×" 24dp góc trên-phải (nền `bg_floating_native_close`). Default `true`. |
| `fnav_enableAnimation` | `boolean` | Bật animation scale/alpha khi hiện (280ms, `OvershootInterpolator`) và khi đóng/ẩn (180ms). Default `true`. |
| `fnav_initialAlignment` | `enum` | Vị trí neo ban đầu trong `FrameLayout` cha, áp dụng một lần khi attach vào window. Giá trị: `none`(0, default — không đổi gravity), `topStart`(1), `topEnd`(2), `bottomStart`(3), `bottomEnd`(4), `center`(5), `centerStart`(6), `centerEnd`(7), `topCenter`(8), `bottomCenter`(9). |

### Styleable `FloatingTemplateNativeAdView` (prefix `ftnav_`)
Bộ attribute dành cho một template floating native ad có thể tuỳ biến màu sắc: `ftnav_size` (dimension), `ftnav_cornerRadius` (dimension), `ftnav_backgroundColor` (color), `ftnav_ctaColor` (color), `ftnav_ctaTextColor` (color), `ftnav_showAdBadge` (boolean), `ftnav_showBottomGradient` (boolean).

**Lưu ý:** tại thời điểm hiện tại, styleable này **được khai báo nhưng chưa có view nào trong repo đọc nó** (không có `R.styleable.FloatingTemplateNativeAdView` nào được tham chiếu trong Kotlin) — coi như API dự phòng, ngữ nghĩa chi tiết chưa được cài đặt.

### Style `AppTheme.Ads`
Style (parent `Theme.AppCompat.Light`) cho **nhãn "Ad"** dạng chip: text mặc định `"Ad"`, chữ trắng 11sp, nền `@drawable/ads_icon` (xanh `colorAds`, bo góc bất đối xứng), `wrap_content`, min 15px. Được cả hai layout shimmer dùng cho nhãn góc trên-trái.

### Hợp đồng View ID (`R.id.*`)
Các ID public mà consumer (wayAd) tra cứu bằng `findViewById` / gán khi dựng view bằng code:

| ID | View trong layout | Vai trò |
|---|---|---|
| `ad_media` | `MediaView` | Vùng media (ảnh/video) của native ad |
| `ad_headline` | `TextView` | Headline (ẩn trong template floating) |
| `ad_body` | `TextView` | Body text |
| `ad_call_to_action` | `TextView`/`Button` | Nút CTA |
| `ad_app_icon` | `ImageView` | Icon app |
| `ad_attribution` | `TextView` | Nhãn "Ad" |
| `ad_stars` | `RatingBar` | Đánh giá sao (5 sao, bước 0.5) |
| `ad_store` | `TextView` | Tên store |
| `ad_price` | `TextView` | Giá |
| `ad_advertiser` | `TextView` | Tên nhà quảng cáo |
| `shimmer_container_banner` | `ShimmerFrameLayout` | Container shimmer placeholder |

## Public models

Module **không có model/class Kotlin nào** (không có source code, chỉ có resources). Bảng dưới liệt kê các resource giá trị/drawable public còn lại:

### Colors (`res/values/colors.xml`)
| Resource | Giá trị | Ý nghĩa |
|---|---|---|
| `lightTransparent` | `#16000000` | Xám đen trong suốt nhẹ — nền các khối skeleton trong layout shimmer |
| `colorAds` | `#068abf` | Xanh dương — nền drawable `ads_icon` (nhãn "Ad") |
| `colorWhite` | `#ffffffff` | Trắng — nền nội dung shimmer banner nhỏ |

### Drawables
| Resource | Hình dạng | Ý nghĩa |
|---|---|---|
| `bg_card_ads` | rectangle, bo góc 10dp, nền trắng, viền 1dp `#5E58595A` | Nền dạng card cho 2 layout shimmer |
| `ads_icon` | shape nền `colorAds`, bo góc bất đối xứng (topLeft 10dp, bottomRight 13dp…) | Nền nhãn "Ad" của style `AppTheme.Ads` |
| `bg_floating_native_container` | rectangle bo góc 12dp, nền đen | Nền container của floating native ad |
| `bg_floating_native_cta` | rectangle bo góc 8dp, nền cam `#FFFF6A00` | Nền nút CTA floating |
| `bg_floating_native_ad_badge` | rectangle bo góc 4dp, nền cam `#FFFF6A00` | Nền badge "Ad" floating |
| `bg_floating_native_bottom_gradient` | gradient dọc từ `#8C000000` (đáy) → trong suốt | Lớp phủ gradient dưới đáy floating ad, làm nổi CTA |
| `bg_floating_native_close` | oval, nền `#8C000000` | Nền tròn của nút đóng "×" (do `FloatingNativeAdView` ở wayAd dựng bằng code) |

## Lưu ý platform

- **Android-only.** Khác với nhiều module khác của WaySDK (KMP/Compose Multiplatform), `wayLayout` là plain Android library dùng plugin `androidLibrary` + `kotlin.android`, không có source set iOS và **không có Composable nào** — UI hoàn toàn là XML View. Trên iOS, `wayAd` không dùng module này.
- Các layout native ad yêu cầu **Google Play Services Ads** ở runtime (root view là `com.google.android.gms.ads.nativead.NativeAdView`); layout shimmer yêu cầu **Facebook Shimmer**. Cả hai được khai báo `implementation` trong module nên consumer qua `:wayAd` (vốn expose `wayLayout` bằng `api`) cần đảm bảo hai thư viện này có mặt trên classpath runtime.
- `minSdk = 26`; consumer phải có minSdk >= 26.
