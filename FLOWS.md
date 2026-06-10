# Flows

End-to-end sequences across the three repos. Code locations cited so you can trace each step.

---

## 1. Hotel Staff Sign-In (B2B)

```
User → web-b2b-hotel /auth/login
     ↓ POST /api/v1/auth/login   { email, password, recaptchaToken }
     ↓ (AuthController → StaffUserService)
     │   - RecaptchaService.verify()
     │   - BCrypt.Verify(password)
     │   - issue access JWT (1h) + refresh JWT (7d)
     │   - persist RefreshToken row
     ← 200 { accessToken, refreshToken, user: { id, role, hotelId, branchId } }
     ↓ AuthStore.setTokens() → localStorage
     ↓ APP_INITIALIZER → /api/v1/auth/me → UserInfoModel
     ↓ Router → /dashboard
```

**On 401:** `httpErrorInterceptorFactoryFunc` (from `@takumo/core`) calls `/api/v1/auth/refresh-token` with the refresh token, replays original request. If refresh fails → `AuthStore.logout()` → `/auth/login`.

Files: [AuthController.cs](../takumo-api/src/Takumo.B2BHotel.API/Controllers/AuthController.cs), [StaffUserService](../takumo-api/src/Takumo.B2BHotel.Application/Services/StaffUserService/), [RecaptchaService.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/ExternalServices/RecaptchaService.cs), [AuthStore](../takumo-ui/projects/web-b2b-hotel/src/app/stores/), `bearerTokenAuthHeaderInterceptorFactoryFunc` in `@takumo/core`.

---

## 2. Traveler Sign-Up & Email Verification (B2C)

```
User → web-b2c-traveler /auth/register
     ↓ POST /api/v1/traveler/auth/register
     │   (TravelerAuthController → TravelerService)
     │   - create TravelerUser (status = PendingVerification)
     │   - create VerificationToken (type = EmailVerification)
     │   - SendGridEmailService renders RazorLight template, sends link
     ← 202 Accepted
     
User → clicks email link → web-b2c-traveler /auth/verify-email?token=…
     ↓ POST /api/v1/traveler/auth/verify-email  { token }
     │   - VerificationTokenService.consume()
     │   - mark TravelerUser.Status = Active
     ← 200 OK
```

Files: [TravelerAuthController.cs](../takumo-api/src/Takumo.B2BHotel.API/Controllers/Traveler/TravelerAuthController.cs), [VerificationTokenService](../takumo-api/src/Takumo.B2BHotel.Application/Services/VerificationTokenService/), [SendGridEmailService.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/ExternalServices/SendGridEmailService.cs), [Templates/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Templates/).

---

## 3. Hotel Staff Creates a Luggage Booking

This is the core revenue-bearing flow.

```
Staff → /bookings/new (web-b2b-hotel)
       1. Search guest via GuestInfoController → SearchGuestService
          - local DB lookup, then PMS lookup if integration enabled
          - PmsGuestSearchStrategyFactory picks Mews | Cloudbeds | Oracle
       2. Fill pickup info (date, hotel branch pickup time zone)
       3. Fill destination (another hotel branch, address, or airport counter)
       4. Add luggage items (LuggageInfo: size, type, special handling)
       5. Consent boxes: ProhibitedGoodsConsent, CarrierLiabilityConsent
       6. Submit
       
     ↓ POST /api/v1/bookings   (BookingController)
     ↓ BookingService.CreateAsync()
        - FluentValidation on request
        - assign BookingRef = "BK{yyyyMMdd}-{sequence}"
        - persist Booking (Status = Pending) + LuggageInfo[]
        - emit Notification (NotificationTrigger = BookingCreated)
            → FirebaseRealtimeDatabase.WriteAsync(/notifications/{hotelId}/...)
        - enqueue Hangfire job: ShipmentCreationJob
     ← 201 { bookingId, bookingRef, status: "Pending" }
     
Background (Hangfire):
     ShipmentCreationStrategyFactory selects carrier:
       - ShippingProvider.ShipAndCo  → ShipAndCoShipmentCreationStrategy
       - ShippingProvider.YamatoB2Cloud → YamatoB2CloudShipmentCreationStrategy
     POST to carrier API → returns ShipmentId, TrackingNumber, LabelUrl
     Booking.Status = Booked
     ShipmentLabelUploader → upload PDF label → AWS S3 (LabelsBucketName)
     Booking.LabelUrl set
     Notification emitted (BookingBooked)
     SendGridEmailService → confirmation email to guest
     
On carrier failure:
     ShipmentApiFailureLog row written
     Booking.Status = PendingRetry, then Failed after max attempts
     ShipmentRetryBackgroundJobService re-tries per schedule
```

Files: [BookingController.cs](../takumo-api/src/Takumo.B2BHotel.API/Controllers/BookingController.cs), [BookingService.cs](../takumo-api/src/Takumo.B2BHotel.Application/Services/BookingService/BookingService.cs), [Infrastructure/Strategies/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Strategies/), [BackgroundJobs/ShipmentRetryBackgroundJobService.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/BackgroundJobs/ShipmentRetryBackgroundJobService.cs), [FirebaseRealtimeDatabase.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/ExternalServices/FirebaseRealtimeDatabase.cs).

---

## 4. Shipment Tracking Lifecycle

The Booking status machine:

```
Pending ──► Booked ──► Collected ──► InTransit ──► Delivered
   │           ▲                                       │
   ▼           │                                       ▼
PendingRetry ──┘                                   Returned
   │
   ▼
Failed
                                       ╲
                                        ─► Cancelled
                                       ╱
                                       ─► DeliveryFailed
                                       ╲
                                        ─► Unknown
```

(See [BookingStatus.cs](../takumo-api/src/Takumo.B2BHotel.Domain/Enums/BookingStatus.cs))

**Yamato tracking poll** ([YamatoTrackingPollJob.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/BackgroundJobs/YamatoTrackingPollJob.cs)):

- Recurring Hangfire job — cron from `BackgroundJobSettings.TrackingShipmentCron` (default `*/10 * * * *`) and `YamatoTracking.PollCron` (`0 * * * *`)
- Picks up to `MaxBookingsPerTick` (500) bookings in tracked states
- Batches up to `MaxNumbersPerCall` (20) tracking numbers per request
- Throttles per-customer to `MinIntervalSecondsPerCustomer` (5.5s)
- Retries up to `MaxHttpRetries` (3); after `MaxConsecutiveFailuresBeforeDisable` (10) disables the tracker
- Updates Booking.Status + writes `BookingShipmentHistory` row + emits Notification

**ShipAndCo monitor** ([ShipAndCoMonitorService.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/ExternalServices/ShipAndCoMonitorService.cs)) runs analogously for ShipAndCo-shipped bookings.

---

## 5. Hotel Subscription / Billing (Stripe)

```
Admin → /admin/subscription/checkout (web-b2b-hotel admin section)
     ↓ POST /api/v1/checkout/session  { planId, billingPeriod }
     ↓ CheckoutController → CheckoutService
        - StripeService.CreateCheckoutSession()
        - returns Stripe-hosted URL
     ← 200 { url }
     
Browser → Stripe Checkout → success URL on B2B app

Stripe → webhook → POST /api/v1/stripe/webhook (StripeWebhookController)
     ↓ verify signature
     ↓ StripeEventHandlerFactory routes by event type:
        - checkout.session.completed → CheckoutSessionCompletedHandler
            • create HotelSubscription, set Status = Active
            • create Invoice
        - invoice.payment_succeeded → InvoicePaymentSucceededHandler
        - invoice.payment_failed   → InvoicePaymentFailedHandler
        - customer.subscription.trial_will_end → TrialWillEndHandler
        - customer.subscription.updated → SubscriptionUpdatedHandler
     ↓ persist StripeWebhookEvent (idempotency)
     ↓ enqueue notification email via SendGrid
     ← 200 OK
```

**Recurring reminders** ([SubscriptionBackgroundJobService.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/BackgroundJobs/SubscriptionBackgroundJobService.cs)) — fires at `TakumoSettings.SubscriptionExpiringReminderDays` (auto-renewal: `[3]` days before; non-auto-renewal: `[7, 1]` days before).

Files: [CheckoutController.cs](../takumo-api/src/Takumo.B2BHotel.API/Controllers/CheckoutController.cs), [StripeWebhookController.cs](../takumo-api/src/Takumo.B2BHotel.API/Controllers/StripeWebhookController.cs), [Webhooks/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Webhooks/), [StripeService.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/ExternalServices/StripeService.cs), [HotelSubscriptionService](../takumo-api/src/Takumo.B2BHotel.Application/Services/HotelSubscriptionService/).

---

## 6. Real-Time Notifications (Firebase)

The B2B portal needs live updates when a booking changes status, a shipment is collected, an invoice fails, etc.

```
API event (e.g. BookingService) 
   → NotificationService.Emit(trigger, entity, scope)
   → write Notification row (Postgres)
   → FirebaseNotificationSender writes to /notifications/{hotelId}/{branchId}/{notificationId}
   
web-b2b-hotel
   → AngularFire listens on /notifications/{hotelId}/{branchId}
   → updates in-memory store; bell-icon badge increments
   → on click: fetches /api/v1/notifications/{id} for full detail; PATCH to mark read
```

Notification triggers enumerated in [NotificationTrigger.cs](../takumo-api/src/Takumo.B2BHotel.Domain/Enums/NotificationTrigger.cs); object types in [NotificationObjectType.cs](../takumo-api/src/Takumo.B2BHotel.Domain/Enums/NotificationObjectType.cs).

User-status flattening (per migration `20251231042555_FlattenNotificationUserStatuses`) means read/unread is stored per-user-per-notification, not as a JSON blob.

---

## 7. Guest Pass (Apple / Google Wallet)

Travelers receive a digital pass for hotel pickup identification.

```
Traveler → /pass (web-b2c-traveler)
     ↓ POST /api/v1/traveler/passes   { hotelBranchId, ... }
     ↓ TravelerPassController → GuestPassService
        - create GuestPass row
        
For Apple Wallet:
     ↓ GET  /api/v1/apple-wallet/{passId}
        - AppleWalletServiceV2 builds .pkpass
        - signs with AppleWWDRCAG6.cer + team certificate
        - returns application/vnd.apple.pkpass
     ↓ user adds to Wallet → device registers with Apple Push (APNs)
     ↓ /api/v1/apple-wallet/devices/.../registrations/...  (AppleWalletWebServiceController)
        - persists ApplePassDevice + ApplePassRegistration

For Google Wallet:
     ↓ GET  /api/v1/google-wallet/{passId}/save-link
        - GoogleWalletServiceV2 mints signed JWT save-link
        - persist GooglePassRegistration
     ↓ user opens link → added to Google Wallet
        
On booking update:
     UpdateWalletPassBackgroundJobService picks up changed bookings
     → re-issues pass, pushes to APNs (Apple) / Google Wallet API
     → device pulls fresh pass
```

Settings: [AppleWalletSettings, GoogleWalletSettings in appsettings.json](../takumo-api/src/Takumo.B2BHotel.API/appsettings.json).

Files: [AppleWalletWebServiceController.cs](../takumo-api/src/Takumo.B2BHotel.API/Controllers/AppleWalletWebServiceController.cs), [ExternalServices/WalletPass/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/ExternalServices/WalletPass/), [UpdateWalletPassBackgroundJobService.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/BackgroundJobs/UpdateWalletPassBackgroundJobService.cs).

---

## 8. PMS Guest Search

Used inside the booking flow when staff needs to look up a guest the hotel's PMS already knows about.

```
GuestInfoController.SearchPmsGuests(query)
   → SearchGuestService
   → HotelPmsIntegration row provides PmsProvider + auth fields
   → PmsGuestSearchStrategyFactory:
       PmsType.Mews       → MewsGuestSearchStrategy
       PmsType.Cloudbeds  → CloudbedsGuestSearchStrategy
       PmsType.Oracle     → OracleGuestSearchStrategy
   → each strategy maps to a common GuestInfoDto
   ← list of guests with check-in/check-out dates
```

Per-provider auth fields are declared in [appsettings.json `PmsProviderSettings.Providers`](../takumo-api/src/Takumo.B2BHotel.API/appsettings.json) (Mews: ApiUrl + ClientToken + AccessToken; Cloudbeds: ApiUrl + PropertyIDs + ApiKey; Oracle: ApiUrl + ClientID + ClientSecret + EnterpriseID + ApplicationKey + Scope + HotelID).

Files: [HotelPmsIntegrationService](../takumo-api/src/Takumo.B2BHotel.Application/Services/HotelPmsIntegrationService/), [Strategies/*GuestSearchStrategy.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Strategies/).

---

## 9. Address Validation

Google Places + custom validation rules ensure pickup/destination addresses are deliverable.

```
Frontend → TkmGooglePlacesAutocomplete (from _widgets) emits placeId
        → AddressApi.validate(placeId)
        → POST /api/v1/addresses/validate
        → AddressController → AddressService → GoogleAddressService
            • Google Places Details API
            • Maps to internal Address VO
            • Applies business rules (Japan postal code format, etc.)
        ← validated address with normalized fields + warnings
```

Files: [AddressController.cs](../takumo-api/src/Takumo.B2BHotel.API/Controllers/AddressController.cs), [AddressService](../takumo-api/src/Takumo.B2BHotel.Application/Services/AddressService/), [GoogleAddressService.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/ExternalServices/GoogleAddressService.cs).

---

## 10. Marketing Site PMS Partnership Inquiry

```
Visitor → takumo-landing-page /contact (or /pms-partnerships)
       → fills custom form (inquiry_type, name_jp, name_kana, email, company_name, message, privacy_policy)
       → POST admin-post.php?action=pms_contact_form
       → inc/pms-contact-form-handler.php
           • nonce check
           • email-format validation
           • wp_mail() to admin_email
       → redirect with ?status=success|error|invalid_email
       → JS reads query param and renders banner
```

No API call to `takumo-api` from this flow — the inquiry stays in WordPress email/admin.

---

## 11. Language Switching (Both SPAs)

```
User clicks language selector in shell
   → LanguageSwitcherComponent emits languageChanged
   → HasLanguageSwitcherBaseDirective intercepts
   → window.location.href = /{newLocale}/{currentPath}
   → page reloads with the other locale's pre-built bundle
   → main.ts fetches /translations/{locale}.json
   → TranslateService.setTranslation() + .use()
```

URL is always source of truth for locale. Build outputs are fully separate per-locale (`dist/<app>/browser/en/`, `dist/<app>/browser/ja/`).

---

## 12. Deployments

| Repo | Trigger | Target |
|---|---|---|
| takumo-ui | push `staging` branch → staging; tag `v*` → prod; manual workflow → dev | AWS S3 + CloudFront, OIDC, then `purge-cache` action |
| takumo-landing-page | push `staging` → staging; push `main` → prod | AWS EC2 via SSM `RunShellScript` → `/var/www/scripts/deploy.sh $BRANCH` |
| takumo-api | Not codified in repo; uses `Dockerfile.migration` + main Dockerfile in `src/Takumo.B2BHotel.API/`. Assume similar CI/CD outside scope. | Docker image (likely ECS / EC2) |
