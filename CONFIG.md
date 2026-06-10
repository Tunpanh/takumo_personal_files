# Configuration

Every config key, environment variable, secret file, and where it's consumed. **No actual secret values are reproduced here** — only key names, default values where committed, and their location.

---

## 1. Backend — `takumo-api`

### 1.1 `appsettings.json` ([file](../takumo-api/src/Takumo.B2BHotel.API/appsettings.json))

Bound into strongly-typed POCOs under [Application/Settings/](../takumo-api/src/Takumo.B2BHotel.Application/Settings/) and registered in `AddApplicationServices` / `AddInfrastructureServices`.

#### `Logging` / `Serilog`
Standard ASP.NET logging levels + Serilog sinks (Console, File). Files roll daily at `logs/takumo-{date}.txt`, retained 30 days.

#### `EntityFramework`
| Key | Default | Notes |
|---|---|---|
| `CommandTimeout` | `30` (sec) | EF command timeout |
| `EnableRetryOnFailure` | `true` | Use Npgsql resilience |
| `MaxRetryCount` | `3` | |
| `MaxRetryDelay` | `00:00:05` | |
| `RunMigrationsOnStartup` | `false` | Migrations must run separately (see [SETUP.md](./SETUP.md)) |

#### `JwtSettings`
| Key | Default |
|---|---|
| `Issuer` | `Takumo.B2BHotel.API` |
| `Audience` | `Takumo.B2BHotel.Client` |
| `AccessTokenExpiry` | `01:00:00` |
| `RefreshTokenExpiry` | `7.00:00:00` |
| `ClockSkew` | `00:00:00` |
| `Key` | **(env / secret only — not in file)** |

#### `ShipAndCoSettings`
| Key | Default |
|---|---|
| `ApiUrl` | `https://api.shipandco.com/v1` |
| `ApiKey` | **secret** |

#### `DateTimeSettings`
| Key | Default |
|---|---|
| `TimeZone` | `Asia/Tokyo` |

#### `FirebaseSettings`
| Key | Default |
|---|---|
| `CredentialPath` | `Configs/firebase-credentials.json` (gitignored file) |
| `BasePath` | `notifications` |

#### `AwsSesSettings`
| Key | Default |
|---|---|
| `Region` | `ap-northeast-1` |
| `AccessKey` / `SecretKey` | **env-only** |

#### `AwsS3Settings`
| Key | Default |
|---|---|
| `LabelsBucketName` | `takumo-dev-labels` |
| `GeneralBucketName` | `takumo-dev-general` |

Production override expected to point at prod buckets.

#### `GoogleWalletSettings`
Pass design + classes (logo desc, headers, labels). Static template config.
- `ClassIdSuffix`: `HotelGuestPass`
- `ObjectIdPrefix`: `guest_`
- `BackgroundColor`: `#A12850`
- `DefaultLanguage`: `en-US`
- `Content.*`: all visible pass labels (server-side translated)
- Service account: **env / secret file (not in repo)**

#### `AppleWalletSettings`
| Key | Default |
|---|---|
| `PassTypeIdentifier` | `pass.com.takumo.passmaker` |
| `TeamIdentifier` | `65TN39WV65` |
| `Description` | `Takumo Guest Pass` |
| `OrganizationName` | `Takumo` |
| `LogoText` | `Takumo Guest Pass` |
| `BackgroundColor` | `rgb(161, 40, 80)` |
| `LabelColor` / `ForegroundColor` | white |
| `AppleWWDRCACertificatePath` | `Configs/AppleWWDRCAG6.cer` (committed) |
| `IconPath` / `Icon2x`/`3x` | `Configs/AppleWalletConfigs/icon.png` (committed) |
| `LogoPath` / `Logo2x`/`3x` | `Configs/AppleWalletConfigs/logo.png` (committed) |
| `BarcodeEncoding` | `utf-8` |
| `ResponseContentType` | `application/vnd.apple.pkpass` |
| `Apns.AuthType` | `token` |
| `Apns.ServerUrl` | `api.push.apple.com` |
| `Apns.Port` | `443` |
| `Apns.UseSandbox` | `false` |
| Pass-signing certificate & APNs auth key | **gitignored Configs/** |

#### `TakumoSettings`
| Key | Default | Meaning |
|---|---|---|
| `EstimatedDeliveryDateAddDays` | `2` | Offset shown to user at booking time |
| `SubscriptionExpiringReminderDays.AutoRenewal` | `[3]` | Days-before-renewal to email reminder |
| `SubscriptionExpiringReminderDays.NonAutoRenewal` | `[7, 1]` | Days-before-cancellation to remind |

#### `BackgroundJobSettings`
| Key | Default |
|---|---|
| `TrackingShipmentCron` | `*/10 * * * *` |

#### `StripeSettings`
| Key | Default | Notes |
|---|---|---|
| `TestMode` | `true` | Toggle for test/live mode |
| `DefaultTrialPeriodDays` | `7` | |
| `SecretKey` | **env-only** | |
| `PublishableKey` | **env-only** (not used server-side except for Setup Intents) |
| `WebhookSecret` | **env-only** | Signature verification |

#### `YamatoB2Settings`
| Key | Default | Notes |
|---|---|---|
| `BaseUrl` | empty in committed file | Per-env override |
| `ApiUserId` | empty | |
| `AccessToken` | empty | **secret in env** |
| `InvoiceCode` / `InvoiceCodeExt` / `InvoiceFreightNo` | empty | Yamato invoice identifiers |

#### `YamatoTracking`
| Key | Default |
|---|---|
| `BaseUrl` | `https://inter-consistent2.kuronekoyamato.co.jp/consistent2/cts` |
| `PollCron` | `0 * * * *` |
| `MaxBookingsPerTick` | `500` |
| `MaxNumbersPerCall` | `20` |
| `MinIntervalSecondsPerCustomer` | `5.5` |
| `MaxHttpRetries` | `3` |
| `MaxConsecutiveFailuresBeforeDisable` | `10` |

#### `RecaptchaSettings`
| Key | Default |
|---|---|
| `VerifyEndpoint` | `https://www.google.com/recaptcha/api/siteverify` |
| `TimeoutSeconds` | `10` |
| `AllowedHostnames` | `[]` |
| `SecretKey` | **env-only** |

#### `PmsProviderSettings.Providers[]`
Schema of credential fields per provider — driven from config so the frontend can render dynamic forms.

| Provider | Required auth fields |
|---|---|
| Mews | `ApiUrl`, `ClientToken`, `AccessToken` |
| Cloudbeds | `ApiUrl`, `PropertyIDs`, `ApiKey` |
| Oracle | `ApiUrl`, `ClientID`, `ClientSecret`, `EnterpriseID`, `ApplicationKey`, `Scope`, `HotelID` |

### 1.2 Connection strings
Conventionally `ConnectionStrings__DefaultConnection` (Postgres). Format:

```
Host=postgres;Database=takumo_b2b_hotel_dev;Username=takumo_user;Password=dev_password;Port=5432;Pooling=true;Minimum Pool Size=1;Maximum Pool Size=5
```

A `ConnectionStrings__RedisConnection` is set in [scripts/run-local-api.sh](../takumo-api/scripts/run-local-api.sh) (`localhost:6379`) — Redis is provisioned but not yet actively used (no Redis client registration found at Application/Infrastructure level; reserved for future caching).

### 1.3 Environment variables (Docker Compose default values)

From [docker-compose.yml](../takumo-api/docker-compose.yml) / [docker-compose.dependencies.yml](../takumo-api/docker-compose.dependencies.yml):

| Variable | Default | Used by |
|---|---|---|
| `POSTGRES_DB` | `takumo_b2b_hotel_dev` | postgres container |
| `POSTGRES_USER` | `takumo_user` | postgres container |
| `POSTGRES_PASSWORD` | `dev_password` | postgres container |
| `POSTGRES_PORT` | `5432` | host port mapping |
| `REDIS_PORT` | `6379` | redis container |
| `API_PORT` | `8080` | api container |
| `DEFAULT_CONNECTION` | (see above) | api + migration services |
| `PGADMIN_EMAIL` | `admin@takumo.com` | pgadmin |
| `PGADMIN_PASSWORD` | `admin123` | pgadmin |
| `PGADMIN_PORT` | `8080` | pgadmin |
| `ASPNETCORE_ENVIRONMENT` | `Development` (in dev compose) | .NET host |

**Secrets expected via env or user-secrets but NOT committed:**
- `JwtSettings__Key`
- `StripeSettings__SecretKey`, `StripeSettings__WebhookSecret`
- `AwsSesSettings__AccessKey`, `AwsSesSettings__SecretKey` (or use IAM role)
- `RecaptchaSettings__SecretKey`
- `GoogleApiKey` (server-side for Maps/Places)
- `YamatoB2Settings__AccessToken`
- `SendGrid` API key (under SendGrid settings — not visible in appsettings.json so env-only)
- `ShipAndCoSettings__ApiKey`

### 1.4 Filesystem-mounted secrets (gitignored)

Under [src/Takumo.B2BHotel.API/Configs/](../takumo-api/src/Takumo.B2BHotel.API/Configs/):

| File | Purpose |
|---|---|
| `firebase-credentials.json` | Firebase service account |
| `AppleWalletConfigs/*.cer` and `*.key` | Apple Pass signing certificate + private key |
| `AppleWWDRCAG6.cer` | Apple WWDR intermediate cert (committed — public cert) |
| `AppleWalletConfigs/apns_auth_key.p8` | APNs token-auth key |
| Google Wallet service account JSON | Pass class management |

`.csproj` `<None Update="Configs\**\*">` copies them to build output.

---

## 2. Frontend — `takumo-ui`

### 2.1 Runtime config — `app-config.json`

Loaded via `fetch()` in `main.ts` before app bootstrap. **Template is committed; real file is gitignored**.

#### B2B Hotel ([template](../takumo-ui/projects/web-b2b-hotel/src/app/app-config.template.json))

```json
{
  "api": { "baseUrl": "placeholder" },
  "firebase": {
    "apiKey": "placeholder",
    "authDomain": "placeholder",
    "databaseURL": "placeholder",
    "projectId": "placeholder",
    "storageBucket": "placeholder",
    "messagingSenderId": "placeholder",
    "appId": "placeholder"
  },
  "googleApiKey": "placeholder",
  "recaptchaSiteKey": "placeholder",
  "travelerUrl": "placeholder",
  "termsVersion": "placeholder",
  "privacyVersion": "placeholder"
}
```

#### B2C Traveler ([template](../takumo-ui/projects/web-b2c-traveler/src/app/app-config.template.json))

```json
{
  "api": { "baseUrl": "placeholder" },
  "googleApiKey": "placeholder",
  "recaptchaSiteKey": "placeholder",
  "termsVersion": "placeholder",
  "privacyVersion": "placeholder"
}
```

B2C does NOT load Firebase config — only the B2B portal listens to realtime notifications.

### 2.2 Build-time config

Static via `angular.json` per project — output paths, asset folders, locale lists, budget limits. No secrets here.

| Item | Where |
|---|---|
| Supported locales | `angular.json` → `projects.<app>.i18n.locales` |
| Output budgets | `angular.json` → `projects.<app>.architect.build.configurations.production.budgets` |
| Build script env (CI) | `.github/workflows/*.yml` — see [§4 below](#4-deployment-config) |

### 2.3 Dev proxy

[development/proxy.conf.base.js](../takumo-ui/development/proxy.conf.base.js) rewrites `/global/*` → `/{locale}/global/*` for the dev server so assets load correctly under each locale. Separate proxies per language: `proxy.conf.js` (EN), `proxy.conf.ja.js` (JA).

---

## 3. Marketing — `takumo-landing-page`

### 3.1 `wp-config.php`

Not committed (gitignored). The committed reference is [wp-config-sample.php](../takumo-landing-page/wp-config-sample.php) — standard WordPress install template.

Custom additions to be made in `wp-config.php`:
- Image processing timeout increased (theme expects long uploads)
- Automatic image resizing disabled to avoid UI hangs

### 3.2 Theme & site settings

ACF Options Page ("General Settings" — registered in [inc/general-settings.php](../takumo-landing-page/wp-content/themes/takumo-theme/inc/general-settings.php)) holds:
- Header logo upload
- Header / footer menu selection
- Contact email, phone, address
- Business hours

These are stored in the `wp_options` table, edited via WP admin, not in code.

### 3.3 Build config

[wp-content/themes/takumo-theme/package.json](../takumo-landing-page/wp-content/themes/takumo-theme/package.json) and [webpack.config.js](../takumo-landing-page/wp-content/themes/takumo-theme/webpack.config.js). No environment variables — output goes to theme `dist/`.

### 3.4 Required runtime plugins (not in repo)
- **ACF** (Free or Pro) — required, theme assumes it
- **Polylang** — required for EN/JA
- **Contact Form 7** — referenced by `dist/js/cf7-redirect.js`; not required if all forms use the custom PMS handler

---

## 4. Deployment Config

### `takumo-ui` GitHub Actions

In [.github/workflows/](../takumo-ui/.github/workflows/):

| Env var | Purpose |
|---|---|
| AWS role ARN (OIDC) | Assume role for S3 + CloudFront access |
| `S3_BUCKET` | Target bucket per environment |
| `CLOUDFRONT_DISTRIBUTION_ID` | For cache invalidation |
| `APP_CONFIG_JSON` (built secret) | The runtime `app-config.json` injected at deploy time |

Environments: `dev`, `staging`, `prod`. Trigger:
- Push to `staging` branch → staging
- Tag `v*` → prod
- Manual `deploy-to-dev-manually.yml` → dev

### `takumo-landing-page` GitHub Actions

In [.github/workflows/deploy-site.yaml](../takumo-landing-page/.github/workflows/deploy-site.yaml):

| Env var | Source | Used in |
|---|---|---|
| `ROLE_TO_ASSUME` | repo env var per environment | AWS OIDC assume-role |
| `AWS_REGION` | repo env var | AWS SDK |
| `INSTANCE_ID` | repo env var | SSM SendCommand target (EC2 instance ID) |
| `DEPLOY_USER` | repo env var | `sudo -u $DEPLOY_USER` on EC2 |

Deploy script on the EC2: `/var/www/scripts/deploy.sh $BRANCH`. Branch decides target dir (prod or staging).

### `takumo-api` CI/CD
Not codified in the repo. Dockerfile lives at [src/Takumo.B2BHotel.API/Dockerfile](../takumo-api/src/Takumo.B2BHotel.API/Dockerfile). Migration container at [Dockerfile.migration](../takumo-api/Dockerfile.migration). Deploy mechanism owned outside this repo.

---

## 5. Configuration Loading Order (.NET)

Standard ASP.NET Core order, last wins:
1. `appsettings.json`
2. `appsettings.{Environment}.json` (none committed currently — add per-environment overrides as needed)
3. User secrets (Development only)
4. Environment variables (e.g. `JwtSettings__Key`, `ConnectionStrings__DefaultConnection`)
5. Command-line arguments

Double underscore `__` translates the `:` separator (e.g. `JwtSettings__Key` ↔ `JwtSettings:Key`).
