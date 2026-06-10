# Data Model

Authoritative source: PostgreSQL via Entity Framework Core 9, defined in [takumo-api/src/Takumo.B2BHotel.Domain/Entities/](../takumo-api/src/Takumo.B2BHotel.Domain/Entities/) with type configurations in [Infrastructure/Data/Configurations/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Data/Configurations/).

- **DbContext:** [TakumoDbContext.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Data/TakumoDbContext.cs)
- **Migrations:** 87 files under [Infrastructure/Migrations/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Migrations/) — first `20251126063159_InitialMigration`, latest as of writing `20260213081314_AddSupportTickets`
- **Primary keys:** `Guid` everywhere (via `BaseEntity<TPrimaryKey>` defaulted to `Guid`)
- **Schema dump:** [database/takumo_schema.sql](../takumo-api/database/takumo_schema.sql)

---

## 1. Base Types & Cross-cutting Interfaces

All in [Domain/Common/](../takumo-api/src/Takumo.B2BHotel.Domain/Common/).

| Type | Role |
|---|---|
| `BaseEntity<TPK>` / `BaseEntity` | `Id` (Guid by default) |
| `BaseAuditableEntity<TPK>` / `BaseAuditableEntity` | adds `CreatedAtUtc`, `CreatedBy` (Guid?), `CreatedByUser` (nav), `LastModifiedAtUtc`, `LastModifiedBy`, `LastModifiedByUser` |
| `IBaseAuditableEntity<TPK>` | interface form |
| `IAuditable` | opt-in: changes recorded to `AuditLog` table |
| `ISoftDelete` | global query filter excludes deleted; physical row remains |
| `IHotelScoped` | entity carries `HotelId` — used to enforce tenant isolation |
| `IBranchScoped` | entity carries `HotelBranchId` |
| `[AuditIgnore]` | property attribute — exclude field from audit diff |
| `[EnumCode("entity", "field")]` | tags enum so `/api/metadata/<name>` can stringify it consistently |

User audit fields (`CreatedByUser`, `LastModifiedByUser`) navigate to `User` — set by `DbContextUserContext` interceptor from the current JWT.

---

## 2. Entity Catalog (35 entities)

Grouped by domain area. Field lists show only the essentials; see source for full property sets.

### Identity & Authentication

| Entity | Key fields | Purpose |
|---|---|---|
| `User` | `Email`, `PasswordHash`, `SystemType`, `AuthProvider`, `GoogleId?`, `IsEmailVerified`, `IsActive`, `PreferredLanguage` | Shared identity for staff and travelers. Polymorphic via `SystemType`. |
| `StaffUser` | `UserId`, `HotelId`, `HotelBranchId?`, `Role` (`Administrator`/`Receptionist`) | Hotel-side identity. Implements `IHotelScoped`. |
| `TravelerUser` | `UserId`, `GuestPassId?` | Customer-side identity. |
| `RefreshToken` | `UserId`, `Token`, `TokenType`, `ExpiresAtUtc`, `RevokedAtUtc?` | JWT refresh. `TokenType` distinguishes staff vs traveler vs Firebase. |
| `VerificationToken` | `UserId`, `Token`, `TokenType` (EmailVerification, PasswordReset), `ExpiresAtUtc`, `UsedAtUtc?` | One-shot tokens emailed to user. |
| `UserLegalConsent` | `UserId`, `DocumentType`, `Version`, `AcceptedAtUtc` | Versioned terms / privacy acceptance audit. |

### Hotel Structure (multi-tenant root)

| Entity | Key fields | Purpose |
|---|---|---|
| `Hotel` | `Name`, `HotelSubscriptionId?` | Root tenant. Has many branches, staff, PMS integrations. |
| `HotelBranch` | `HotelId`, `Name`, `Location` (Address VO), `Email`, `Phone`, `IsMainBranch`, `HotelPmsIntegrationId?`, `HotelBranchCarrierIntegrationId?`, `HotelBranchCarrierSelectionId?` | Physical location. `IHotelScoped`. |
| `HotelPmsIntegration` | `HotelId`, `PmsType` (Mews/Cloudbeds/Oracle), `IsEnabled`, `AuthConfig` (JSON) | Per-hotel PMS connection. |
| `HotelBranchCarrierIntegration` | `HotelBranchId`, carrier credentials | Per-branch shipping carrier auth. |
| `HotelBranchCarrierSelection` | `HotelBranchId`, `ActiveCarriers[]` | Which carriers a branch may use. |
| `CarrierProviderConfiguration` | `Carrier`, schema of fields | Reference table for available carriers. |
| `CarrierPickupTimeZone` | `Carrier`, `PickupTimeZone`, time-range | Allowed pickup windows per carrier. |

### Bookings & Luggage (core revenue object)

| Entity | Key fields | Purpose |
|---|---|---|
| `Booking` | `BookingRef` (`BK{yyyyMMdd}-{seq}`), `HotelId`, `HotelBranchId`, `DestinationHotelBranchId?`, `GuestInfoId`, `BookingCareId`, `Status` ([BookingStatus](../takumo-api/src/Takumo.B2BHotel.Domain/Enums/BookingStatus.cs)), `PickupInfo` (VO), `DestinationInfo` (VO), `CarrierName` ([ShippingCarrier](../takumo-api/src/Takumo.B2BHotel.Domain/Enums/ShippingCarrier.cs)), `ShipmentId?`, `TrackingNumber?`, `LabelUrl?`, `CancellationReason?`, `HasProhibitedGoodsConsent`, `HasCarrierLiabilityConsent`, `SendEmailStatus`, `ShipmentBackgroundJobId?`, `ArrivalDate?`, `LuggageSpecialNotes?` | The shipment order. `IBranchScoped`. |
| `LuggageInfo` | `BookingId`, `LuggageType`, `ParcelSize`, `Weight?`, `SpecialHandling?`, `ItemDescription`, `CustomDescription?` | Individual piece(s) per booking. |
| `BookingShipmentHistory` | `BookingId`, status timeline rows | Append-only status log. |
| `ShipmentApiFailureLog` | `BookingId`, `ApiType`, payload, error | Carrier API failures for diagnostic + retry. |
| `GuestInfo` | `FirstName`, `LastName`, `Email`, `AdditionalEmail?`, `Phone`, `RoomNumber?` | Per-booking guest snapshot (denormalized — guests are not full users). |

### Notifications

| Entity | Key fields | Purpose |
|---|---|---|
| `Notification` | `HotelId`, `HotelBranchId?`, `Trigger`, `ObjectType`, `ObjectId`, `Title`, `Body`, per-user read state | Persistent notification record, also mirrored to Firebase RTDB. |

### Travel Pass / Wallet

| Entity | Key fields | Purpose |
|---|---|---|
| `GuestPass` | `HotelBranchId?`, `TravelerId?`, guest name/email/phone, `PickupDate?`, `Carrier?`, `PickupTime?`, `DestinationInfo?`, `LuggageInfo` (List of VO), `AuthenticationToken`, `GuestEmailHash` | Digital pass usable from Wallet apps. |
| `ApplePassDevice` | `DeviceLibraryIdentifier`, `PushToken` | Apple device registered for pass updates. |
| `ApplePassRegistration` | `DeviceId`, `PassTypeIdentifier`, `SerialNumber` (=GuestPassId) | Per-device pass registration. |
| `GooglePassRegistration` | `GuestPassId`, Google Wallet object/class refs | Google Wallet pass instances. |

### Subscriptions & Billing

| Entity | Key fields | Purpose |
|---|---|---|
| `HotelSubscription` | `HotelId`, `SubscriptionPlanId`, `StripeCustomerId`, `StripeSubscriptionId`, `Status` ([SubscriptionStatus](../takumo-api/src/Takumo.B2BHotel.Domain/Enums/SubscriptionStatus.cs)), `BillingPeriod`, `CurrentPeriodStart/End`, `IsCancelAtPeriodEnd`, `NumberOfBranches` | The active subscription per hotel. |
| `SubscriptionPlan` | `Name`, `StripeMonthlyPriceId?`, `MonthlyBaseFee?`, `StripeYearlyPriceId?`, `YearlyBaseFee?`, `StripeMeteredPriceId?`, `MeteredBaseFee?`, `Currency`, `IncludedBranches`, `IsAllowAddMoreBranches`, `DisplayOrder`, `IsActive` | Plan catalog. |
| `SubscriptionPlanFeature` | `SubscriptionPlanId`, `FeatureId`, `Limit?` | Plan ↔ feature link with optional usage limit. |
| `Feature` | `FeatureCode`, `Name`, `Description` | Feature catalog (toggleable per plan). |
| `Invoice` | `HotelId`, `StripeInvoiceId`, `StripeInvoiceNumber`, `AmountPaid`, `Currency`, `PaymentMethod?`, `PaidAt`, `Status`, `BillingAddress?`, `InvoicePdfUrl` | Stripe invoice mirror with PDF link. |
| `StripeWebhookEvent` | `EventId`, `EventType`, payload, `ProcessedAtUtc?` | Idempotency table for Stripe events. |

### Reference / Catalog

| Entity | Purpose |
|---|---|
| `Airport` | Japan airport catalog (JAL/ANA codes, name, region) |
| `ShopOffice` | Carrier shop/office endpoints (Yamato counters, etc.) |
| `YamatoB2CloudConfiguration` | Yamato B2 Cloud account config |
| `YamatoTrackingState` | Per-customer Yamato tracking poll state |

### Audit / Support

| Entity | Purpose |
|---|---|
| `AuditLog` | `EntityType`, `EntityId`, `Action`, `UserId`, `Diff` (JSON), `Timestamp`. Written by EF Core `SaveChangesInterceptor`. |
| `SupportTicket` | `HotelId`, `Subject`, `Body`, `Status`, `Priority` |

---

## 3. Value Objects

In [Domain/ValueObjects/](../takumo-api/src/Takumo.B2BHotel.Domain/ValueObjects/), persisted as **owned types** (no separate table; columns inlined on owner).

| VO | Fields | Used by |
|---|---|---|
| `Address` | `Country`, `Zip`, `Province`, `City`, `AddressLine1/2`, plus Kanji versions (`FormattedAddressKanji`, `ProvinceKanji`, `CityKanji`, `TownKanji`, `AddressLine1Kanji`, `AddressLine2Kanji`) | `HotelBranch.Location`, `PickupInfo.PickupLocation`, `DestinationInfo`, `Invoice.BillingAddress` (serialized) |
| `PickupInfo` | `PickupLocation` (Address), `PickupDate` (DateOnly), `PickupTime` (PickupTimeZone?) | `Booking.PickupInfo` |
| `DestinationInfo` | dest hotel branch / address / `AirportCounter`, `ShippingEndpoint` discriminator | `Booking.DestinationInfo`, `GuestPass.DestinationInfo` |
| `AirportCounter` | airport + counter identifier | nested in `DestinationInfo` |
| `LuggageInfoObject` | snapshot fields for guest pass | `GuestPass.LuggageInfo` (List) |
| `ActivationReason` / `DeactivationReason` / `DeletionReason` | `Type` + `OtherDescription?` | On `User`, `StaffUser`, etc. |

Bilingual `Address` (English + Kanji) reflects the Japan-centric domain — many Japanese addresses are stored in both romaji and kanji forms.

---

## 4. Enums

All in [Domain/Enums/](../takumo-api/src/Takumo.B2BHotel.Domain/Enums/). Most carry `[EnumCode]` so `/api/metadata/*` returns stable string codes.

### Booking lifecycle
- **`BookingStatus`** — `Pending`, `PendingRetry`, `Failed`, `Booked`, `Collected`, `InTransit`, `Delivered`, `Cancelled`, `DeliveryFailed`, `Returned`, `Unknown`
- **`BookingSpecialHandling`** — fragile, refrigerate, etc.
- **`SendEmailStatus`** — confirmation email delivery state

### Luggage
- **`LuggageType`** — suitcase, backpack, box, etc.
- **`LuggageSize`** — S/M/L bands
- **`LuggageSpecial`** — special-handling categories
- **`LuggageItemDescription`** — pre-defined item categories (allows `CustomDescription` when "Other")

### Shipping
- **`ShippingCarrier`** — Yamato, Sagawa, etc.
- **`ShippingProvider`** — `ShipAndCo`, `YamatoB2Cloud` (the integration API, not the consumer-facing brand)
- **`ShippingEndpoint`** — destination kind: `HotelBranch`, `Address`, `AirportCounter`
- **`PickupTimeZone`** — morning, afternoon, evening windows
- **`CollectAmountMode`** — payment-on-pickup vs prepaid
- **`ShipmentApiType`** — discriminates failure-log source

### Identity / Auth
- **`UserRole`** — `Administrator`, `Receptionist`
- **`SystemType`** — `Hotel`, `Traveler`, `Admin`
- **`AuthProvider`** — `Local`, `Google`
- **`PortalType`** — which portal the user belongs to
- **`Language`** — `En`, `Ja`
- **`RefreshTokenType`** / **`VerificationTokenType`**

### Subscriptions / Billing
- **`SubscriptionStatus`** — `Trialing`, `Active`, `PastDue`, `Canceled`, `Unpaid`, …
- **`BillingPeriod`** — `Monthly`, `Yearly`
- **`InvoiceStatus`**
- **`FeatureCode`** — opaque codes referenced in `SubscriptionPlanFeature`

### PMS
- **`PmsProvider`** — `Mews`, `Cloudbeds`, `Oracle`

### Notifications
- **`NotificationTrigger`** — booking events, shipment events, billing events
- **`NotificationObjectType`** — type of the linked entity

### Other
- **`Status`** — generic active/inactive
- **`ActivationReasonType`**, **`DeactivationReasonType`**, **`DeletionReasonType`** — categorized reasons
- **`AuditLogAction`** — Create / Update / Delete / SoftDelete / Restore
- **`AuditEntity`** — entity type catalog

---

## 5. Database Conventions

| Concern | Convention |
|---|---|
| Naming | EF Core default snake_case via `Npgsql.EntityFrameworkCore.PostgreSQL` (verify per `BaseAuditableEntityConfiguration`) |
| Primary key | `Guid`, generated by EF Core (`HiLo` or default UUID) |
| Auditing | `CreatedAtUtc`, `CreatedBy(User)`, `LastModifiedAtUtc`, `LastModifiedBy(User)` on every `BaseAuditableEntity` row |
| Soft delete | `ISoftDelete` adds `IsDeleted` + `DeletedAtUtc?`; global query filter applied in `OnModelCreating` |
| Tenancy | `IHotelScoped` / `IBranchScoped` are marker interfaces. Filtering is enforced by services using `IUserContext` (current JWT's hotelId/branchId), not by query filters. |
| Concurrency | Default (no explicit RowVersion on most entities) |
| JSON columns | Used for `HotelPmsIntegration.AuthConfig`, `AuditLog.Diff`, `Notification` user-status flatten, `BillingAddress` |
| Migrations | One EF migration per logical change; PR author runs `dotnet ef migrations add` in `Infrastructure` project |

---

## 6. Migrations Worth Knowing

Reading the migration names tells you the evolution of the model:

| Migration | What it added |
|---|---|
| `20251126063159_InitialMigration` | Whole initial schema |
| `20251203033611_UpdateTravelerGuestPassRelationship` | Linked `TravelerUser.GuestPassId` |
| `20251209090950_AddAuditUserNavigationProperties` | `CreatedByUser` / `LastModifiedByUser` navs |
| `20251215030936_AddTokenTypeForRefreshToken` | `RefreshToken.TokenType` enum column |
| `20251216034722_AddMeteredBaseFeeToSubscriptionPlan` | Usage-based pricing |
| `20251218073835_RemoveAdditionalBranchFeeInSubscriptionPlan` | Simplified pricing |
| `20251218100220_AddDisplayOrderInSubscriptionPlan` | Plan list ordering |
| `20251220130003_AddMoreFieldInUserAccountTable` | User profile expansion |
| `20251231042555_FlattenNotificationUserStatuses` | Per-user-per-notification read flag (was JSON) |
| `20260114033812_AddNumberOfBranchesInHotelSubscription` | Tracks branches-bought for billing |
| `20260115031027_AddBillingAddressToInvoice` | Invoice `BillingAddress` JSON |
| `20260121032413_UpdateProfileSchema` | Profile fields refactor |
| `20260130024121_AddStripeEvents` | `StripeWebhookEvent` idempotency table |
| `20260206071150_AddStatuszFieldsToUser` | User status + reasons |
| `20260212021646_AddCarrierIntegrationTable` | `HotelBranchCarrierIntegration` |
| `20260212113650_AddHotelBranchCarrierSelection` | `HotelBranchCarrierSelection` |
| `20260212170408_AddAuditLogs` | `AuditLog` table |
| `20260213081314_AddSupportTickets` | `SupportTicket` table |

---

## 7. Seed Data

[Infrastructure/Data/SeedData/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Data/SeedData/) — applied on first migration / startup. Seeds reference catalogs: airports, shop offices, default subscription plans + features.

---

## 8. Wire Types (Frontend View)

In Angular, the data model is mirrored as TypeScript DAOs:

- `projects/web-b2b-hotel/src/app/core/daos/*.dao.d.ts` — wire-format types matching DTOs
- `projects/web-b2b-hotel/src/app/models/*.model.d.ts` — view-layer types
- `projects/web-b2b-hotel/src/app/shared/converters/` — DAO → Model converters

DAOs are kept thin (no class, no behavior) so JSON deserialization is just a cast.
