# Architecture

Takumo is a **B2B/B2C hotel-guest luggage delivery platform**. Hotel staff and travelers can book luggage pickup at a hotel and delivery to a destination (airport, another hotel, residence) through integrated shipping carriers.

The system is split across three repositories under [c:\Hong\Takumo](./):

| Repo | Stack | Role |
|---|---|---|
| [takumo-api/](../takumo-api/) | .NET 9 / ASP.NET Core / PostgreSQL | Backend REST API (single deployable serving both portals) |
| [takumo-ui/](../takumo-ui/) | Angular 21 / pnpm monorepo | Two SPAs: B2B hotel staff portal + B2C traveler app |
| [takumo-landing-page/](../takumo-landing-page/) | WordPress + custom theme | Public marketing site |

---

## 1. System Diagram

```
                    ┌────────────────────────────────┐
                    │  takumo-landing-page (WP)      │
                    │  Marketing · /contact · /news  │
                    └────────────┬───────────────────┘
                                 │ CTA links
                                 ▼
   ┌──────────────────────┐   ┌──────────────────────┐
   │ web-b2b-hotel SPA    │   │ web-b2c-traveler SPA │
   │ Hotel staff portal   │   │ Traveler app         │
   │ EN/JA, signals,      │   │ EN/JA, signals,      │
   │ Firebase realtime    │   │ Apple/Google Wallet  │
   └──────────┬───────────┘   └──────────┬───────────┘
              │       HTTPS/JWT          │
              ▼                          ▼
            ┌──────────────────────────────────┐
            │   takumo-api (ASP.NET Core 9)    │
            │   Clean Architecture · Swagger   │
            │   /api/v1/...                    │
            └─┬──────┬─────────┬───────┬───────┘
              │      │         │       │
              ▼      ▼         ▼       ▼
       ┌─────────┐ ┌─────┐ ┌──────────┐ ┌─────────────────────┐
       │Postgres │ │Redis│ │ Hangfire │ │  External services  │
       │   15    │ │  7  │ │(jobs in  │ │  Stripe · SendGrid  │
       │         │ │     │ │ Postgres)│ │  Firebase · AWS S3  │
       └─────────┘ └─────┘ └──────────┘ │  AWS SES · Google   │
                                        │  Wallet · Apple     │
                                        │  Wallet · reCAPTCHA │
                                        │  ShipAndCo · Yamato │
                                        │  B2Cloud · Yamato   │
                                        │  Tracking · PMS:    │
                                        │  Mews/Cloudbeds/    │
                                        │  Oracle             │
                                        └─────────────────────┘
```

Both SPAs talk to the same API. Roles in JWT distinguish hotel staff (Administrator, Receptionist) from travelers.

---

## 2. Backend Architecture — `takumo-api`

**Pattern:** Clean Architecture (Domain-driven, dependency-inverted layers). Solution: [Takumo.B2BHotel.slnx](../takumo-api/Takumo.B2BHotel.slnx).

### Projects

| Project | Type | Target | Role |
|---|---|---|---|
| [Takumo.B2BHotel.Domain](../takumo-api/src/Takumo.B2BHotel.Domain/) | Class library | net9.0 | Entities, value objects, enums, base classes. No external deps. |
| [Takumo.B2BHotel.Application](../takumo-api/src/Takumo.B2BHotel.Application/) | Class library | net9.0 | Services, DTOs, validators, mappers, interfaces. MediatR, FluentValidation, BCrypt, Stripe.net. |
| [Takumo.B2BHotel.Infrastructure](../takumo-api/src/Takumo.B2BHotel.Infrastructure/) | Class library | net9.0 | EF Core, repos, external integrations, Hangfire, AWS, Firebase, SendGrid, Google APIs, RazorLight, Polly. |
| [Takumo.B2BHotel.API](../takumo-api/src/Takumo.B2BHotel.API/) | ASP.NET Core Web API | net9.0 | HTTP entry. Controllers, middleware, Swagger, JWT bearer auth, Serilog, rate limiting. |
| [Takumo.Integrations.YamatoB2Cloud](../takumo-api/shared/Takumo.Integrations.YamatoB2Cloud/) | Class library | net9.0 | Yamato B2Cloud carrier client. |
| [Takumo.Integrations.YamatoTracking](../takumo-api/shared/Takumo.Integrations.YamatoTracking/) | Class library | net9.0 | Yamato tracking client. |

### Dependency Direction

```
   API ──────► Application ──────► Domain
    │              │                  ▲
    └─► Infrastructure ───────────────┘
            │
            └─► YamatoB2Cloud · YamatoTracking
```

Domain has no outbound references. Infrastructure implements interfaces declared in Application.

### Request Pipeline ([Program.cs](../takumo-api/src/Takumo.B2BHotel.API/Program.cs))

1. Serilog (structured logging → console + daily file)
2. Swagger (Development / Staging only)
3. HTTPS redirect
4. Static files
5. **Global exception handler** ([ExceptionHandler/GlobalExceptionHandler](../takumo-api/src/Takumo.B2BHotel.API/ExceptionHandler/))
6. CORS
7. Rate limiter
8. JWT authentication
9. Role-based authorization
10. Controllers
11. Recurring jobs registered: `SubscriptionJobInitializer`, `ShipmentJobInitializer`

### Cross-cutting

- **Auth:** JWT bearer (access 1h, refresh 7d) — see [JwtSettings](../takumo-api/src/Takumo.B2BHotel.API/appsettings.json) and [`[RequireAccessToken]`, `[RoleAuthorization]`](../takumo-api/src/Takumo.B2BHotel.API/Attributes/) attributes.
- **Background jobs:** Hangfire on PostgreSQL backend ([Infrastructure/BackgroundJobs/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/BackgroundJobs/)) — booking retries, shipment retries, subscription lifecycle, wallet pass updates, Yamato tracking poll.
- **Auditing:** `BaseAuditableEntity` + per-entity `IAuditable` + `AuditLog` table. User context resolved via `DbContextUserContext`.
- **Soft delete:** entities implementing `ISoftDelete` flagged via global query filters.
- **Multi-tenancy:** `IHotelScoped` / `IBranchScoped` marker interfaces enforce hotel/branch data isolation.

### Database

- **PostgreSQL 15**, EF Core 9 (`Npgsql.EntityFrameworkCore.PostgreSQL` 9.0.2)
- Single DbContext: [TakumoDbContext.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Data/TakumoDbContext.cs)
- 35+ entity configurations under [Data/Configurations/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Data/Configurations/)
- 87 migrations under [Migrations/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Migrations/) (initial = `20251126063159_InitialMigration`)
- Migrations run on startup is **off by default** (`EntityFramework.RunMigrationsOnStartup = false`); use [Dockerfile.migration](../takumo-api/Dockerfile.migration) or [scripts/migrate.sh](../takumo-api/scripts/migrate.sh)

---

## 3. Frontend Architecture — `takumo-ui`

**Monorepo** managed by pnpm workspaces. Angular 21.2, TypeScript 5.9 (strict), Node ≥ 24.1.

### Projects

| Project | Type | Role |
|---|---|---|
| [web-b2b-hotel](../takumo-ui/projects/web-b2b-hotel/) | App | Hotel staff portal (selector `b2b-hotel-root`) |
| [web-b2c-traveler](../takumo-ui/projects/web-b2c-traveler/) | App | Traveler-facing app (selector `b2c-traveler-root`) |
| [_common](../takumo-ui/projects/_common/) | Non-buildable | Interceptors, form bases, constants, utils. Import via `@takumo/common`. |
| [_widgets](../takumo-ui/projects/_widgets/) | ng-packagr library | UI kit (selector prefix `tkm-*`). Import via `@takumo/widgets`. |
| [_core](../takumo-ui/projects/_core/) | ng-packagr library | Auth, layout, profile, dialogs, app bootstrapper. Import via `@takumo/core`. |

### Patterns

- **Standalone components only** (no NgModules). Schematic default: `OnPush`, SCSS, tests skipped.
- **Routing:** lazy-loaded feature folders under `app/modules/`. Guards: `AuthenticatedGuard`, `AnonymousGuard`, `AdminGuard`.
- **State:** Angular Signals (`AuthStore` extends `AuthBaseStore` from `@takumo/core`); RxJS for HTTP; local component state for feature UI. No NgRx/Akita.
- **HTTP:** `provideHttpClient(withInterceptors([...]), withFetch())`. Interceptor chain: general headers → bearer token → Google Places headers → base API path → app-specific error → generic 401-refresh → error-code extraction.
- **DAO/Model split:** `core/daos/*.dao.d.ts` = wire format, `models/*.model.d.ts` = view-layer types, `shared/converters/` = mapping.
- **Forms:** Reactive only. Validators in [_widgets/validators](../takumo-ui/projects/_widgets/src/lib/validators/): `EmailValidator`, `MustMatchToValidator`, `MustDifferFromValidator`, `NonEmptyObjectValidator`.

### Runtime config

Each app reads `app/app-config.json` (committed template: `app-config.template.json`, real file gitignored) via `fetch()` before `bootstrapApplication`. Keys include `api.baseUrl`, `firebase.*`, `googleApiKey`, `recaptchaSiteKey`, `travelerUrl`, `termsVersion`, `privacyVersion`.

### i18n

Hybrid:
- **Build-time** (`$localize` / `i18n` attr) → `projects/<app>/locales/messages.json` + `messages.ja.json` → per-locale bundles under `/en/`, `/ja/`
- **Runtime** (custom `TranslateService` in `_widgets`) → `projects/<app>/public/translations/{en,ja}.json` → parameterized + ICU plural

Full detail: [takumo-ui/docs/i18n.md](../takumo-ui/docs/i18n.md).

### Build & deploy

- `pnpm build:<project>` per app, then `deployments/minify-translations.sh` and `deployments/hoist-global-assets.sh` post-process the output
- CI: [.github/workflows](../takumo-ui/.github/workflows/) → AWS S3 + CloudFront (OIDC, no static keys)

---

## 4. Marketing Site Architecture — `takumo-landing-page`

Standard WordPress install with **one custom theme** ([wp-content/themes/takumo-theme](../takumo-landing-page/wp-content/themes/takumo-theme/)). Plugins directory is intentionally empty in Git — ACF and Polylang are installed at the host.

### Theme highlights

- **Page-builder:** Gutenberg only, with **19 custom blocks** in `blocks/` (hero, in-page-nav, plans-features, news-cards, contact-form, partner-benefits, etc.). Block category: "Takumo Blocks".
- **Custom post types:** `news` (archived at `/news`), `faq` (with `faq_category` taxonomy).
- **PMS partnership contact form:** custom POST handler in [inc/pms-contact-form-handler.php](../takumo-landing-page/wp-content/themes/takumo-theme/inc/pms-contact-form-handler.php), separate from Contact Form 7.
- **Auto page creator** ([inc/auto-page-creator.php](../takumo-landing-page/wp-content/themes/takumo-theme/inc/auto-page-creator.php)): on first admin load, creates Home / About / Services / Contact / Privacy / Blog plus primary menu.
- **i18n:** Polylang (EN/JA). Japanese fonts: IBM Plex Sans JP, self-hosted.
- **Build:** Webpack 5 + Babel + Sass + PostCSS. FLOCSS naming under `assets/scss/`. Output to theme `dist/` (committed).
- **Deploy:** [.github/workflows/deploy-site.yaml](../takumo-landing-page/.github/workflows/deploy-site.yaml) → AWS EC2 via SSM, branches `main`→prod, `staging`→staging.

---

## 5. Cross-Repo Conventions

| Concern | Convention |
|---|---|
| Languages | English (default) + Japanese throughout (UI, theme, transactional emails) |
| Timezone | `Asia/Tokyo` (set in API via `DateTimeSettings.TimeZone`) |
| Line endings | LF only (enforced via `.gitattributes` + Prettier + Husky in `takumo-ui`) |
| Branching | `dev` → `staging` → tag `v*` (UI). `main`/`staging` (WP). API branching not codified in repo. |
| CI/CD | GitHub Actions in all three. OIDC for AWS, no long-lived keys. |

---

## 6. Where to Read Next

- **Request/data flows end-to-end:** [FLOWS.md](./FLOWS.md)
- **Per-module catalog:** [MODULES.md](./MODULES.md)
- **REST endpoints & auth:** [API.md](./API.md)
- **Entities, enums, migrations:** [DATA_MODEL.md](./DATA_MODEL.md)
- **All configuration keys, env vars, secrets:** [CONFIG.md](./CONFIG.md)
- **Coding standards across the three stacks:** [CONVENTIONS.md](./CONVENTIONS.md)
- **Business rules & invariants:** [BUSINESS_RULES.md](./BUSINESS_RULES.md)
- **Test strategy & coverage:** [TESTING.md](./TESTING.md)
- **Local environment from zero:** [SETUP.md](./SETUP.md)
