# API Reference

`takumo-api` is the single HTTP surface for both SPAs. All paths below are relative to the API base (e.g. `https://api.example.com`).

- **Base prefix:** `/api` (from `ApiConstants.BaseRoute`; some Apple Wallet endpoints use `ApiConstants.VersionV1` = `/v1`)
- **Versioning:** No path-version yet — single live version. Wallet web service uses `/v1` per Apple's spec.
- **Discovery:** Swagger at `/swagger` in Development & Staging. Spec at `/swagger/v1/swagger.json`.
- **Content type:** `application/json` for almost everything. `application/vnd.apple.pkpass` for Apple Wallet `.pkpass`. `multipart/form-data` only on upload endpoints.
- **Max request body:** 50 MB (configured in `Program.cs`).
- **Date format:** ISO-8601, **timezone Asia/Tokyo** (from `DateTimeSettings.TimeZone`).
- **Rate limiting:** Enabled globally via `RateLimitingExtensions` — see [src/Takumo.B2BHotel.API/Extentions/](../takumo-api/src/Takumo.B2BHotel.API/Extentions/).

---

## 1. Authentication

### Scheme

JWT Bearer in the `Authorization` header (`Bearer <accessToken>`). Configured in `JwtSettings`:

| Setting | Value |
|---|---|
| Issuer | `Takumo.B2BHotel.API` |
| Audience | `Takumo.B2BHotel.Client` |
| Access token expiry | 1 hour |
| Refresh token expiry | 7 days |
| Clock skew | 0 |

### Authorization Attributes

| Attribute | Source | Meaning |
|---|---|---|
| `[AllowAnonymous]` | ASP.NET | No auth required |
| `[Authorize]` | ASP.NET | Any valid JWT |
| `[RequireAccessToken]` | [Attributes/](../takumo-api/src/Takumo.B2BHotel.API/Attributes/) | Custom — admin/system access tokens |
| `[RoleAuthorization]` | [Attributes/](../takumo-api/src/Takumo.B2BHotel.API/Attributes/) | Any active staff user (any role) |
| `[RoleAuthorization(UserRole.Administrator)]` | same | Administrator role only |

`UserRole` values: `Administrator`, `Receptionist` (see [UserRole.cs](../takumo-api/src/Takumo.B2BHotel.Domain/Enums/UserRole.cs)).

### Token endpoints

| Audience | Endpoint family |
|---|---|
| Hotel staff | `/api/auth/*` |
| Travelers (B2C) | `/api/travelers/auth/*` |

These two sets of users are **separate identity stores** (`StaffUser` vs `TravelerUser`); their tokens are not interchangeable.

---

## 2. Endpoint Catalog — Hotel Staff (B2B)

### Auth — [`AuthController.cs`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/AuthController.cs) `/api/auth`

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/login` | anon | Email/password → tokens |
| POST | `/refresh` | anon | Refresh access token |
| POST | `/revoke` | bearer | Revoke refresh token |
| GET  | `/me` | role | Current user info |
| GET  | `/firebase-token` | bearer | Mint Firebase auth token (for realtime DB listening) |
| POST | `/firebase-token/refresh` | anon | Refresh Firebase token |
| POST | `/verify-email` | anon | Confirm email with token |
| GET  | `/verify-reset-token` | anon | Validate password-reset token |
| POST | `/forgot-password` | anon | Email reset link |
| POST | `/reset-password` | anon | Set new password |
| PATCH | `/password` | role | Change password (logged in) |
| POST | `/register` | anon | Hotel/admin self-signup |

### Bookings — [`BookingController.cs`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/BookingController.cs) `/api/bookings`

All require `[RoleAuthorization]` (any active staff).

| Method | Path | Purpose |
|---|---|---|
| POST | `/` | Create booking |
| GET  | `/` (with query) | List bookings (branch-scoped, paged, filterable) |
| GET  | `/{id}` | Booking detail |
| GET  | `/{id}/status` | Status-only lookup (lighter) |
| GET  | `/{id}/download-label` | Carrier shipping label PDF |
| POST | `/{id}/cancel` | Cancel booking with reason |
| POST | `/estimate-rate` | Pre-create shipping rate quote |
| POST | `/{id}/resend-email` | Re-send confirmation email |

### Hotel Branches — [`HotelBranchController.cs`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/HotelBranchController.cs) `/api/branches`

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/` | admin | Create branch |
| GET  | `/{id}` | role | Branch detail |
| PUT  | `/{id}` | admin | Update branch |
| DELETE | `/{id}` | admin | Soft-delete branch |
| GET  | `/` (paged) | admin | List branches |
| GET  | `/all` | role | All branches (lightweight) |
| GET  | `/unconfigured-pms` | role | Branches missing PMS config |
| GET  | `/unconfigured-carrier` | role | Branches missing carrier config |
| GET  | `/{id}/carrier-configurations` | role | Carrier configs available to branch |
| PUT  | `/{id}/carrier-selections` | admin | Set branch's active carrier(s) |
| GET  | `/{id}/carrier-selections` | anon | Public read (used by B2C / Wallet) |

### Staff Users — [`StaffsController.cs`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/StaffsController.cs) `/api/staffs`

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET  | `/profile` | role | Self profile |
| PUT  | `/profile` | role | Update self |
| GET  | `/{id}` | admin | Staff detail |
| GET  | `/` (paged) | admin | List staff |
| POST | `/` | admin | Invite new staff |
| POST | `/{id}/resend-invitation` | admin | Re-send invitation email |
| PUT  | `/{id}` | admin | Update staff |
| DELETE | `/{id}` | admin | Soft-delete staff |
| PATCH | `/{id}/activate` | admin | Activate (with reason) |
| PATCH | `/{id}/deactivate` | admin | Deactivate (with reason) |
| GET  | `/available-receptionists` | role | Receptionists eligible for branch assignment |

### Guests — [`GuestInfoController.cs`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/GuestInfoController.cs) `/api/guests`

`[RoleAuthorization]`. Search (PMS-backed if integration configured) + CRUD on locally stored `GuestInfo`.

### Notifications — [`NotificationController.cs`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/NotificationController.cs) `/api/notifications`

| Method | Path | Purpose |
|---|---|---|
| POST | `/{id}/read` | Mark single as read |
| POST | `/{id}/unread` | Mark single as unread |
| POST | `/read-all` | Mark all as read |
| GET  | `/unread-count` | Badge count |
| GET  | `/unread` | List unread |

### PMS Configuration — [`PmsConfigurationController.cs`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/PmsConfigurationController.cs) `/api/pms-configurations`

`[RoleAuthorization(UserRole.Administrator)]` on class.

| Method | Path | Purpose |
|---|---|---|
| GET  | `/providers` | List supported PMS providers + their auth-field schemas |
| POST | `/{id}/toggle` | Enable / disable integration |
| PATCH | `/{id}/basic-info` | Update name, branch links |
| PATCH | `/{id}/auth-config` | Update credentials (per-provider fields) |
| DELETE | `/{id}` | Remove integration |

Supported `PmsProvider`: `Mews`, `Cloudbeds`, `Oracle` (auth-field schemas in [appsettings.json `PmsProviderSettings`](../takumo-api/src/Takumo.B2BHotel.API/appsettings.json)).

### Carrier Configuration — [`CarrierConfigurationController.cs`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/CarrierConfigurationController.cs) `/api/carrier-configurations`

Admin-only.

| Method | Path | Purpose |
|---|---|---|
| GET  | `/providers` | Supported `ShippingProvider`s and auth-field schemas |
| GET  | `/{id}` | Detail |
| PATCH | `/{id}/basic-info` | Update name |
| PATCH | `/{id}/integration-settings` | Update credentials |
| DELETE | `/{id}` | Remove integration |
| POST | `/test-connection` | Validate credentials before saving (via Strategy factory) |

### Subscription / Plans / Invoices

| Controller | Route | Auth | Notable endpoints |
|---|---|---|---|
| [`SubscriptionController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/SubscriptionController.cs) | `/api/subscriptions` | admin | `PATCH /plan`, `POST /auto-renew/toggle`, `POST /cancel`, `POST /retry-payment` |
| [`SubscriptionPlanController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/SubscriptionPlanController.cs) | `/api/subscription-plans` | anon | `GET /all`, `GET /total-fee` |
| [`CheckoutController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/CheckoutController.cs) | `/api/checkout` | anon | `POST /session` — create Stripe Checkout session |
| [`InvoiceController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/InvoiceController.cs) | `/api/invoices` | admin | `GET /{invoiceId}`, `GET /{invoiceId}/download` |
| [`PaymentMethodController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/PaymentMethodController.cs) | `/api/payment-methods` | admin | `POST /setup-intent`, `POST /setup-intent/confirm`, `POST /{paymentMethodId}/set-default`, `DELETE /{paymentMethodId}` |
| [`TransactionController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/TransactionController.cs) | `/api/transactions` | admin | List + detail |
| [`StripeWebhookController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/StripeWebhookController.cs) | `/api/stripe-webhook` | anon (signature-verified) | Receives all Stripe events; dispatch via [Webhooks/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Webhooks/) |

### Misc

| Controller | Route | Notes |
|---|---|---|
| [`AddressController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/AddressController.cs) | `/api/addresses` | `POST /validate` (Google Places + business rules) |
| [`AuditLogController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/AuditLogController.cs) | `/api/audit-logs` | Admin-only; filter by entity/action/user |
| [`HealthController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/HealthController.cs) | `/api/health` | `GET /`, `GET /detailed` |
| [`LegalDocumentsController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/LegalDocumentsController.cs) | `/api/legal-documents` | `POST /consent` (record acceptance of versioned terms/privacy) |
| [`MetadataController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/MetadataController.cs) | `/api/metadata` | All `[AllowAnonymous]` enum-and-reference lookups — see below |
| [`ShipmentsController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/ShipmentsController.cs) | `/api/shipments` | `GET /{trackingNumber}` |
| [`SupportTicketController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/SupportTicketController.cs) | `/api/support-tickets` | CRUD support tickets |
| [`GuestPassController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/GuestPassController.cs) | `/api/guest-passes` | `GET /{id}` |

#### `/api/metadata` (all anonymous, GET)

Static-ish reference data. Caches well in the client.

`countries`, `japan-airports`, `shop-offices`, `booking-statuses`, `shipping-carriers`, `pickup-time-zones`, `luggage-types`, `luggage-sizes`, `luggage-specials`, `shipping-endpoints`, `reasons/activation`, `reasons/deactivation`, `reasons/deletion`, `user-roles`, `preferred-languages`, `audit-entities`, `audit-actions`, `bookings-special-handlings`, `luggage-item-descriptions`.

---

## 3. Endpoint Catalog — Admin (System Console)

All under `[RequireAccessToken]` — system/admin operators only.

| Controller | Route | Endpoints |
|---|---|---|
| [`AdminHotelController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/Admin/AdminHotelController.cs) | `/api/hotels` | CRUD hotels |
| [`AdminFeatureController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/Admin/AdminFeatureController.cs) | `/api/admin/features` | `GET /all`, `PUT /{id}`, `DELETE /{id}`, `GET /subscription-plan/{planId}`, `GET /check-access/{hotelId}/{featureCode}` |
| [`AdminSubscriptionController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/Admin/AdminSubscriptionController.cs) | `/api/admin/subscriptions` | `POST /reminders/expiring/send` (manual trigger of scheduled job) |
| [`AdminSubscriptionPlanController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/Admin/AdminSubscriptionPlanController.cs) | `/api/admin/subscription-plans` | CRUD subscription plans (with features, billing periods) |

---

## 4. Endpoint Catalog — Travelers (B2C)

### Traveler Auth — [`TravelerAuthController.cs`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/Traveler/TravelerAuthController.cs) `/api/travelers/auth`

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/register` | anon | Email-password signup |
| POST | `/login` | anon | Email-password login |
| POST | `/google` | anon | Google federated sign-in (`GoogleAuthService`) |
| POST | `/refresh` | anon | Refresh token |
| POST | `/revoke` | bearer | Revoke refresh token |
| GET  | `/me` | bearer | Current traveler info |
| POST | `/verify-email` | anon | Email confirm |
| POST | `/forgot-password` | anon | |
| POST | `/reset-password` | anon | |
| GET  | `/verify-reset-token` | anon | |
| PATCH | `/password` | bearer | Change password |

### Traveler Profile / Bookings / Misc

| Controller | Route | Endpoints |
|---|---|---|
| [`TravelerProfileController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/Traveler/TravelerProfileController.cs) | `/api/travelers/profile` | Get/Update profile (bearer) |
| [`TravelerBookingController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/Traveler/TravelerBookingController.cs) | `/api/travelers/bookings` | `GET /bookings`, `GET /bookings/{id}` (bearer) |
| [`TravelerController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/Traveler/TravelerController.cs) | `/api/travelers` | `GET /hotels` (anon — for booking start) |
| [`TravelerPassController`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/Traveler/TravelerPassController.cs) | `/api/travelers/passes` | `POST /passes` (anon create), `GET /passes/google` (anon save-link), `GET /passes/apple` (anon download .pkpass), `GET /passes` (bearer list), `PUT /passes` (bearer update), `PUT /passes/{passId}` (anon — used by Wallet update flow) |

---

## 5. Apple Wallet Web Service

[`AppleWalletWebServiceController.cs`](../takumo-api/src/Takumo.B2BHotel.API/Controllers/AppleWalletWebServiceController.cs) — base route `/v1` per Apple's [PassKit Web Service Reference](https://developer.apple.com/library/archive/documentation/PassKit/Reference/PassKit_WebService/WebService.html).

| Method | Path |
|---|---|
| POST   | `/v1/devices/{deviceLibraryIdentifier}/registrations/{passTypeIdentifier}/{serialNumber}` |
| DELETE | `/v1/devices/{deviceLibraryIdentifier}/registrations/{passTypeIdentifier}/{serialNumber}` |
| GET    | `/v1/devices/{deviceLibraryIdentifier}/registrations/{passTypeIdentifier}` |
| GET    | `/v1/passes/{passTypeIdentifier}/{serialNumber}` |
| POST   | `/v1/log` |

All `[AllowAnonymous]` — devices authenticate via Apple's pass-specific token in `Authorization: ApplePass <token>`.

---

## 6. Error Format

`GlobalExceptionHandler` returns RFC 7807 Problem Details:

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.6.1",
  "title": "An error occurred",
  "status": 500,
  "detail": "...",
  "instance": "/api/bookings",
  "errorCode": "BOOKING_VALIDATION_FAILED",
  "errors": { "field": ["message"] }   // for 400 validation
}
```

Domain-defined error codes are extracted and documented at the OpenAPI level by `ErrorCodeDocumentFilter` (Swagger).

---

## 7. Frontend Consumption

Each Angular API service in [projects/web-b2b-hotel/src/app/core/apis/](../takumo-ui/projects/web-b2b-hotel/src/app/core/apis/) (and the B2C equivalent) wraps a controller. Naming: `AuthenticationApi` → `AuthController`, `BookingApi` → `BookingController`, etc.

Wire types live next to the API service as `*.dao.d.ts`. View-layer mapping is done in `app/shared/converters/`. Base URL is injected at runtime via `baseApiPathInterceptorFactoryFunc` (see [FLOWS.md §1](./FLOWS.md#1-hotel-staff-sign-in-b2b)).
