# Modules

Per-repo catalog of code units. Use this as the directory of "where does X live."

---

## Backend — `takumo-api`

### `src/Takumo.B2BHotel.Domain/` — pure model

| Folder | Contents |
|---|---|
| [Entities/](../takumo-api/src/Takumo.B2BHotel.Domain/Entities/) | 35 entities (see [DATA_MODEL.md](./DATA_MODEL.md)) |
| [ValueObjects/](../takumo-api/src/Takumo.B2BHotel.Domain/ValueObjects/) | `Address`, `PickupInfo`, `DestinationInfo`, `LuggageInfoObject`, `AirportCounter`, `ActivationReason`, `DeactivationReason`, `DeletionReason` |
| [Enums/](../takumo-api/src/Takumo.B2BHotel.Domain/Enums/) | 33 enums incl. `BookingStatus`, `PmsProvider`, `ShippingCarrier`, `UserRole`, `SubscriptionStatus`, `LuggageSize`, `PortalType` |
| [Common/](../takumo-api/src/Takumo.B2BHotel.Domain/Common/) | `BaseEntity`, `BaseAuditableEntity`, `IBaseAuditableEntity`, `IAuditable`, `ISoftDelete`, `IHotelScoped`, `IBranchScoped`, `EnumCodeAttribute`, `AuditIgnoreAttribute` |

### `src/Takumo.B2BHotel.Application/` — use cases

| Folder | Contents |
|---|---|
| [Services/](../takumo-api/src/Takumo.B2BHotel.Application/Services/) | One folder per service, each with `I*Service.cs` + `*Service.cs`. 26 services — see breakdown below. |
| [DTOs/](../takumo-api/src/Takumo.B2BHotel.Application/DTOs/) | Request/response shapes per controller |
| [Validators/](../takumo-api/src/Takumo.B2BHotel.Application/Validators/) | FluentValidation rules (per DTO) |
| [Mappers/](../takumo-api/src/Takumo.B2BHotel.Application/Mappers/) | Entity ↔ DTO mapping |
| [Interfaces/](../takumo-api/src/Takumo.B2BHotel.Application/Interfaces/) | Abstractions implemented in Infrastructure (e.g. `IEmailService`, `IShipmentService`, `IFirebaseNotificationSender`) |
| [Settings/](../takumo-api/src/Takumo.B2BHotel.Application/Settings/) | Strongly-typed `*Settings` POCOs bound from `appsettings.json` |
| [Models/](../takumo-api/src/Takumo.B2BHotel.Application/Models/) | App-layer types (paging, results, etc.) |
| [Extensions/](../takumo-api/src/Takumo.B2BHotel.Application/Extensions/) | `AddApplicationServices(IConfiguration)` DI extension |

**Service list:**

| Service | Purpose |
|---|---|
| `AddressService` | Address validation + Google Places lookup wrapper |
| `AuditLogService` | Read audit history |
| `BookingService` | Booking CRUD, status transitions |
| `CheckoutService` | Stripe checkout session creation |
| `FeatureService` | Feature-flag lookup per subscription plan |
| `GuestInfoService` | Local guest record management |
| `GuestPassService` | Issue/manage `GuestPass` (Wallet-bound) |
| `HotelBranchService` | Branch CRUD, pickup-time-zone config |
| `HotelCarrierIntegrationService` | Per-hotel carrier credentials |
| `HotelCarrierSelectionService` | Branch-level carrier choice |
| `HotelPmsIntegrationService` | PMS credentials per hotel |
| `HotelService` | Hotel CRUD |
| `HotelSubscriptionService` | Subscription lifecycle |
| `InvoiceService` | Invoice generation, billing-address attach |
| `LegalDocumentService` | Terms/privacy versions and consent |
| `NotificationService` | Persist + emit notifications |
| `PaymentMethodService` | Stripe payment-method CRUD |
| `SearchGuestService` | Cross-PMS guest lookup |
| `ShipmentService` | Shipment retry orchestration |
| `ShippingRateService` | Carrier rate quotes |
| `StaffUserService` | Hotel staff CRUD, password reset |
| `SubscriptionPlanService` | Public plan catalog |
| `SupportTicketService` | Support ticket CRUD |
| `TravelerService` | Traveler user CRUD, sign-up |
| `VerificationTokenService` | Issue/consume email-verification & password-reset tokens |
| `YamatoB2ShippingService` | Yamato B2 wrapper used by ShipmentService |
| `YamatoTracking` | Yamato tracking poll helpers |

### `src/Takumo.B2BHotel.Infrastructure/` — adapters

| Folder | Contents |
|---|---|
| [Data/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Data/) | `TakumoDbContext`, `TakumoModelCacheKeyFactory`, [Configurations/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Data/Configurations/) (35+ `IEntityTypeConfiguration`s), [Identity/DbContextUserContext](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Data/Identity/), [Interceptors/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Data/Interceptors/) (audit, soft-delete), [SeedData/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Data/SeedData/) |
| [Migrations/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Migrations/) | 87 EF migrations |
| [ExternalServices/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/ExternalServices/) | `FirebaseNotificationSender`, `FirebaseRealtimeDatabase`, `GoogleAddressService`, `GoogleAuthService`, `RecaptchaService`, `S3FileUploadService`, `SendGridEmailService`, `ShipAndCoMonitorService`, `ShipAndCoService`, `ShipmentLabelUploader`, `StripeService`, `YamatoTrackingMonitorService`, [WalletPass/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/ExternalServices/WalletPass/) (`ApnsPushNotificationService`, `ApplePassRegistrationService`, `AppleWalletServiceV2`, `GoogleWalletServiceV2`) |
| [BackgroundJobs/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/BackgroundJobs/) | `BookingBackgroundJobService`, `ShipmentRetryBackgroundJobService`, `SubscriptionBackgroundJobService`, `UpdateWalletPassBackgroundJobService`, `YamatoTrackingPollJob` |
| [Strategies/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Strategies/) | Strategy + Factory for PMS guest search (Mews / Cloudbeds / Oracle), carrier shipment creation (ShipAndCo / YamatoB2Cloud), connection-test (ShipAndCo / YamatoB2Cloud), Stripe event handler |
| [Webhooks/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Webhooks/) | Stripe event handlers: `CheckoutSessionCompletedHandler`, `InvoicePaymentSucceededHandler`, `InvoicePaymentFailedHandler`, `SubscriptionUpdatedHandler`, `SubscriptionExpiringSoonHandler`, `TrialWillEndHandler` |
| [Services/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Services/) | Infra-side service implementations (e.g. concrete repos that don't fit external) |
| [Queries/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Queries/) | Raw/optimized read queries |
| [Templates/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Templates/) | RazorLight email templates |
| [Extensions/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Extensions/) | `AddInfrastructureServices`, `EnsureDbMigrated`, `UseFireBase` |

### `src/Takumo.B2BHotel.API/` — HTTP host

| Folder | Contents |
|---|---|
| [Controllers/](../takumo-api/src/Takumo.B2BHotel.API/Controllers/) | 22 root + 4 `Admin/` + 5 `Traveler/` controllers (see [API.md](./API.md)) |
| [Attributes/](../takumo-api/src/Takumo.B2BHotel.API/Attributes/) | `RoleAuthorization`, `RequireAccessToken` |
| [Constants/](../takumo-api/src/Takumo.B2BHotel.API/Constants/) | `ApiConstants` (`BaseRoute = "api"`, `VersionV1`) |
| [ExceptionHandler/](../takumo-api/src/Takumo.B2BHotel.API/ExceptionHandler/) | `GlobalExceptionHandler` → ProblemDetails |
| [Filters/](../takumo-api/src/Takumo.B2BHotel.API/Filters/) | Swagger filters: `AccessTokenOperationFilter`, `AuthorizeCheckOperationFilter`, `ErrorCodeDocumentFilter` |
| [Extentions/](../takumo-api/src/Takumo.B2BHotel.API/Extentions/) | `ServiceCollectionExtentions`, `RateLimitingExtensions`, `CustomEnumModelBinder` |
| [Helpers/](../takumo-api/src/Takumo.B2BHotel.API/Helpers/) | `PropertyPathHelper` |
| [Configs/](../takumo-api/src/Takumo.B2BHotel.API/Configs/) | Wallet certificates, Firebase credentials (gitignored), Apple WWDR CA |
| `Program.cs` | Composition root |
| `appsettings.json` | Public config — see [CONFIG.md](./CONFIG.md) |

### `shared/`

| Project | Contents |
|---|---|
| [Takumo.Integrations.YamatoB2Cloud](../takumo-api/shared/Takumo.Integrations.YamatoB2Cloud/) | Yamato B2 Cloud REST client + DTOs + options |
| [Takumo.Integrations.YamatoTracking](../takumo-api/shared/Takumo.Integrations.YamatoTracking/) | Yamato tracking REST client + DTOs + options |

---

## Frontend — `takumo-ui`

### Apps

#### [web-b2b-hotel](../takumo-ui/projects/web-b2b-hotel/) — hotel staff portal

Feature modules under [src/app/modules/](../takumo-ui/projects/web-b2b-hotel/src/app/modules/):

| Module | Path | Guard(s) | Purpose |
|---|---|---|---|
| `_auth` | `/auth` | `AnonymousGuard` | Login, register, forgot/reset password, verify email |
| `dashboard` | `/dashboard` | `AuthenticatedGuard` | Landing post-login: today's bookings, notifications |
| `bookings` | `/bookings` | `AuthenticatedGuard` | List, detail, create, edit bookings |
| `users` | `/users` | `AuthenticatedGuard` + `AdminGuard` | Staff CRUD |
| `settings` | `/settings` | `AuthenticatedGuard` + `AdminGuard` | Hotel-wide settings, subscription, billing |
| `profile` | `/profile` | `AuthenticatedGuard` | Self-edit |
| `pms-configurations` | `/pms` | `AuthenticatedGuard` + `AdminGuard` | PMS credential mgmt (Mews/Cloudbeds/Oracle) |
| `carrier-configurations` | `/carriers` | `AuthenticatedGuard` + `AdminGuard` | Carrier credentials, branch-level selection |
| `notifications` | `/notifications` | `AuthenticatedGuard` | Notification center |
| `help` | `/help` | `AuthenticatedGuard` | Help docs, support tickets |

Local infrastructure under [src/app/](../takumo-ui/projects/web-b2b-hotel/src/app/):
- `core/apis/` — 14 API services (see [API.md](./API.md))
- `core/daos/` — wire-format types
- `core/guards/` — `AuthenticatedGuard`, `AnonymousGuard`, `AdminGuard`
- `core/interceptors/` — `b2bHotelHttpErrorInterceptor`
- `core/tokens/` — DI tokens for runtime config
- `stores/` — `AuthStore` (signal-based)
- `dialogs/`, `drawers/`, `services/`, `shared/`, `models/`

#### [web-b2c-traveler](../takumo-ui/projects/web-b2c-traveler/) — traveler app

| Module | Path | Guard(s) | Purpose |
|---|---|---|---|
| `_auth` | `/auth` | `AnonymousGuard` | Login, register, forgot/reset, verify email |
| `dashboard` | `/dashboard` | none | Public landing |
| `shipment` | `/shipment` | none | Track shipment by ID |
| `bookings` | `/bookings` | `AuthenticatedGuard` | Own bookings |
| `pass` | `/pass` | none | Guest pass view |
| `profile` | `/profile` | `AuthenticatedGuard` | Profile edit |

### Libraries

#### [_common](../takumo-ui/projects/_common/) — `@takumo/common`

Non-buildable, included via tsconfig paths. Exports:
- Interceptors: `generalHeadersInterceptor`, `baseApiPathInterceptorFactoryFunc`, `googlePlacesAPIHeadersInterceptorFactoryFunc`
- Form base directives: `BaseFormGroupControlDirective`, `BaseStandaloneControlDirective`
- Constants: date formats, locale, pagination defaults, regex (`PasswordRegex`, `PhoneNumberRegex`, `UUIDV4Regex`), storage key prefixes
- DI tokens: `IsFormDataContentToken`
- Utils: date-time, string helpers

#### [_widgets](../takumo-ui/projects/_widgets/) — `@takumo/widgets`

ng-packagr library, selector prefix `tkm-*`. Categories:
- Buttons (`tkm-button`, toggle, link)
- Form fields (`tkm-float-form-field`, checkbox, radio, select, password, google-places-autocomplete, textarea)
- Data (`tkm-table`, data-grid, data-list)
- Pagination, menus (dropdown, context menu)
- Overlays (`tkm-dialog`, drawer, popover, tooltip)
- Panels (card, accordion, tabs)
- Typography, loading, info (alert, badge, tag)
- Custom validators: `EmailValidator`, `MustMatchToValidator`, `MustDifferFromValidator`, `NonEmptyObjectValidator`
- Custom `TranslateService` + `TranslatePipe` (runtime i18n) — see [takumo-ui/docs/i18n.md](../takumo-ui/docs/i18n.md)
- `LanguageSwitcherComponent`

#### [_core](../takumo-ui/projects/_core/) — `@takumo/core` + `@takumo/core/static`

ng-packagr library, shared app behaviors.

| Category | Exports |
|---|---|
| Bootstrapper | `bootstrapApp`, `createAppInitializer`, `createFetchUserInfo` |
| Auth components | `LoginComponent`, `ForgotPasswordComponent`, `ResetPasswordComponent`, `VerifyEmailComponent`, `PasswordComplexityComponent` |
| Auth services | `LoginBaseService`, `SignUpBaseService`, `ForgotPasswordBaseService`, `ResetPasswordBaseService`, `VerifyEmailBaseService`, `ProfileBaseService` |
| Store | `AuthBaseStore<TUser>` (signal-based) |
| Layouts | `ShellLayoutComponent` (+ header & sidebar), `LandingLayoutComponent`, `SplitLayoutComponent`, `DialogLayoutComponent` |
| Form controls | `AddressGroupFormComponent`, `GuestInfoFormComponent`, `LuggageInfoFormComponent`, `DestinationInfoFormComponent`, `PickupInfoFormComponent` |
| Dialogs | `InformationComponent`, `OperationConfirmationComponent`, `DialogAbstractionBaseDirective` |
| Interceptors | `bearerTokenAuthHeaderInterceptorFactoryFunc`, `httpErrorInterceptorFactoryFunc`, `pickingErrorCodeInterceptor` |
| Base directives | `LocalizedPageBaseDirective`, `ProfileBaseDirective`, `SignUpBaseDirective`, `CanStepThroughDirective`, `HasLanguageSwitcherBaseDirective` |
| Factories | `createAuthChildRoutes` |
| Static (separate entry) | NotFound 404 page via `@takumo/core/static` |

### Tooling folders

| Folder | Contents |
|---|---|
| [development/](../takumo-ui/development/) | `proxy.conf.base.js`, `proxy.conf.js`, `proxy.conf.ja.js` (dev proxy with locale rewrite) |
| [deployments/](../takumo-ui/deployments/) | `extract-config.sh`, `hoist-global-assets.sh`, `minify-translations.sh`, `sort-translation-keys.sh`, `validate-translations.sh`, `lambda/` (CloudFront edge rewrites) |
| [.github/workflows/](../takumo-ui/.github/workflows/) | `pr-checks.yml`, `deploy-to-s3.yml`, `deploy-to-dev-manually.yml` |
| [docs/](../takumo-ui/docs/) | [development-flow.md](../takumo-ui/docs/development-flow.md), [i18n.md](../takumo-ui/docs/i18n.md) |

---

## Marketing — `takumo-landing-page`

Only the custom theme is in scope; WP core is vendored at host.

### [wp-content/themes/takumo-theme](../takumo-landing-page/wp-content/themes/takumo-theme/)

| Folder | Contents |
|---|---|
| `*.php` (root) | Template files: `front-page.php`, `page-home.php`, `page-news.php`, `single-news.php`, `archive-news.php`, `template-pms-partnerships.php`, `template-text-page.php`, `header.php`, `header-partners.php`, `footer.php`, `footer-partners.php` |
| `functions.php` | Theme bootstrap: `after_setup_theme`, enqueue, CPT registration (`faq`, `news`), menus, ACF dependency |
| `inc/` | 11 helper modules: `blocks.php` (register 19 custom blocks), `pms-contact-form-handler.php`, `auto-page-creator.php`, `general-settings.php` (ACF options page), `customizer.php`, `template-functions.php`, `template-tags.php`, `responsive-images.php`, `image-helpers.php`, `alt-text-manager.php`, `custom-css-handler.php` |
| `blocks/` | 19 custom Gutenberg block source dirs — each with `block.json` + `render.php`/`index.php` + optional React |
| `assets/scss/` | FLOCSS: `foundation/`, `layout/`, `object/{component,project,utility}/`, `blocks/` |
| `assets/js/` | `main.js`, `customizer.js`, `cf7-redirect.js`, `blocks/` |
| `assets/fonts/` | Brandon Grotesque, IBM Plex Sans (EN), IBM Plex Sans JP |
| `assets/images/` | `hero/`, `icons/`, `pagination/`, `products/` |
| `dist/` | Webpack output (committed) |
| `scripts/` | `optimize-images.js`, `fetch-material-symbols.js` |
| `webpack.config.js`, `package.json`, `.stylelintrc`, `.eslintrc` | Build & lint |

### CI

| File | Purpose |
|---|---|
| [.github/workflows/deploy-site.yaml](../takumo-landing-page/.github/workflows/deploy-site.yaml) | Push `main`→prod, `staging`→staging; AWS OIDC → SSM RunShellScript → EC2 |
