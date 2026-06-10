# Conventions

Coding standards across the three repos. The frontend monorepo has the strictest published rules (see also [takumo-ui/docs/development-flow.md](../takumo-ui/docs/development-flow.md)).

---

## 1. Cross-cutting

| Concern | Rule |
|---|---|
| **Line endings** | LF only, enforced via `.gitattributes` + `.editorconfig` + Prettier + Husky pre-commit. Windows devs: `git config --global core.autocrlf input` |
| **Languages in product** | English (default) + Japanese (`ja`). All user-facing text must support both. |
| **Timezone for stored times** | UTC in DB (`CreatedAtUtc`, `LastModifiedAtUtc`, `PaidAt`). Display in `Asia/Tokyo` (configured in `DateTimeSettings.TimeZone`) |
| **Commits** | Short, imperative subject. No mandated format (no conventional-commits enforced). |
| **Branching** | UI: `dev` → `staging` → tag `v*` (prod). WP: push `main` → prod, `staging` → staging. API: not codified. |
| **Secrets** | Never committed. Reference `app-config.template.json` (UI), `Configs/` gitignore (API), `wp-config-sample.php` (WP). |

---

## 2. Backend — `.NET` (`takumo-api`)

### 2.1 Project layout
- **Clean Architecture** layering: `Domain ← Application ← Infrastructure ← API`. Domain has no outbound deps. Infrastructure implements interfaces declared in Application.
- One project per layer; **no horizontal subdivision by feature** — feature code is grouped by layer (controllers under API, services under Application/Services, entities under Domain/Entities).
- Service folders follow `Services/<Name>Service/I<Name>Service.cs + <Name>Service.cs` pattern.

### 2.2 Naming
| Item | Convention | Example |
|---|---|---|
| File | PascalCase, one type per file | `BookingService.cs` |
| Class | PascalCase | `BookingService` |
| Interface | `I` prefix | `IBookingService` |
| Async method | `Async` suffix | `CreateAsync` |
| DTO | `<Verb><Noun>Request` / `Response` | `CreateBookingRequest`, `GetBookingResponse` |
| Entity | Singular noun | `Booking`, `HotelBranch` |
| Enum | Singular noun | `BookingStatus` |
| Settings POCO | `<Name>Settings` | `JwtSettings`, `StripeSettings` |
| Cron / job | `<Name>Job` / `<Name>BackgroundJobService` | `YamatoTrackingPollJob` |
| Strategy + Factory | `<Provider><Operation>Strategy` + `*StrategyFactory` | `MewsGuestSearchStrategy`, `PmsGuestSearchStrategyFactory` |

### 2.3 Code style
- `TreatWarningsAsErrors = true` ([Takumo.B2BHotel.API.csproj](../takumo-api/src/Takumo.B2BHotel.API/Takumo.B2BHotel.API.csproj))
- `Nullable = enable` everywhere — use `?` and `required` instead of nullable annotations in docs
- `ImplicitUsings = enable`
- `GenerateDocumentationFile = true` (for Swagger) — public APIs need XML doc comments where they materially clarify behavior
- `NoWarn=1591` suppresses "missing XML comment" so document only what matters
- **Async**: every I/O method is `Task`/`Task<T>`, accepts `CancellationToken cancellationToken = default` as last parameter, forwards to inner calls.
- **Errors**: throw domain-specific exceptions from Application/Domain; `GlobalExceptionHandler` translates to ProblemDetails. Use FluentValidation for request-shape validation.
- **DI**: register in `AddApplicationServices(IConfiguration)` / `AddInfrastructureServices(IConfiguration)`. Scopes: services scoped, options singleton, factories singleton, EF context scoped.

### 2.4 Patterns
- **Strategy + Factory** for variation: PMS search ([PmsGuestSearchStrategyFactory.cs](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Strategies/PmsGuestSearchStrategyFactory.cs)), carrier shipment creation, carrier connection test, Stripe event dispatch.
- **Webhook handlers** are individual classes inheriting `BaseStripeWebhookHandler` ([Webhooks/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Webhooks/)); dispatcher resolves them by event type.
- **Background jobs** are Hangfire methods; `JobInitializer.ConfigureRecurringJobs(IServiceProvider)` registers schedules at startup ([Program.cs](../takumo-api/src/Takumo.B2BHotel.API/Program.cs)).
- **Soft delete** via `ISoftDelete` + global query filter. Do not write `WHERE NOT IsDeleted` manually.
- **Multi-tenancy** enforced at service layer using `IUserContext` (current JWT's `HotelId`/`HotelBranchId`). Entities tagged `IHotelScoped` / `IBranchScoped` must be filtered on read.

### 2.5 Database
- Migrations are PR-author's responsibility: `dotnet ef migrations add <Name> --project src/Takumo.B2BHotel.Infrastructure --startup-project src/Takumo.B2BHotel.API`
- Name migrations as `<Verb><Subject>` (`AddCarrierIntegrationTable`, `UpdateProfileSchema`)
- One logical change per migration
- `RunMigrationsOnStartup = false` — production deploys must run migrations explicitly (`Dockerfile.migration` or `scripts/migrate.sh`)

---

## 3. Frontend — Angular (`takumo-ui`)

Authoritative source: [takumo-ui/docs/development-flow.md](../takumo-ui/docs/development-flow.md).

### 3.1 Toolchain
| Tool | Version |
|---|---|
| Node | ≥ 24.1.0 (enforced via `engines`) |
| pnpm | ≥ 10.11.0 (enforced) |
| Angular | 21.2 |
| TypeScript | 5.9 (strict) |

### 3.2 Naming
| Item | Convention | Example |
|---|---|---|
| File | kebab-case | `booking-details.component.ts` |
| Class | PascalCase | `BookingDetailsComponent` |
| Component selector (B2B) | `b2b-hotel-*` | `b2b-hotel-dashboard` |
| Component selector (B2C) | `b2c-traveler-*` | `b2c-traveler-booking` |
| Widget selector (`_widgets`) | `tkm-*` | `tkm-button`, `tkm-dialog` |
| DAO | `*.dao.d.ts` | `booking-details.dao.d.ts` |
| Model | `*.model.d.ts` | `booking-details.model.d.ts` |
| Signal property | suffix `$` | `userInfo$`, `isLoading$` |
| Constant | UPPER_SNAKE_CASE | `PAGINATION_DEFAULT` |

### 3.3 Component defaults (schematic-enforced)
- **Standalone** components (no NgModules)
- `OnPush` change detection
- SCSS styles
- Tests skipped by schematic (`skipTests: true` in [angular.json](../takumo-ui/angular.json))

### 3.4 Code style
- **Prettier 3.8** — `printWidth: 120`, single quotes, tab 2, `arrowParens: avoid`, `trailingComma: none`, LF
- HTML parser: Angular (template-aware)
- TS: `strict`, `noImplicitOverride`, `noPropertyAccessFromIndexSignature`, `noImplicitReturns`, `noFallthroughCasesInSwitch`, target `ES2024`
- Angular compiler: `strictTemplates`, `strictInjectionParameters`, `strictInputAccessModifiers`
- **Use Angular CLI** for all new artifacts (`ng generate component …`) — do not hand-roll

### 3.5 Architecture
- **Lazy load** every feature module: `loadChildren: () => import('./modules/x/x.routes').then(m => m.routes)`
- **DAO vs Model split** mandatory — DAOs are wire shapes (TS `.d.ts` only, no class), Models are view types, `shared/converters/` does the mapping
- **Stores are signal-based**, scope global to the app: extend `AuthBaseStore` from `@takumo/core` rather than rolling your own
- **Forms = Reactive only**. Custom validators live in `_widgets/validators/`.
- Selector prefixes prevent collision across the two apps + library

### 3.6 i18n
Hybrid — full detail in [takumo-ui/docs/i18n.md](../takumo-ui/docs/i18n.md).
- Static UI: `$localize` or `i18n` attribute → `projects/<app>/locales/messages*.json`
- Dynamic content: `| translate` pipe / `TranslateService.instant()` → `projects/<app>/public/translations/{en,ja}.json`
- Always provide both EN + JA. The pre-push hook (`pnpm translate:validate`) rejects pushes with missing or orphaned keys.
- Sort keys with `pnpm translate:sort-keys`

### 3.7 Git hooks (Husky)
- **pre-commit**: rejects CRLF
- **pre-push**: `pnpm format:ci && pnpm translate:validate`

### 3.8 Path aliases ([tsconfig.json](../takumo-ui/tsconfig.json))
- `@takumo/common/*` → `projects/_common/*`
- `@takumo/widgets` → `projects/_widgets/public-api.ts`
- `@takumo/core` → `projects/_core/src/public-api.ts`
- `@takumo/core/static` → `projects/_core/src/static/static.ts`
- `@b2b2-hotel/*` → `projects/web-b2b-hotel/src/app/*`
- `@b2c2-traveler/*` → `projects/web-b2c-traveler/src/app/*`

---

## 4. Marketing — WordPress (`takumo-landing-page`)

### 4.1 Theme conventions
- All theme code under [wp-content/themes/takumo-theme/](../takumo-landing-page/wp-content/themes/takumo-theme/) — no other theme should be active
- Helper modules under `inc/` — one concern per file, included via `functions.php`
- **Gutenberg only** — no Elementor / Divi / WPBakery. New page content must be a custom block in `blocks/` or use existing ones.
- Custom blocks register via `block.json` (WP block API v2) and a `render.php` (server-side rendered)

### 4.2 SCSS — FLOCSS
- `foundation/` — reset, variables, mixins
- `layout/` — header, footer, main containers
- `object/component/` — small reusable pieces
- `object/project/` — page-specific sections
- `object/utility/` — single-purpose classes
- `blocks/` — per-block styles, loaded with the block

### 4.3 JS / Build
- Webpack 5 + Babel + Sass + PostCSS
- Source under `assets/`, output to `dist/` (committed so deploy doesn't need a build step on EC2)
- ESLint + Prettier + stylelint (standard SCSS config)
- Run `npm run build` from the theme dir before pushing

### 4.4 PHP
- Match WordPress coding style (Yoda conditions optional; spaces inside parentheses for function calls)
- Text domain: `takumo-theme` on every `__()` / `_e()` / `_n()` call
- Always escape output: `esc_html()`, `esc_attr()`, `esc_url()`, `wp_kses()`

### 4.5 i18n
- Polylang for EN/JA. Per-page translations created in WP admin.
- Theme strings translated via `__()` with `takumo-theme` text domain; `.po`/`.mo` files in `wp-content/languages/themes/` (not in repo).
- Japanese typography via locally hosted IBM Plex Sans JP font files — do not link to Google Fonts.

### 4.6 Plugin policy
- Plugins are NOT versioned in this repo. ACF and Polylang are installed at the host and configured manually.
- Custom forms use the in-house [pms-contact-form-handler.php](../takumo-landing-page/wp-content/themes/takumo-theme/inc/pms-contact-form-handler.php) — prefer extending it over adding Contact Form 7 unless you need its features.

---

## 5. PR & Review Norms

| Repo | Pre-merge requirements |
|---|---|
| takumo-api | Build green, migrations included if schema changed, Swagger updated implicitly |
| takumo-ui | `pr-checks.yml` must pass: Prettier format, translation completeness, build for all apps |
| takumo-landing-page | Theme rebuild committed if SCSS/JS changed; manual smoke test |

Across all three: write a focused PR (one feature / one fix), keep diffs reviewable.
