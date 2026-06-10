# Setup

End-to-end local environment. Tested order: backend dependencies → backend API → frontend SPAs → marketing site.

---

## 1. Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| **Docker Desktop** | recent | Postgres, Redis, optional pgAdmin |
| **.NET SDK** | 9.0 (project targets `net9.0`) | API build & run |
| **Node.js** | ≥ **24.1.0** | UI build & dev server |
| **pnpm** | ≥ **10.11.0** | UI package manager |
| **PHP** | 7.4+ (only for landing-page work) | WordPress runtime |
| **Git** | recent | All repos |
| **PowerShell** (Windows) or **bash** (macOS / Linux / WSL) | — | Scripts ship in both flavors |

**Windows-specific git config** (avoid CRLF chaos):
```bash
git config --global core.autocrlf input
```

The Angular `engines` field will refuse `pnpm install` if Node or pnpm is older than required.

---

## 2. Repo Layout (after cloning)

You should end up with:

```
c:/Hong/Takumo/
├── takumo-api/                # .NET backend
├── takumo-ui/                 # Angular monorepo
├── takumo-landing-page/       # WordPress install
└── docs/                      # ← this set
```

If you don't yet have them, clone:
```bash
git clone https://github.com/Softint/takumo-api.git
git clone https://github.com/Softint/takumo-ui.git
git clone https://github.com/Softint/takumo-landing-page.git
```

---

## 3. Backend — `takumo-api`

### 3.1 Bring up dependencies (Postgres + Redis + pgAdmin)

```powershell
cd c:/Hong/Takumo/takumo-api
./scripts/start-dependencies.ps1
# or
bash ./scripts/start-dependencies.sh
```

This runs `docker-compose -f docker-compose.yml -f docker-compose.dependencies.yml up -d postgres redis pgadmin`.

Default ports/credentials:

| Service | URL / Port | Credentials |
|---|---|---|
| PostgreSQL | `localhost:5432` | `takumo_user` / `dev_password` / db `takumo_b2b_hotel_dev` |
| Redis | `localhost:6379` | none |
| pgAdmin | http://localhost:8080 | `admin@takumo.com` / `admin123` |

### 3.2 Apply database schema

The `db-init` service (in [docker-compose.dev.yml](../takumo-api/docker-compose.dev.yml)) applies [database/takumo_schema.sql](../takumo-api/database/takumo_schema.sql) if the `takumo` schema doesn't yet exist:

```powershell
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up --no-deps db-init
```

### 3.3 Run EF Core migrations

```powershell
./scripts/migrate.ps1
# or
bash ./scripts/migrate.sh
```

This runs the migration container (`Dockerfile.migration`). To apply migrations manually from the host:

```powershell
dotnet ef database update --project src/Takumo.B2BHotel.Infrastructure --startup-project src/Takumo.B2BHotel.API
```

### 3.4 Secrets / Configs

Create the gitignored config files referenced in `appsettings.json`:

```
src/Takumo.B2BHotel.API/Configs/
├── firebase-credentials.json          # Firebase service account
├── AppleWalletConfigs/
│   ├── icon.png  (+ logo.png)
│   ├── pass-cert.p12  (Apple Pass signing)
│   └── apns_auth_key.p8 (APNs auth)
└── google-wallet-service-account.json
```

Set secrets via environment variables (or `dotnet user-secrets init` for local dev):

```powershell
$env:JwtSettings__Key = "<random 32+ chars>"
$env:StripeSettings__SecretKey = "sk_test_..."
$env:StripeSettings__WebhookSecret = "whsec_..."
$env:RecaptchaSettings__SecretKey = "..."
$env:SendGrid__ApiKey = "..."
$env:AwsSesSettings__AccessKey = "..."
$env:AwsSesSettings__SecretKey = "..."
$env:YamatoB2Settings__AccessToken = "..."
```

See [CONFIG.md](./CONFIG.md) for the full list and where each is consumed.

### 3.5 Run the API

```powershell
./scripts/run-local-api.ps1
# or
bash ./scripts/run-local-api.sh
```

The script sets the connection string env vars, then runs `dotnet run --urls "http://localhost:5000;https://localhost:5001"` from the API project.

Verify:
- API: http://localhost:5000
- Swagger: http://localhost:5000/swagger or https://localhost:5001/swagger
- Health: http://localhost:5000/api/health

### 3.6 One-shot full bring-up

```powershell
./scripts/start-dev.ps1
```

This composes dependencies + db-init + migrations + API into one Docker stack on `localhost:8080` (mapped to API container's 8080).

---

## 4. Frontend — `takumo-ui`

### 4.1 Install

```powershell
cd c:/Hong/Takumo/takumo-ui
pnpm install
```

The `postinstall` script wires up Husky hooks automatically.

### 4.2 Create runtime config

For each app you want to run, copy the template and fill in real values:

```powershell
# B2B Hotel
copy projects/web-b2b-hotel/src/app/app-config.template.json `
     projects/web-b2b-hotel/src/app/app-config.json

# B2C Traveler
copy projects/web-b2c-traveler/src/app/app-config.template.json `
     projects/web-b2c-traveler/src/app/app-config.json
```

Edit each file. Minimum required:
- `api.baseUrl` → `http://localhost:5000/api` (matches the local backend; the `baseApiPathInterceptor` will prepend this)
- `googleApiKey`, `recaptchaSiteKey` → use test/dev keys
- B2B-only: `firebase.*` block → Firebase test project credentials, `travelerUrl` → `http://localhost:4200` (or whichever port the traveler app runs on)
- `termsVersion`, `privacyVersion` → any string for local

> **Never commit `app-config.json`.** It is in `.gitignore`. See [takumo-ui/README.md §Getting started](../takumo-ui/README.md#getting-started).

### 4.3 Start dev server

```bash
pnpm dev                    # B2B Hotel (en), http://localhost:4200
pnpm dev:hotel              # same as above
pnpm dev:hotel:ja           # B2B Hotel (ja)
pnpm dev:traveler           # B2C Traveler (en)
pnpm dev:traveler:ja        # B2C Traveler (ja)
```

The dev proxy (`development/proxy.conf.js` for EN, `proxy.conf.ja.js` for JA) rewrites `/global/*` so locale-specific assets resolve correctly.

### 4.4 Build for production

```bash
pnpm build                       # all projects
pnpm build:web-b2b-hotel
pnpm build:web-b2c-traveler
```

Outputs to `dist/<app>/browser/{en,ja}/`. Post-build, [deployments/minify-translations.sh](../takumo-ui/deployments/minify-translations.sh) and [deployments/hoist-global-assets.sh](../takumo-ui/deployments/hoist-global-assets.sh) trim translation JSON and deduplicate `global/`.

### 4.5 Translation workflow

```bash
pnpm translate:extract          # update messages.json from $localize tags
pnpm translate:sort-keys        # sort runtime translation keys
pnpm translate:validate         # fails on missing or orphaned keys
```

The pre-push hook runs `pnpm format:ci && pnpm translate:validate` automatically.

Full i18n details: [takumo-ui/docs/i18n.md](../takumo-ui/docs/i18n.md).

---

## 5. Marketing — `takumo-landing-page`

### 5.1 WordPress host

The repo is a full WP install **minus** secrets, plugins, and uploads (per `.gitignore`). You need a local WP host:

- **Recommended for local dev:** [Local by Flywheel](https://localwp.com/), [Lando](https://lando.dev/), or [DDEV](https://ddev.com/)
- Set the site root to the repo dir (`c:/Hong/Takumo/takumo-landing-page/`)

### 5.2 Configure WordPress

1. Copy `wp-config-sample.php` → `wp-config.php` (gitignored) and edit DB credentials per your local host
2. Install required plugins (not in repo):
   - **Advanced Custom Fields (ACF)** — Free or Pro
   - **Polylang** — for EN/JA
   - **Contact Form 7** (optional — only if you use CF7 forms; the PMS form is custom)
3. Browse to the WP admin; the [auto-page-creator](../takumo-landing-page/wp-content/themes/takumo-theme/inc/auto-page-creator.php) module creates Home / About / Services / Contact / Privacy / Blog plus the primary menu on first access
4. **Activate the `takumo-theme`** under Appearance → Themes
5. Set front page = "Home", posts page = "Blog" (auto-creator does this, but verify)

### 5.3 Build theme assets

```bash
cd wp-content/themes/takumo-theme
npm install
npm run build           # production webpack + wp-scripts
# or
npm run dev             # watch mode
```

Output lands in `dist/` (committed — the EC2 deploy doesn't run npm).

### 5.4 Verify

- http://takumo-landing.local/ (or whatever your host assigns)
- Switch language via Polylang
- Submit the PMS partnership form on `/pms-partnerships` or `/contact` — should redirect with `?status=success`

---

## 6. End-to-End Local Stack

For a full multi-repo dev loop:

| Service | URL |
|---|---|
| Marketing site (WP) | http://takumo-landing.local/ |
| Hotel portal (B2B) | http://localhost:4200 |
| Traveler app (B2C) | http://localhost:4200 (different terminal, different port — change `--port` flag) |
| API | http://localhost:5000 |
| Swagger | http://localhost:5000/swagger |
| Postgres | localhost:5432 |
| Redis | localhost:6379 |
| pgAdmin | http://localhost:8080 |

To run both Angular apps simultaneously, give them different ports:
```bash
# Terminal A
pnpm dev:hotel
# Terminal B
ng serve web-b2c-traveler --port 4300
```

---

## 7. Common Pitfalls

| Symptom | Cause / Fix |
|---|---|
| `pnpm install` rejects Node version | Upgrade Node to ≥ 24.1.0 |
| API exits with "Unable to resolve service for type 'IOptions<JwtSettings>'" | `appsettings.json` missing or `JwtSettings__Key` env var not set |
| API 500 on login: "BCrypt password format invalid" | A user row's `PasswordHash` is empty — re-seed or re-register |
| Angular build fails: "Missing translation `messages.ja.json`" | Run `pnpm translate:extract` then translate the new keys in JA |
| Husky pre-push fails on CRLF | `pnpm format` then re-commit; on Windows ensure `core.autocrlf input` |
| Stripe webhooks not arriving locally | Use `stripe listen --forward-to localhost:5000/api/stripe-webhook` (Stripe CLI) and set `StripeSettings__WebhookSecret` to the printed `whsec_...` |
| Apple Wallet pass downloads but doesn't open | Pass-signing cert missing from `Configs/AppleWalletConfigs/`; check `Apns.UseSandbox` in `appsettings.json` |
| Cannot clone private repo | Clear cached creds: `cmdkey /delete:git:https://github.com`, then re-clone — GCM popup will appear |
| `dotnet ef migrations add ...` errors `Build failed` | Run `dotnet build` first to surface the real error; often a `TreatWarningsAsErrors` nullable issue |

---

## 8. Where to Go Next

- Big-picture map: [ARCHITECTURE.md](./ARCHITECTURE.md)
- End-to-end flows: [FLOWS.md](./FLOWS.md)
- All API endpoints: [API.md](./API.md)
- Schema reference: [DATA_MODEL.md](./DATA_MODEL.md)
- Coding standards: [CONVENTIONS.md](./CONVENTIONS.md)
