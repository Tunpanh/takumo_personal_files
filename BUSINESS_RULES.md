# Business Rules

Domain invariants and behavioral rules enforced by the backend. Citations are to actual code so you can audit them.

---

## 1. The Domain in One Sentence

A **hotel guest** ships luggage from a **hotel branch** to a **destination** (another hotel branch, an airport counter, a shop office, or an arbitrary address) using a **shipping carrier**. Hotel staff create bookings on the guest's behalf; the system creates the carrier shipment, tracks status, and notifies the guest. Hotels pay a **subscription** (Stripe) per included branches / metered usage.

---

## 2. Users & Identity

### 2.1 Two parallel identity stores
- **Staff users** (`StaffUser`) belong to a **`Hotel`** and optionally a **`HotelBranch`**. Roles: `Administrator`, `Receptionist`.
- **Traveler users** (`TravelerUser`) are end-customers with optional Google sign-in.
- Both stem from a shared `User` row distinguished by `SystemType` (`Hotel` / `Traveler` / `Admin`).
- Tokens are **not interchangeable** — staff JWTs come from `/api/auth`, traveler JWTs from `/api/travelers/auth`.

### 2.2 Self-action restrictions ([UserErrors.cs](../takumo-api/src/Takumo.B2BHotel.Application/Common/Constants/ErrorCodes/UserErrors.cs))
A user cannot perform these actions on themselves:
- `CannotDeactivateSelf` — administrator can't deactivate own account
- `CannotActivateSelf` — same
- `CannotDeleteSelf`
- `CannotUpdateSelf` — for the admin-level user-management edit path

### 2.3 Password rules
- Hashed with **BCrypt** (Application's `BCrypt.Net-Next` ref).
- Change-password rejects same-as-current (`NewPasswordSameAsCurrent`).
- Reset / verification tokens are one-shot (`VerificationToken` row with `UsedAtUtc`).
- Refresh tokens are typed (`RefreshTokenType` — staff/traveler/firebase) and persisted; revoke marks `RevokedAtUtc`.

### 2.4 Email verification gate
- `User.IsEmailVerified` defaults to `false`.
- Login rejects unverified users with `AuthErrors.EmailNotVerified`.
- Verification token sent via SendGrid on signup; consumed via `POST /verify-email`.

### 2.5 reCAPTCHA
- Login / forgot-password / register endpoints require a reCAPTCHA token verified via `RecaptchaService` (Google Site Verify).
- `AllowedHostnames` whitelist enforced from `RecaptchaSettings`.

### 2.6 Multi-tenancy
- Every read of an `IHotelScoped` / `IBranchScoped` entity is filtered by the current JWT's `HotelId` / `HotelBranchId` (via `IUserContext`).
- Cross-tenant access returns `NotFound`, not `Forbidden`, so the existence of other hotels' data is not leaked.

---

## 3. Bookings

### 3.1 Booking reference format
`BookingRef = "BK{yyyyMMdd}-{sequence}"` — minted by `IBookingReferenceGenerator` at creation. Date portion is server-local **Asia/Tokyo**.

### 3.2 Required consents ([CreateBookingRequestValidator.cs](../takumo-api/src/Takumo.B2BHotel.Application/Validators/Bookings/CreateBookingRequestValidator.cs))
Both must be `true` to create a booking:
- `HasProhibitedGoodsConsent`
- `HasCarrierLiabilityConsent`

Recorded on the `Booking` row for audit.

### 3.3 Pickup date constraints
- `Date >= today` (in TZ Asia/Tokyo) → else `BookingValidationErrors.PickupDateInPast`
- `Date <= today + 1 month` → else `BookingValidationErrors.AdvanceLimitExceeded`

### 3.4 Luggage limits ([CreateLuggageInfoDtoValidator](../takumo-api/src/Takumo.B2BHotel.Application/Validators/Bookings/CreateBookingRequestValidator.cs))
- At least **1** item required
- At most **10** items per booking
- Parcel size must be one of: **60, 80, 100, 120, 140, 160, 180, 200** (Japanese standard 〜cm sizes)
- `CustomDescription` required only when `ItemDescription == Other`, max 500 chars

### 3.5 Address rules
- Guest phone must match Japan phone pattern (`RegexConstants.PhoneFormatRegex`)
- Pickup address comes from the hotel branch's `Location` — verified against the chosen carrier's per-field character limits (`ValidateAddressForCarrier`). Failing fields are returned to the UI so the operator knows what to shorten.
- Destination address must include both English & **Kanji** forms. If the request lacks `Kanji`, the service auto-fills it via `GoogleAddressService.EnsureDestinationLocationKanjiAsync`; if that fails, the booking is rejected with `InvalidBookingDestinationLocation`.
- Destination prefecture must be a serviceable region for the carrier (returned as `InvalidDestinationLocationPrefecture` when not).

### 3.6 Carrier resolution
- `PickupSelection.Carrier` must be one of the branch's active carrier selections (see [§4 Carriers](#4-carriers)).
- `ValidateAndResolveCarrierAsync` ensures the carrier is enabled for the branch and the credentials are present in `HotelBranchCarrierIntegration`.

### 3.7 Status lifecycle ([BookingStatus.cs](../takumo-api/src/Takumo.B2BHotel.Domain/Enums/BookingStatus.cs))

```
Pending → Booked → Collected → InTransit → Delivered
   │        ▲                                  │
   ▼        │                                  ▼
PendingRetry┘                              Returned
   │
   ▼
Failed
       → Cancelled
       → DeliveryFailed
       → Unknown
```

- `Pending` is the initial state; shipment-creation job moves it to `Booked` (success), `PendingRetry`, or `Failed`.
- `_undeliveredStatuses = { Booked, Collected, InTransit }` is the set the tracking poll keeps polling.
- `Cancelled` requires a `CancellationReason` and is only allowed in pre-shipping or in-transit states — otherwise `BookingErrors.CancellationNotAllowed`.

### 3.8 Notifications on booking events
Each state transition emits a `Notification` row (`NotificationTrigger`) AND writes to Firebase Realtime DB under `/notifications/{hotelId}/{branchId}/...` so the staff UI updates in real-time without polling.

---

## 4. Carriers

### 4.1 Provider model
- `ShippingProvider` (integration API) — `ShipAndCo`, `YamatoB2Cloud`
- `ShippingCarrier` (carrier brand) — multiple per provider

Shipment creation goes through `ShipmentCreationStrategyFactory` ([Infrastructure/Strategies/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Strategies/)) which dispatches to `ShipAndCoShipmentCreationStrategy` or `YamatoB2CloudShipmentCreationStrategy`.

### 4.2 Per-branch configuration
- `HotelBranchCarrierIntegration` holds per-branch credentials (one row per branch+provider).
- `HotelBranchCarrierSelection` declares which carriers the branch makes available to staff at booking time.
- A branch with no `HotelBranchCarrierSelection` cannot create bookings — `GET /api/branches/unconfigured-carrier` lists such branches.
- `POST /api/carrier-configurations/test-connection` validates credentials before saving via `CarrierConnectionTestStrategyFactory`.

### 4.3 Pickup time windows
- `CarrierPickupTimeZone` defines which `PickupTimeZone`s (morning/afternoon/evening) each carrier accepts.
- Mismatched selection → `BookingValidationErrors.PickupTimeCarrierMismatch`.

### 4.4 Retry policy ([ShipmentRetryBackgroundJobService.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/BackgroundJobs/ShipmentRetryBackgroundJobService.cs))
- Failed carrier calls become `ShipmentApiFailureLog` rows + `Booking.Status = PendingRetry`.
- Hangfire job re-tries on a schedule; on `MaxConsecutiveFailuresBeforeDisable` (for Yamato: 10) the integration is disabled and an admin notification fires.

### 4.5 Yamato tracking poll ([YamatoTrackingPollJob.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/BackgroundJobs/YamatoTrackingPollJob.cs))
Per-customer throttling enforced strictly so Yamato's API isn't overrun:
- ≤ 500 bookings per tick (`MaxBookingsPerTick`)
- ≤ 20 tracking numbers per HTTP call (`MaxNumbersPerCall`)
- ≥ 5.5 s between requests for the same customer (`MinIntervalSecondsPerCustomer`)
- Max 3 HTTP retries

---

## 5. Hotels & Branches

### 5.1 Main branch
- `HotelBranch.IsMainBranch` — one main branch per hotel (enforced at service layer when creating/updating).

### 5.2 Branch caps
- `HotelSubscription.NumberOfBranches` tracks paid-for branch count.
- `SubscriptionPlan.IncludedBranches` is the baseline.
- If `IsAllowAddMoreBranches`, branches beyond included are billed via the metered Stripe price (`StripeMeteredPriceId` / `MeteredBaseFee`).
- Creating a branch beyond plan capacity is rejected (admins see plan-upgrade prompt).

### 5.3 Branch deletion
- Soft delete (`ISoftDelete`). Active bookings on the branch block deletion (verified before mark-deleted).

---

## 6. PMS Integration

### 6.1 Supported providers
`Mews`, `Cloudbeds`, `Oracle` (declared in [appsettings.json `PmsProviderSettings.Providers`](../takumo-api/src/Takumo.B2BHotel.API/appsettings.json)).

### 6.2 Per-provider auth schema
The provider schema is server-driven — the frontend renders the auth form dynamically from `/api/pms-configurations/providers`, so adding a new provider requires no UI change beyond service-side strategy.

| Provider | Required fields |
|---|---|
| Mews | `ApiUrl`, `ClientToken`, `AccessToken` |
| Cloudbeds | `ApiUrl`, `PropertyIDs`, `ApiKey` |
| Oracle | `ApiUrl`, `ClientID`, `ClientSecret`, `EnterpriseID`, `ApplicationKey`, `Scope`, `HotelID` |

### 6.3 Toggle behavior
- `HotelPmsIntegration.IsEnabled = false` keeps credentials but bypasses guest search.
- A `HotelBranch` without `HotelPmsIntegrationId` is listed by `GET /api/branches/unconfigured-pms`.

### 6.4 Guest search
- `SearchGuestService` first looks at local `GuestInfo`, then queries PMS if integration is enabled.
- Results are merged with conflict resolution favoring PMS data (it's the source of truth for active reservations).

---

## 7. Subscriptions & Billing

### 7.1 Plan model
A plan can offer **monthly**, **yearly**, and/or **metered** pricing (`StripeMonthlyPriceId`, `StripeYearlyPriceId`, `StripeMeteredPriceId`). Currency is per-plan.

### 7.2 Trial
- Default trial: `StripeSettings.DefaultTrialPeriodDays = 7`.
- Trial-ending notification triggered by Stripe `customer.subscription.trial_will_end` → `TrialWillEndHandler`.

### 7.3 Renewal reminders
Driven by `TakumoSettings.SubscriptionExpiringReminderDays`:
- **Auto-renewal** subscriptions: reminder 3 days before next charge
- **Non-auto-renewal**: reminders 7 and 1 days before cancellation
- Job: [SubscriptionBackgroundJobService.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/BackgroundJobs/SubscriptionBackgroundJobService.cs)

### 7.4 Cancellation
- `POST /api/subscriptions/cancel` flags `IsCancelAtPeriodEnd = true` in Stripe and the local row.
- Cancellation is **effective at period end**, not immediately — features remain available until then.

### 7.5 Payment retry
- `POST /api/subscriptions/retry-payment` re-attempts the latest invoice via Stripe.
- Stripe `invoice.payment_failed` increments past-due state; multiple failures move subscription to `PastDue`/`Unpaid`.

### 7.6 Webhook idempotency
- Every Stripe event is persisted in `StripeWebhookEvent`. Handlers check `ProcessedAtUtc` and skip duplicates — Stripe may deliver the same event multiple times.

---

## 8. Guest Pass & Wallet

### 8.1 Pass creation
- `POST /api/travelers/passes` is **anonymous** — anyone with the right inputs can create a pass. Authentication is by `AuthenticationToken` + `GuestEmailHash` on the pass itself.
- `AuthenticationToken` is required to read the pass back; it's also what Apple Wallet sends in `Authorization: ApplePass <token>`.

### 8.2 Apple Wallet rules
- `PassTypeIdentifier`: `pass.com.takumo.passmaker`
- `TeamIdentifier`: `65TN39WV65`
- Pass background `#A12850` (Takumo brand red).
- Devices register via `POST /v1/devices/.../registrations/...` and receive APNs pushes on pass changes — handled by `ApnsPushNotificationService` (token-auth, not certificate).

### 8.3 Google Wallet rules
- Pass class suffix `HotelGuestPass`, object ID prefix `guest_`.
- A signed JWT save-link is minted per pass — link is single-use to add to wallet, after which Google Wallet manages updates.

### 8.4 Pass updates
- On any pass-relevant change (booking status, pickup time, carrier), `UpdateWalletPassBackgroundJobService` re-issues the pass JSON and notifies Apple/Google.

---

## 9. Legal Documents (Terms & Privacy)

- The UI carries `termsVersion` and `privacyVersion` strings in `app-config.json`.
- On accepting, the UI calls `POST /api/legal-documents/consent` which writes a `UserLegalConsent` row pinning user + document type + version + timestamp.
- A subsequent version change re-prompts on next login — gated client-side by comparing the stored `termsVersion` with the user's last consent.

---

## 10. Audit

- Every entity implementing `IAuditable` is captured by an EF Core `SaveChangesInterceptor` and written to `AuditLog`:
  - `EntityType`, `EntityId`, `Action` (Create/Update/Delete/SoftDelete/Restore), `UserId`, `Diff` (JSON)
- Properties marked `[AuditIgnore]` are excluded from the diff (e.g. `PasswordHash`, refresh tokens).
- Query via `GET /api/audit-logs` (admin only).

---

## 11. Notifications

- Persisted in `Notification` table AND mirrored to Firebase Realtime DB.
- Per-user read state was flattened in migration `20251231042555_FlattenNotificationUserStatuses` — each (user, notification) is its own row, so reads scale with hotel size.
- Frontend listens on `/notifications/{hotelId}/{branchId}` via AngularFire; clicking marks read via `POST /api/notifications/{id}/read`.

---

## 12. Localization Rules

- `User.PreferredLanguage` (`En` / `Ja`) drives:
  - Transactional emails (RazorLight template selection in [Infrastructure/Templates/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Templates/))
  - Default content language for Wallet passes (the pass JSON has multi-language fallback)
- The UI's URL prefix (`/en/` / `/ja/`) is independent — staff can browse in EN even with `Ja` preference.
