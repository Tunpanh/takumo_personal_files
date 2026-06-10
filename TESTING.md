# Testing

A candid map of what testing infrastructure currently exists. **The honest summary:** as of the snapshot in this repo, automated testing is sparse. The README describes an intended strategy that isn't yet realized in code.

---

## 1. Backend — `takumo-api`

### 1.1 Current state

**No test projects exist in the repo.** The solution file lists only the 6 production projects (4 in `src/`, 2 in `shared/`) — no `*.Tests.csproj` is referenced. The `tests/` folder mentioned in the [takumo-api README §1](../takumo-api/README.md) is **aspirational**:

> ```
> tests/                              # Test projects for each layer
> ├── Takumo.B2BHotel.Domain.Tests/
> ├── Takumo.B2BHotel.Application.Tests/
> ├── Takumo.B2BHotel.Infrastructure.Tests/
> └── Takumo.B2BHotel.API.Tests/
> ```

That structure is the documented goal, not what's wired up.

### 1.2 What the README says the strategy should be

| Layer | Style |
|---|---|
| Domain | Unit tests for invariants & status transitions |
| Application | Unit tests for services with mocked dependencies |
| Infrastructure | Integration tests against PostgreSQL via Testcontainers |
| API | End-to-end HTTP tests using `WebApplicationFactory` |

Target coverage: **80%+ on business logic** (README §Testing Strategy).

### 1.3 Recommended tooling (per README + package conventions)

- **xUnit** — test framework
- **FluentAssertions** — readable assertions
- **NSubstitute** or **Moq** — mocking
- **Testcontainers.PostgreSql** — real DB in CI
- **Microsoft.AspNetCore.Mvc.Testing** — `WebApplicationFactory<Program>` for API tests
- `Microsoft.EntityFrameworkCore.InMemory` is already a PackageReference in [Takumo.B2BHotel.API.csproj](../takumo-api/src/Takumo.B2BHotel.API/Takumo.B2BHotel.API.csproj) — useful for fast service-layer tests, but **not equivalent to PostgreSQL** for query semantics

### 1.4 What manual verification covers today

Without a test suite, correctness verification depends on:
- Swagger UI in dev/staging — manual endpoint exercising
- FluentValidation catches obviously-malformed inputs before reaching services
- The `TreatWarningsAsErrors=true` compile-time gate (catches null misuse, unused params, etc.)
- Per-PR human review

---

## 2. Frontend — `takumo-ui`

### 2.1 Current state

**Zero `.spec.ts` files in the workspace.** Schematics are configured with `skipTests: true` for component, service, pipe, directive, guard generators — see [angular.json](../takumo-ui/angular.json).

No `karma.conf.js`, `jest.config.*`, `vitest.config.*`, `cypress.config.*`, or `playwright.config.*` exists. No `e2e/` or `cypress/` folder.

### 2.2 What CI does check

[.github/workflows/pr-checks.yml](../takumo-ui/.github/workflows/pr-checks.yml) runs:
1. **Prettier format check** (`pnpm format:ci`)
2. **Translation completeness** (`pnpm translate:validate`) — fails if any key is missing in `en.json`/`ja.json` or orphaned
3. **Build for all projects** — production build per app (this catches type errors, template-binding errors, missing imports, broken dependency injection — significant coverage in practice because `strictTemplates`, `strict: true`, and `TreatWarningsAsErrors` style gates are aggressive)

The build succeeding ≠ runtime correctness, but it does block a large class of regressions.

### 2.3 Husky hooks (pre-push)

`pnpm format:ci && pnpm translate:validate` — same checks as CI, locally.

### 2.4 If you add tests

Recommended path based on the Angular 21 stack:

- **Component / service:** `vitest` + `@angular/build` test target (or fall back to Karma + Jasmine, the historical Angular default — though Angular 21 deprecates Karma)
- **E2E:** Playwright is currently the Angular-recommended choice
- Place under per-project `src/**/*.spec.ts`; add `test` target to `angular.json`

---

## 3. Marketing — `takumo-landing-page`

### 3.1 Current state

No automated tests. PHP code is exercised by manual smoke-testing on staging after deploys (`staging` branch → staging environment via [.github/workflows/deploy-site.yaml](../takumo-landing-page/.github/workflows/deploy-site.yaml)).

### 3.2 What is automated

- **ESLint** via `npm run lint:js`
- **stylelint** via `npm run lint:css`
- **Prettier** for formatting

No CI workflow runs the linters automatically — they must be invoked manually before pushing.

---

## 4. Acceptance Testing Across the Stack

The end-to-end "user can create a booking" path crosses all three repos:
- Marketing site → traveler clicks signup
- Traveler app → calls API
- Hotel app → staff sees notification and processes

There is **no automated cross-repo test harness**. Acceptance verification is manual on staging.

---

## 5. Adding the First Test (Recommended Order)

If you're starting from scratch, the highest-value places to add tests in order:

1. **`BookingService.CreateBookingAsync` happy path + key failure modes** ([BookingService.cs](../takumo-api/src/Takumo.B2BHotel.Application/Services/BookingService/BookingService.cs)) — this is the revenue-bearing flow; status transitions are subtle.
2. **`CreateBookingRequestValidator`** ([validator](../takumo-api/src/Takumo.B2BHotel.Application/Validators/Bookings/CreateBookingRequestValidator.cs)) — easy to unit-test, prevents validation regressions on a frequently-edited DTO.
3. **`StripeWebhookHandler` per event type** ([Webhooks/](../takumo-api/src/Takumo.B2BHotel.Infrastructure/Webhooks/)) — idempotency + side-effect tests catch billing bugs that are otherwise discovered in production.
4. **`AuthController.Login` + token refresh** — security-critical; mocking `RecaptchaService` and `StaffUserService` is straightforward.
5. **Frontend `AuthStore`** signal transitions and `httpErrorInterceptorFactoryFunc` 401-refresh behavior — these mediate every authenticated request.

---

## 6. Manual Test Checklist (pre-release)

Until automation exists, a baseline smoke list for a release:

- **B2B**: login → create booking → cancel booking → invite staff → toggle PMS integration → switch language
- **B2C**: register → verify email → view shipment → add to Apple Wallet → view in browser, then in Wallet
- **API**: `/api/health/detailed` returns OK; `/swagger` loads
- **Marketing**: home page renders; switch JA / EN via Polylang; submit PMS partnership form

Run on the staging environment after deploy, before tagging `v*`.
