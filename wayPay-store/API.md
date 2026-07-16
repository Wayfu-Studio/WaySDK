# wayPay-store — Public API

Backend billing "tự viết" cho `:wayPay`: Google Play Billing Library trên Android, StoreKit 1 trên iOS. Chọn backend này khi app muốn tự vận hành billing, không mất phí bên thứ ba (so với `:wayPay-revenuecat` tính ~1% MTR). Toàn bộ hành vi mua/khôi phục đi theo contract của interface `AppPurchase` — xem `wayPay/API.md`; tài liệu này chỉ mô tả entry point và hành vi riêng của backend.

Phụ thuộc: `api(:wayPay)`, Billing Library (androidMain). Package: `com.waypay.store`.

## Khởi tạo

```kotlin
// Gọi 1 lần lúc app khởi động, TRƯỚC khi init ads / trước khi connect:
WayPayStore.install()
AppPurchaseManager.getInstance().connect(PurchaseConfig(...))
```

Android: yêu cầu `LifecycleProvider` (module `lifecycle_kmp`) đã được khởi tạo trước — backend lấy `Context` từ đó.

## Public API

### `expect object WayPayStore`

Entry point duy nhất của module. Implementation (`AppPurchaseImpl`, mapper) là `internal` — app không truy cập trực tiếp.

#### `fun install(): AppPurchase`

Tạo (nếu chưa có) backend store và đăng ký vào `AppPurchaseManager`. Idempotent — gọi lặp trả về instance cũ, không tạo `BillingClient` thứ hai (CAS thread-safe). Nếu một backend KHÁC (ví dụ RevenueCat) đã được install trước, backend đó thắng theo semantics first-wins của `AppPurchaseManager.install`.

**Trả về:** instance `AppPurchase` đang hoạt động.

**Throw:** Android — `IllegalStateException` nếu `LifecycleProvider` chưa init (không lấy được `Context`). iOS — không throw.

## Hành vi riêng của backend

- **Verify hook được tôn trọng đầy đủ**: `PurchaseConfig.verifier` trả `false`/throw → purchase KHÔNG được acknowledge/finish, caller nhận `PurchaseError.VerifyFailed`. Hệ quả: Android — Google tự refund sau ~3 ngày (ack window); iOS SK1 — không auto-refund, transaction chưa finish sẽ được `SKPaymentQueue` replay ở lần khởi động sau cho tới khi verify thành công và finish.
- **Finalize tự động**: consumable → `consume` (Android) / `finish` (iOS); non-consumable & subscription → `acknowledge` / `finish`.
- **`purchaseEvents` phát 3 source**: `USER_INITIATED`, `RESTORE`, `EXTERNAL` (pending→approved, mua thiết bị khác, Family Sharing, và renewal trên iOS). `Source.RENEWAL` hiện KHÔNG được phát; renewal Android chỉ phản ánh qua `activeEntitlements` sau lần query kế tiếp.
- **`PurchaseRecord` đầy đủ field verify**: Android có `purchaseToken`/`signature`/`originalJson`; iOS `purchaseToken` là **app receipt base64** (StoreKit 1) — backend verify qua App Store receipt validation / App Store Server API, không phải decode JWS.

## Lưu ý platform

### Android
- Dùng Google Play Billing Library (version theo `libs.versions.toml`, hiện 8.x).
- `connect()` suspend đến khi `BillingClient` connect xong; `isReady` = `BillingClient.isReady`.
- `openSubscriptionSettings()` mở deep link `https://play.google.com/store/account/subscriptions`.

### iOS
- Dùng **StoreKit 1** (SK2 là Swift-only, không có Obj-C bridge nên Kotlin/Native cinterop không truy cập được). Hệ quả: các tính năng SK2-only (offer codes API, `Transaction.currentEntitlements` streaming...) không có; renewal/external phát hiện qua `SKPaymentQueue` observer.
- `connect()` return ngay (StoreKit không có khái niệm connect); `isReady` = `true` sau đó.
- **Bắt buộc có nút Restore** gọi `restorePurchases()` (App Review Guideline 3.1.1).
- `openSubscriptionSettings()` mở trang quản lý subscription của App Store.

## Khi nào chọn backend này

- App doanh thu lớn — tránh phí % MTR của RevenueCat.
- App cần toàn quyền với receipt (Android: `purchaseToken`; iOS: app receipt base64) để verify trên backend riêng.
- Chấp nhận: tự maintain edge case billing, không có Offerings/paywall/experiments, iOS kẹt StoreKit 1.

## Log TAG

Module log với TAG `WayPay/Store` (cả Android lẫn iOS). Toàn bộ quy ước TAG của WaySDK: xem `wayCore/API.md`.
