# PROJECT STATE FULL - SYNDICATE

Дата аудита: 2026-06-02 20:45 UTC+3  
Последнее обновление (debug session): **2026-06-03** — CSRF, Argon2, wheel race, чистка dead code, UI/constructor/mailer (см. §14).  
Версия проекта: 2.0.0 (`package.json`)  
Цель документа: единый фактический state проекта + сверка старых документов + roadmap для дальнейшей имплементации.

Этот файл вобрал и заменил собой ранее существовавшие рабочие документы (удалены при чистке docs/ 2026-06-02): `TAILWIND_MIGRATION.md`, `SECURITY_FIXES_FINAL.md`, `DOMAIN_SPLIT_IMPLEMENTATION_PLAN.md`, `DOMAIN_SPLIT_REPORT_RU.md`, `DEPLOYMENT_AND_TUNNELS.md`, `GITHUB_CLI.md`, `DATA_FLOW.md`, `TODO.md`, дубли `SECOND_LAPTOP.md`, `HOSTING_MINIMUMS.md`, `COLOR_SUPPORT.md`.

Актуальный набор документации проекта теперь минимален:

- `docs/STARTUP.md` — как поднять/перенести проект (ноут + VPS, туннели, флаги, troubleshooting).
- **`docs/SOSTOYANIE_PROEKTA_20260603.md`** — **русскоязычный** источник правды (вердикт, домены, CSRF, чеклист деплоя, журнал §14).
- `docs/PROJECT_STATE_FULL_*.md` — этот файл: полный аудит API/БД, сверка старых доков, §13 (можно EN/RU вперемешку).
- `deploy/README.md` — операционный deploy-гайд (env, туннели, hosting-топология, Docker, packaging).
- `deploy/cloudflared/README.md` — заметка про переносимые tunnel-креды.

Для ежедневной работы на русском — **`SOSTOYANIE_PROEKTA_20260603.md`**. Для глубокого аудита и file:line — этот файл.

---

## 0. Короткий Вердикт

Проект уже не в состоянии "сырой Telegram mini app". Это Express + EJS + SQLite приложение с двумя пользовательскими режимами:

- `app.sndct.cc` - Telegram Mini App режим.
- `sndct.cc` / `www.sndct.cc` - обычный web режим.

Фактически уже реализованы:

- host-based runtime mode через `src/middleware/runtimeMode.js`;
- EJS-рендер страниц с `window.__APP_RUNTIME__`;
- раздельные cookie names для web/miniapp в `src/middleware/currentUser.js`;
- web auth через email/password + email verification;
- `/forgot-password` + `/reset-password`;
- staff/admin плоскость доступа;
- queue-first redeem flow через `bonus_withdrawal_requests`;
- Tailwind v4 + DaisyUI build pipeline;
- базовые Jest smoke tests.

Главная проблема: часть документации уже устарела, а часть кода выглядит "завершённой" только на уровне happy path.

**Bugfix-прогон (2026-06-02, вечер):** после §13 закрыты критичные P0 по выводам, миграциям, staff-паролям, API/FE контрактам и восстановлению `/constructor` + `/shop-cabinet`. Проверено на прод-данных: reject withdrawal + `redeem_refund` + строка в `withdrawal_audit_logs`. Staff UI: модалка, хедер, scope менеджера.

**Debug session (2026-06-02 ночь → 2026-06-03):** ✅ CSRF (только web cookie-session, miniapp с `initData` не трогаем); ✅ rate limit `forgot-password` / `reset-password`; ✅ Argon2id в `src/lib/password.js` + upgrade на логине; ✅ wheel spin в одной транзакции; ✅ `DATABASE_PATH` в `resolveDatabasePath.js`; ✅ SendGrid без click-tracking; ✅ вход web = бот `/login` + `POST /web/bot-login` (не `link/start`); ✅ чистка dead code (см. §14); ✅ `npm run verify` — **19 тестов**. **Остаётся:** Google OAuth stub, `bot/generate-token` hardening, CORS no-Origin policy, service layer, prompt/confirm в constructor (если ещё есть), массовая замена `console.*` → `logger`.

---

## 1. Фактический Стек

Runtime:

- Node.js >= 18, ESM (`"type": "module"`).
- Express 4.
- EJS templates с `.html` views.
- SQLite через `better-sqlite3`.
- Optional Redis cache через `ioredis`.
- Python Telegram bot entrypoint: `bot.py`.
- Telegram auth/bot integration через `telegraf` и Telegram WebApp initData.
- Tailwind CSS v4 + DaisyUI v5.
- Jest + Supertest.
- ESLint flat config.
- Docker deploy files exist: `Dockerfile`, `docker-compose.yml`.

Ключевые команды:

```bash
npm start
npm run dev
npm run migrate
npm test
npm run test:ci
npm run verify
npm run check
npm run build:css
npm run cleanup:test-shops
npm run watch:css
npm run vps:setup
npm run vps:up
```

Build CSS уже есть:

```json
"build:css": "./node_modules/.bin/tailwindcss -i ./public/css/tailwind.input.css -o ./public/css/tailwind.css --minify",
"watch:css": "./node_modules/.bin/tailwindcss -i ./public/css/tailwind.input.css -o ./public/css/tailwind.css --watch"
```

---

## 2. Project Structure

Высокоуровневая структура:

```text
src/
  server.js                       Express bootstrap, pages, health, API mount
  middleware/
    runtimeMode.js                host -> web/miniapp mode
    currentUser.js                user session + Telegram initData auth; CSRF cookie on login
    csrf.js                       CSRF gate (session-only, skips initData)
    staffAuth.js                  staff/admin session resolver
    telegramAuth.js               Telegram payload verification/upsert
  lib/
    password.js                   Argon2id + PBKDF2 + legacy SHA-256 verify/upgrade
    csrf.js                       token issue/verify, cookie `syndicate_csrf`
    mailer.js                     SendGrid; clicktrack/opentrack off
  routes/api/v1/
    index.js                      API router mount
    auth.js                       user auth, admin legacy auth, password reset
    shops.js                      public shops + constructor write API
    bonuses.js                    bonuses, daily, redeem queue creation
    staff.js                      staff login, withdrawals, analytics, logs
    services.js                   services API
    user.js                       profile/user actions
    wheel.js                      wheel/spins (atomic transaction)
    ui.js                         UI metrics/events
  config/
    database.js                   SQLite init + WAL + busy_timeout + migrations
    dbAsyncWorker.js              async DB worker (uses resolveDatabasePath)
    resolveDatabasePath.js        DATABASE_PATH / default syndicate.db
    migrate.js                    migration runner
    redis.js                      optional Redis cache wrapper
    migrations/                   18 migration files (018 re-applies 017 indexes)
  views/
    *.html                        EJS pages
    partials/runtime-head.html    __APP_RUNTIME__ + __CSRF_TOKEN__ (web only)
    partials/shop-editor-form.html constructor / shop-cabinet editor
bot.py                            Python Telegram bot entrypoint
public/
  js/
    pages/                        page controllers
    managers/                     auth/nav/telegram/layout managers
    utils/                        API/runtime/cache/config helpers
    domains/games/                wheel domain JS
  css/
    global/main.css               shell/header/nav/base theme
    tailwind.input.css            Tailwind v4 CSS-first config
    tailwind.css                  compiled output
    components/                   wheel/popup/shop-card CSS still retained
docs/
deploy/
Dockerfile, docker-compose.yml     container deploy path
```

Page routes currently registered in `src/server.js`:

- `/`
- `/bonuses`
- `/services`
- `/login`
- `/reset-password`
- `/forgot-password`
- `/constructor`
- `/admin-panel`
- `/admin-dashboard` -> redirects to `/admin-panel`
- `/shop-cabinet`
- `/profile`
- `/work`

Views found:

- `src/views/index.html`
- `src/views/bonuses.html`
- `src/views/services.html`
- `src/views/login.html`
- `src/views/forgot-password.html`
- `src/views/reset-password.html`
- `src/views/profile.html`
- `src/views/constructor.html`
- `src/views/shop-cabinet.html`
- `src/views/admin-dashboard.html`
- `src/views/work.html`

---

## 3. Source Of Truth By Domain

### 3.1 Runtime / Domain Split

Truth:

- `src/middleware/runtimeMode.js`
- `src/server.js`
- `public/js/utils/runtime.js`
- page templates under `src/views/*.html`

Actual behavior:

- `APP_PUBLIC_HOSTS` controls allowlist when provided.
- Default allowed hosts: `sndct.cc`, `www.sndct.cc`, `app.sndct.cc`, `localhost`, `127.0.0.1`.
- `MINIAPP_HOST` defaults to `app.sndct.cc`.
- `FEATURE_HOST_SPLIT !== 'false'` means host enforcement is on by default.
- Unknown host returns `421 MISDIRECTED_REQUEST`.
- `FEATURE_CANONICAL_REDIRECTS === 'true'` can redirect web hosts to canonical host.

Core snippet:

```js
export function runtimeModeMiddleware(options = {}) {
  const allowedHosts = parseAllowedHosts();
  const miniHost = String(process.env.MINIAPP_HOST || 'app.sndct.cc')
    .trim()
    .toLowerCase();
  const enforceHost = process.env.FEATURE_HOST_SPLIT !== 'false';

  return (req, res, next) => {
    const host = resolveHost(req);
    const isAllowed = host && allowedHosts.has(host);
    const mode = host === miniHost ? 'miniapp' : 'web';

    req.runtimeHost = host || null;
    req.runtimeMode = mode;

    if (enforceHost && !isAllowed) {
      return res.status(421).json({
        error: 'MISDIRECTED_REQUEST',
        message: 'Host is not allowed',
      });
    }

    res.locals.runtimeHost = req.runtimeHost;
    res.locals.runtimeMode = req.runtimeMode;
    next();
  };
}
```

Frontend runtime helper:

```js
window.runtime = {
  mode: mode,
  host: host,
  requestId: requestId,
  isMiniApp: mode === 'miniapp' || isTelegram,
  isWeb: mode === 'web' && !isTelegram,
  isTelegram: isTelegram
};
```

Important correction: docs mention `window.__RUNTIME__` in one place, but actual helper exports `window.runtime`.

### 3.2 User Auth Plane

Truth:

- `src/routes/api/v1/auth.js`
- `src/middleware/currentUser.js`
- `src/middleware/telegramAuth.js`
- `public/js/pages/login.js`
- `public/js/pages/profile.js`

Flows implemented:

- Telegram login widget: `POST /api/v1/auth/browser/telegram`.
- Telegram WebApp auth: `POST /api/v1/auth/browser/webapp`.
- Bot code generation: `POST /api/v1/auth/bot/generate-token`.
- Web bot-code login: бот `/login` → 6-значный код → `POST /api/v1/auth/web/bot-login` (форма на `/login` и `/bonuses`). Не miniapp.
- Web registration: `POST /api/v1/auth/web/register`.
- Email verification: `POST /api/v1/auth/verify-email`.
- Web login: `POST /api/v1/auth/web/login`.
- Forgot password: `POST /api/v1/auth/forgot-password`.
- Reset password: `POST /api/v1/auth/reset-password`.
- Logout: `POST /api/v1/auth/logout`.
- Delete account: `DELETE /api/v1/auth/delete`.

Session cookie behavior:

- Split enabled: `syndicate_web_sid` and `syndicate_mini_sid`.
- Split disabled: legacy `syndicate_sid`.
- Cookies are `HttpOnly; SameSite=Lax`; `Secure` only in production.

Current password hashing (`src/lib/password.js`, 2026-06-03):

- **Новые пароли:** Argon2id (`hashPassword()` → `$argon2id$...`).
- **Verify:** `verifyPasswordAsync()` — argon2, затем PBKDF2 (`pbkdf2$...`), затем legacy SHA-256.
- **Upgrade on login:** `needsPasswordUpgrade()` → re-hash в Argon2 при успешном входе (users `auth.js`, managers `staff.js`).
- Manager create/update (`shops.js`): `await hashPassword()`.
- Дублирующие `hashPasswordStrong` / `verifyPasswordStrong` **удалены из `auth.js`** — только импорт из `lib/password.js`.
- `hashPasswordStrong()` в lib оставлен для тестов/backfill PBKDF2; не использовать для новых записей.

### 3.2.1 CSRF (web shell only, 2026-06-03)

Truth:

- `src/lib/csrf.js` — issue/verify, cookie `syndicate_csrf`, header `X-CSRF-Token`.
- `src/middleware/csrf.js` — после `optionalCurrentUser` на `/api/v1`.
- `createBrowserSession` — выдаёт CSRF + Set-Cookie; revoke чистит `csrf_tokens` + cookie.
- EJS: `partials/runtime-head.html` → `window.__CSRF_TOKEN__` только если `runtimeMode !== 'miniapp'`.
- `server.js` → `renderWebPage()` подставляет токен на web-страницах.
- `public/js/utils/api.js` — шлёт `X-CSRF-Token` если есть `__CSRF_TOKEN__` и **нет** `tg.initData`.
- `GET /auth/status` и ответы логина — поле `csrf_token`.

**Hard rule (не ломать miniapp):**

```js
function csrfRequired(req) {
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) return false
  if (req.authType === 'telegram') return false
  if (req.headers['x-telegram-init-data']) return false
  return req.authType === 'session'
}
```

Tests: `src/tests/csrf.session.test.js` (403 без токена, 200 с токеном, skip при initData header).

**Не CSRF:** staff cookie (`req.staff`), guest forgot/reset, miniapp `POST /wheel/spin` с initData.

### 3.3 Staff / Admin / Control Plane

Truth:

- `src/routes/api/v1/staff.js`
- `src/middleware/staffAuth.js`
- `src/routes/api/v1/shops.js`
- `public/js/pages/constructor.js`
- `src/views/constructor.html`
- `src/views/shop-cabinet.html`
- `src/views/admin-dashboard.html`

Roles:

- `admin`: global.
- `manager`: scoped by `shop_scope_slug`.

Staff endpoints:

- `GET /api/v1/staff/status`
- `POST /api/v1/staff/login`
- `POST /api/v1/staff/logout`
- `GET /api/v1/staff/withdrawals`
- `POST /api/v1/staff/withdrawals/:id/resolve`
- `POST /api/v1/staff/withdrawals/:id/:action`
- `GET /api/v1/staff/analytics`
- `GET /api/v1/staff/logs`

Important behavior:

- `staffAuth.js` falls back from `syndicate_staff_sid` to legacy `syndicate_admin_sid`.
- Constructor logout calls both `/api/v1/staff/logout` and `/api/v1/auth/admin/logout`.
- Manager: `GET /shops/manage/list` и `GET /staff/withdrawals` scoped по `shop_scope_slug`; `PUT /shops/:slug` для manager — только `cities`, `features`, `links`, `visual` (accent/media), без `name`/`sort_order`/manager credentials.
- `/constructor` (admin) и `/shop-cabinet` (manager) — один `constructor.js` + partial `src/views/partials/shop-editor-form.html`; `config.js` + `authManager.js` на staff-страницах.
- `/admin-panel`: пути исправлены на `apiCall('/v1/staff/...')` + подключён `config.js` (бывший P0-9).
- Staff UI (2026-06-02): модалка с прокруткой, `setVisible` для `[hidden]`, карточки `.constructor-card`; менеджеру в форме только медиа/акцент/города/фичи/ссылки; категория убрана из редактора; порядок полей = карточка шопа.

### 3.4 Shops / Constructor / Data Flow

Truth:

- `src/routes/api/v1/shops.js`
- `public/js/pages/constructor.js`
- `src/views/partials/shop-editor-form.html` (EJS include из `constructor.html` / `shop-cabinet.html`)

Public read API:

- `GET /api/v1/shops`
- `GET /api/v1/shops/meta/categories`
- `GET /api/v1/shops/:slug`

Staff write API:

- `GET /api/v1/shops/manage/list`
- `POST /api/v1/shops`
- `PUT /api/v1/shops/:slug`
- `DELETE /api/v1/shops/:slug`
- `POST /api/v1/shops/media/upload`

Data model:

- Structured fields (`cities`, `features`, `links`, `visual`) are persisted as JSON strings.
- Public API serializes JSON strings into objects/arrays.
- Redis cache keys use `shops:*`.
- Writes invalidate `shops:*`.

Snippet:

```js
function serializeShop(shop) {
  return {
    ...shop,
    cities: parseJson(shop.cities_json, []),
    features: parseJson(shop.features_json, []),
    links: parseJson(shop.links_json, {}),
    visual: parseJson(shop.visual_json, {}),
    popup_enabled: Boolean(shop.popup_enabled),
    is_featured: Boolean(shop.is_featured),
    is_verified: Boolean(shop.is_verified),
  }
}
```

### 3.5 Bonuses / Redeem Queue

Truth:

- `src/routes/api/v1/bonuses.js`
- `src/routes/api/v1/staff.js`
- `public/js/pages/bonuses.js`
- `public/js/pages/constructor.js`

Actual business flow:

1. User submits `POST /api/v1/bonuses/redeem`.
2. Backend debits balance transactionally.
3. Backend inserts `bonus_withdrawal_requests` with `status='pending'`.
4. Manager/admin sees request via `GET /api/v1/staff/withdrawals?status=pending`.
5. Manager/admin resolves via `POST /api/v1/staff/withdrawals/:id/resolve`.
6. Reject refunds balance and inserts `redeem_refund`.
7. Resolve writes `withdrawal_audit_logs` под схему 011 (`withdrawal_id`, `ip_address`, `user_agent`; расширенные поля в `metadata_json`). Подтверждено на живой БД после bugfix.

This confirms the Domain Split docs correction: manager Telegram contact is auxiliary UX, not the canonical execution channel.

Redeem creation snippet:

```js
const transaction = db.transaction(() => {
  const debit = db.prepare(
    'UPDATE users SET balance = balance - ? WHERE id = ? AND balance >= ?'
  ).run(amount, user.id, amount)

  if (!debit.changes) {
    return { insufficient: true, balance: Number(user.balance) || 0 }
  }

  const insert = db.prepare(`
    INSERT INTO bonus_withdrawal_requests (
      request_code, user_id, shop_id, shop_slug, amount, status, manager_contact, manager_message
    ) VALUES (?, ?, ?, ?, ?, 'pending', ?, ?)
  `).run(requestCode, user.id, shop.id, shop.slug, amount, managerContact, managerMessage)

  return { insufficient: false, request: requestRow }
})
```

---

## 4. Сверка Документации И Факта

### 4.1 `docs/TAILWIND_MIGRATION.md`

Status: частично устарел, частично уже применён, частично остаётся roadmap.

Что уже реализовано:

- Tailwind v4 installed.
- DaisyUI installed.
- `build:css` and `watch:css` exist in `package.json`.
- `postcss.config.js` uses `@tailwindcss/postcss`.
- `tailwind.config.js` scans `src/views/**/*.html` and `public/js/**/*.js`.
- `public/css/tailwind.input.css` contains `@import "tailwindcss";` and `@plugin "daisyui";`.
- `public/css/tailwind.css` exists.
- `login.html` no longer uses Tailwind CDN.
- `reset-password.html` no longer references missing `/css/pages/login.css`.
- `/forgot-password` page and JS exist.
- `login.html/login.js` are domain-aware and use web default email tab.
- `profile.html/profile.js` are redesigned with Tailwind cards.

Что всё ещё актуально:

- `main.css` must remain: app shell/header/bottom nav live there.
- `wheel.css` should remain: canvas/animation CSS.
- `popup.css` can remain until popup system is redesigned.
- `shop-card.css` still used by `index.html` and `services.html`.
- Need run `npm run build:css` when Tailwind classes change.

What is stale/wrong:

- "No CSS build script" is false.
- "login.html uses CDN" is false.
- "reset-password.html broken due login.css" is false.
- The guide says use `window.__RUNTIME__` in one correction; actual helper is `window.runtime`.

Remaining design debt:

- Too many raw arbitrary color utilities like `text-[#e0e0e0]`, `bg-[#141922]`.
- DaisyUI is installed but barely used as semantic component system.
- External CDN scripts still exist on login: Toastify, JustValidate, GSAP.

### 4.2 `docs/SECURITY_FIXES_FINAL.md`

Status: over-optimistic. Some fixes are real, but "ready for production" is not a safe conclusion.

Confirmed real fixes:

- Email verification uses atomic `UPDATE`.
- Email verify failure returns generic `INVALID_CODE`.
- Register/login/verify/bot-login rate limiters exist.
- DB index definitions exist in `017_add_database_indexes.js`, but the migration runner will not execute this file as written because it exports `default`, not `up(db)`. Verify actual DB indexes before relying on the doc claim.
- `helmet` is mounted.
- `app.set('trust proxy', 1)` exists.
- Runtime host split works in tests.

Important corrections:

- CSP is disabled: `contentSecurityPolicy: false`.
- CORS no-Origin state-changing requests are logged but not blocked.
- ~~CSRF~~ ✅ wired (2026-06-03): `csrf.js` + middleware + EJS + `api.js`; table `csrf_tokens` используется.
- ~~`argon2` unused~~ ✅ Argon2id в `lib/password.js`.
- ~~Password reset rate limit~~ ✅ `forgotPasswordLimiter` / `resetPasswordLimiter` в `auth.js`.
- Forgot-password endpoint has no input validation beyond lowercase/lookup.
- `validateTurnstileToken()` — удалена вместе с hardcoded secret (2026-06-02).
- ~~Duplicate `POST /logout`~~ ✅ удалён дубль в `auth.js` (2026-06-03).
- Observability в `index.js` — **перед** sub-routers; роль из `req.staff` (2026-06-02).
- `public/js/utils/api.js` — парсит JSON-ошибки (`err.error`, `err.payload`) (2026-06-02).

~~Hard blocker (withdrawal audit):~~ **исправлено 2026-06-02** — INSERT в `staff.js` под схему 011; prod reject + audit row проверены.

### 4.3 `docs/DOMAIN_SPLIT_IMPLEMENTATION_PLAN.md`

Status: architecture direction is correct, but many "to add" items are already implemented.

Implemented:

- Single codebase, single backend, single DB.
- Host-based runtime middleware.
- EJS runtime injection.
- `window.runtime` helper.
- `/bonuses` shared route with mode-aware frontend.
- Web auth exists.
- Miniapp auth exists.
- Staff queue model exists.
- Tests cover host mode detection.

Still missing or incomplete:

- No full E2E matrix across both domains.
- ~~No hard CSRF~~ ✅ web-only CSRF (§3.2.1).
- No explicit cookie host/domain policy tests.
- No host-tagged production metrics/alerts.
- No canonical host redirect enabled by default unless env flag set.
- No service layer; business logic still lives inside large route files.

Important env truth:

```env
FEATURE_HOST_SPLIT=true
FEATURE_CANONICAL_REDIRECTS=true
APP_PUBLIC_HOSTS=sndct.cc,www.sndct.cc,app.sndct.cc
MINIAPP_HOST=app.sndct.cc
WEB_SESSION_COOKIE_NAME=syndicate_web_sid
MINI_SESSION_COOKIE_NAME=syndicate_mini_sid
```

### 4.4 `docs/DOMAIN_SPLIT_REPORT_RU.md`

Status: mostly accurate as a narrative summary, but too high-level.

Accurate:

- Project now supports Telegram Mini App + web mode.
- EJS/runtime bridge exists.
- Bonuses UI has been adapted.
- Tests exist and cover basic health/domain split.
- Next steps around Argon2, service layer, staff queue, fraud controls are valid.

Needs correction:

- "Tests successfully pass" should be re-verified after every schema/code change.
- ~~"Argon2id next"~~ ✅ done in `lib/password.js` (2026-06-03).
- "Queue intelligence" is partially implemented at API level but not polished in UI.

---

## 5. Critical Must-Fix Items

| ID | Тема | Статус bugfix 2026-06-02 |
|---|---|---|
| P0 audit | `withdrawal_audit_logs` INSERT | ✅ `staff.js` → схема 011 + `metadata_json` |
| P0 indexes | миграция 017 / индексы | ✅ `018_apply_missing_indexes.js`, 017 с `export function up` |
| P0 manager hash | SHA-256 в `shops.js` | ✅ `src/lib/password.js` PBKDF2 |
| P0 API errors | `api.js` | ✅ JSON propagation |
| P0 observability | порядок middleware | ✅ до route mounts |
| P0 admin FE | `/api/api/v1` | ✅ `/v1/staff/...` + `config.js` |
| P0 password leak | `manage/list` | ✅ без `generated_password` в ответе |
| P0 Turnstile | secret в repo | ✅ функция удалена |
| P0 constructor FE | пустой partial, JS crash | ✅ partial + UI pass (отдельно от таблицы) |
| P0 link/start | miniapp pairing flow | ✅ **удалён**; web = `POST /web/bot-login` + код из бота |
| P0 link/status | session on GET | ✅ routes `/browser/link/*` удалены |
| P0 Google OAuth | stub | ❌ |
| P0 bot token | generate-token | ❌ |

### P0. Fix `withdrawal_audit_logs` Schema Mismatch

**Статус:** ✅ Исправлено без миграции колонок — INSERT под 011 (см. `staff.js`).

Why: staff approve/reject flow can fail when audit insert references columns that migrations do not create.

Current code expects:

```sql
withdrawal_request_id
request_code
user_id
shop_slug
previous_status
next_status
action
reason_code
reason_note
actor_staff_id
actor_role
actor_ip
actor_user_agent
request_id
metadata_json
```

Migration `011` creates a different shape. Add migration `018_align_withdrawal_audit_logs.js`.

Recommended migration pattern:

```js
function columnExists(db, table, column) {
  return db.prepare(`PRAGMA table_info(${table})`).all()
    .some((row) => row.name === column)
}

function addColumn(db, table, column, definition) {
  if (!columnExists(db, table, column)) {
    db.exec(`ALTER TABLE ${table} ADD COLUMN ${column} ${definition}`)
  }
}

export function up(db) {
  addColumn(db, 'withdrawal_audit_logs', 'withdrawal_request_id', 'INTEGER')
  addColumn(db, 'withdrawal_audit_logs', 'user_id', 'INTEGER')
  addColumn(db, 'withdrawal_audit_logs', 'shop_slug', 'TEXT')
  addColumn(db, 'withdrawal_audit_logs', 'previous_status', 'TEXT')
  addColumn(db, 'withdrawal_audit_logs', 'next_status', 'TEXT')
  addColumn(db, 'withdrawal_audit_logs', 'actor_ip', 'TEXT')
  addColumn(db, 'withdrawal_audit_logs', 'actor_user_agent', 'TEXT')

  db.exec(`
    CREATE INDEX IF NOT EXISTS idx_withdrawal_audit_request_id
    ON withdrawal_audit_logs(withdrawal_request_id);
  `)
}
```

Alternative: change `staff.js` to match old schema. Better: align schema to richer audit model.

### P0. Fix Manager Password Hash Creation

**Статус:** ✅ `shops.js` / `staff.js` / `auth.js` → `src/lib/password.js` (2026-06-03: **Argon2id** для новых хешей; PBKDF2 + SHA-256 verify + upgrade on login).

### P0. Fix API Error Propagation

**Статус:** ✅ `public/js/utils/api.js` — `err.error`, `err.payload`, `err.status`.

Why: `login.js` checks `error.error`, but `api.js` throws a generic `Error`, so page-specific error branches do not work.

Current:

```js
if (!res.ok) {
  var text = await res.text()
  throw new Error('HTTP ' + res.status + ': ' + text.slice(0, 120))
}
```

Target:

```js
if (!res.ok) {
  var text = await res.text()
  var payload = null
  try {
    payload = JSON.parse(text)
  } catch (_) {
    payload = { error: 'HTTP_ERROR', message: text }
  }
  var err = new Error(payload.message || payload.error || ('HTTP ' + res.status))
  err.status = res.status
  err.error = payload.error
  err.payload = payload
  throw err
}
```

### P0. Move API Observability Middleware Before Route Mounts

**Статус:** ✅ Middleware перед `router.use('/shops', ...)`; `req.staff` для роли.

Why: in `src/routes/api/v1/index.js`, this block appears after route modules:

```js
router.use('/shops', shopRoutes)
router.use('/services', serviceRoutes)
router.use('/auth', disableStore, authRoutes)
...
router.use((req, res, next) => {
  console.log(`[Host: ${host}] [Role: ${role}] ${req.method} ${req.url} (ReqID: ${req.requestId})`);
  next();
});
```

Handled routes never reach it. Move observability before `router.use('/shops', ...)`, and use `logger`, not `console.log`.

### P0. Fix Admin Dashboard API Paths

**Статус:** ✅ `admin-dashboard.js` + `config.js` на странице.

Why: `/admin-panel` staff status/analytics/logs can call the wrong URL.

Current broken pattern:

```js
// public/js/pages/admin-dashboard.js
window.API.apiCall('/api/v1/staff/status')
```

With `public/js/utils/config.js`, `window.API_BASE` is `<origin>/api`, so the final URL becomes `/api/api/v1/staff/status`.

Target:

```js
window.API.apiCall('/v1/staff/status')
window.API.apiCall('/v1/staff/analytics')
window.API.apiCall('/v1/staff/logs')
```

Add a project convention:

- Use `window.API.apiCall('/v1/...')` when using `public/js/utils/api.js`.
- Use `fetch('/api/v1/...')` only when bypassing `window.API`.
- Never pass `/api/v1/...` into `window.API.apiCall()`.

### P1. CSRF For Web Cookie Sessions

**Статус:** ✅ **Implemented 2026-06-03** (см. §3.2.1, `src/lib/csrf.js`, `src/middleware/csrf.js`, `src/tests/csrf.session.test.js`).

Историческая проблема: глобальный CSRF ломал miniapp wheel (`POST /v1/wheel/spin` с `initData` без cookie CSRF). Решение — scoping ниже; dep `lusca` удалён из `package.json`.

### CSRF Hard Rule (do not violate)

CSRF protection applies ONLY to ambient cookie-session auth. It must NOT apply to Telegram-authenticated requests.

Why a naive global CSRF breaks the miniapp:

- `csrf_tokens.session_token_hash` binds the CSRF token to a web session row.
- The Telegram Mini App authenticates per-request via `X-Telegram-Init-Data` (`public/js/utils/api.js` -> `optionalCurrentUser` -> `req.authType = 'telegram'`), without sending a session-bound CSRF token.
- A global CSRF gate on `/api/v1/*` unsafe methods would therefore reject miniapp `POST /v1/wheel/spin`, `POST /v1/bonuses/redeem`, `POST /v1/bonuses/daily/claim`, etc. This is exactly the wheel-spin breakage symptom.
- Telegram initData is safe against classic CSRF on its own: it is a custom request header (cross-site HTML forms cannot set it) and it is HMAC-verified in `src/middleware/telegramAuth.js`.

Required scoping:

```js
function csrfRequired(req) {
  // Skip safe methods.
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) return false
  // Skip Telegram initData / miniapp auth: not ambient, HMAC-verified, custom header.
  if (req.authType === 'telegram') return false
  if (req.headers['x-telegram-init-data']) return false
  // Enforce only for ambient cookie-session web auth.
  return req.authType === 'session'
}
```

Implemented (2026-06-03):

- [x] Token on `createBrowserSession` + cookie `syndicate_csrf`.
- [x] EJS `partials/runtime-head.html` + `renderWebPage()` (не на `app.sndct.cc`).
- [x] `api.js` + `profile.js` + `csrf_token` в auth responses.
- [x] Middleware `csrfProtection(db)` on `/api/v1`.
- [x] Tests: web POST без токена → 403; с токеном → 200; initData header → CSRF skip.
- [ ] E2E на проде: spin/redeem на miniapp после deploy (ручная проверка).

### P1. CORS No-Origin Policy

Current:

- No-Origin unsafe methods are logged in development only.
- They are not blocked.

Decision needed:

- If form posts without Origin must be allowed for non-browser clients, protect via CSRF.
- If not needed, block unsafe no-Origin in production.

Recommended once CSRF exists:

```js
if (!req.headers.origin && ['POST', 'PUT', 'DELETE', 'PATCH'].includes(req.method)) {
  return res.status(403).json({ error: 'ORIGIN_REQUIRED' })
}
```

### P1. Rate Limit Password Reset

**Статус:** ✅ `forgotPasswordLimiter` (5 / 15 min) и `resetPasswordLimiter` (10 / 15 min) в `auth.js` (2026-06-03). FE: `forgot-password.js`, `reset-password.js` обрабатывают 429.

### P1. Fix Forgot Password Copy

`forgot-password.html` says in form intro: "Ссылка действительна 15 минут." Success state says 30 minutes. Backend token TTL is 30 minutes. Make all copy 30 minutes.

### P1. Remove Hardcoded Turnstile Secret

**Статус:** ✅ `validateTurnstileToken()` удалена (2026-06-02). При включении CAPTCHA — только через env.

### P1. Run And Fix Project Checks

**Статус (2026-06-03):** `npm run verify` = check + eslint + `test:ci` — **19 passed**. Тесты: `staff.withdrawals`, `password.lib` (argon2), `csrf.session`, `wheel.spin`, `auth.email-register`, `mailer.sendgrid-headers`, `domain_split`, `api`.

Required after fixes:

```bash
npm run verify
npm run build:css
```

---

## 6. Roadmap

### Phase 1 - Stabilize Runtime And Broken Contracts

Goal: make existing flows reliable before new product work.

Tasks:

1. ~~Add migration `018_align_withdrawal_audit_logs.js`.~~ → audit fix без ALTER; `018_apply_missing_indexes.js` для индексов.
2. [x] Test staff withdrawal approve/reject — `src/tests/staff.withdrawals.test.js`.
3. [x] Fix `public/js/utils/api.js` error propagation.
4. [x] Fix `admin-dashboard.js` paths + `config.js`.
5. [x] Move API observability middleware before route mounts.
6. [ ] Replace `console.log/error` in route modules with `logger`.
7. [ ] Fix forgot password TTL copy.
8. [x] `npm run verify` (локально).

Exit criteria:

- [x] Staff approve/reject без SQL error на audit.
- [ ] Login UI ветки `EMAIL_NOT_VERIFIED` / `CREDENTIALS_EXPIRED` (нужна ручная проверка UI).
- [x] API logs host/role для handled routes.
- [x] Jest smoke green; [ ] стабильный teardown withdrawals test.

### Phase 2 - Security Hardening

Goal: remove false production-readiness and close obvious auth/security gaps.

Tasks:

1. [x] Shared `src/lib/password.js` with Argon2id for new hashes (2026-06-03).
2. [x] Migration path: PBKDF2 + SHA-256 verify + `needsPasswordUpgrade` on login.
3. [x] `auth.js`, `staff.js`, `shops.js` use `hashPassword` / `verifyPasswordAsync`.
4. [x] CSRF for web unsafe methods (§3.2.1).
5. [x] Reset/forgot rate limiters.
6. [ ] Decide and enforce no-Origin unsafe request policy (после CSRF — опционально `ORIGIN_REQUIRED`).
7. [x] Turnstile hardcoded secret removed.
8. [x] Tests: CSRF session, password argon2, domain_split 421, wheel spin.

Exit criteria:

- [x] New passwords are Argon2id.
- [x] Legacy users/staff login + upgrade.
- [x] Web cookie-session POST CSRF-protected; miniapp initData exempt.
- [ ] CORS no-Origin policy decided.

### Phase 3 - Service Layer Extraction

Goal: stop growing route files as god objects.

Start with these modules:

```text
src/services/
  authService.js
  staffService.js
  shopService.js
  withdrawalService.js
  bonusService.js
src/repositories/
  userRepository.js
  shopRepository.js
  withdrawalRepository.js
```

Extraction order:

1. `withdrawalService.resolveWithdrawal()`
2. `bonusService.createRedeemRequest()`
3. `shopService.create/update/delete`
4. `authService.register/login/verify/reset`

Rules:

- Routes validate transport payloads and call services.
- Services own transactions and business invariants.
- Repositories own SQL.
- Tests should target services first, routes second.

### Phase 4 - Admin Panel / Control Department

Goal: turn constructor/staff panel into operational console.

Current gaps:

- `constructor.js` still uses `window.prompt` / `window.confirm` (reject/delete).
- No SLA badges; withdrawal list — status filter only.
- [x] Audit INSERT works (schema 011); [x] manager-scoped cabinet + PUT whitelist; [x] shop editor partial + modal UX (2026-06-02).
- Admin dashboard API paths fixed; role-aware analytics — TBD.

Tasks:

1. Replace prompt/confirm with Tailwind modal components.
2. Add reject modal:
   - reason code select;
   - reason note textarea;
   - typed confirmation for high-risk actions.
3. Add queue filters:
   - status;
   - shop;
   - amount range;
   - age bucket;
   - manager scope.
4. Add SLA badges:
   - `<5m`;
   - `5-30m`;
   - `30m+`;
   - `overdue`.
5. Add analytics endpoint fields:
   - pending count;
   - median resolution time;
   - reject ratio;
   - pending by shop;
   - oldest pending.
6. Add audit view from `withdrawal_audit_logs`.

Recommended staff queue response extension:

```json
{
  "success": true,
  "scope": "all",
  "role": "admin",
  "requests": [
    {
      "id": 123,
      "request_code": "1716800000000-12345",
      "shop_slug": "shop-x",
      "amount": 250,
      "status": "pending",
      "created_at": "2026-06-02T17:00:00.000Z",
      "age_seconds": 812,
      "sla_bucket": "5-30m",
      "risk_flags": ["new_user", "repeat_redeem"]
    }
  ]
}
```

### Phase 5 - User Flow / Product Flow

Goal: make web and miniapp feel intentional, not just technically split.

Web flow (`sndct.cc`):

1. Landing/services.
2. Login/register via email.
3. Email verification.
4. Bonuses/services/profile.
5. Optional Telegram linking.
6. Redeem status history.

Miniapp flow (`app.sndct.cc`):

1. Telegram WebApp opens.
2. InitData auth attempt.
3. If auth ok: bonuses/services.
4. If auth missing/invalid: Telegram-specific recovery panel, not web email-first UI.
5. No unnecessary "login via Telegram" button inside Telegram.

Must add:

- User-facing withdrawal history endpoint.
- User-facing redeem status UI.
- Empty/loading/error states for bonuses/profile/services.
- Redirect preservation after login.
- Better email verification resend flow.

Recommended endpoint:

```http
GET /api/v1/bonuses/redeem/history?status=pending
```

Response:

```json
{
  "success": true,
  "requests": [
    {
      "id": 123,
      "code": "1716800000000-12345",
      "shop_slug": "shop-x",
      "amount": 250,
      "status": "pending",
      "created_at": "2026-06-02T17:00:00.000Z",
      "resolved_at": null,
      "reason_code": null,
      "reason_note": null
    }
  ]
}
```

### Phase 6 - Design System

Goal: reduce one-off Tailwind soup and make pages consistent.

Current state:

- Tailwind v4 works.
- DaisyUI installed.
- `main.css` still owns shell/navigation/theme.
- `shop-card.css`, `popup.css`, `wheel.css` still used.
- UI uses repeated arbitrary colors and large inline class strings.

Do not remove yet:

- `public/css/global/main.css`
- `public/css/components/wheel.css`
- `public/css/components/popup.css`
- `public/css/components/shop-card.css`

Recommended next design layer:

1. Define semantic Tailwind tokens in `tailwind.input.css`.
2. Replace repeated `[#...]` utilities with semantic utilities where possible.
3. Create HTML snippet conventions, not JS component framework yet.
4. Standardize:
   - page shell;
   - cards;
   - buttons;
   - inputs;
   - badges;
   - modal;
   - table;
   - toast/error blocks.

Canonical snippets:

```html
<div class="bg-bg-tertiary border border-white/10 rounded-xl p-5">
  ...
</div>
```

```html
<button class="py-2 px-4 bg-brand-teal hover:bg-brand-teal-dark text-bg-primary font-bold rounded-lg transition-colors disabled:opacity-50 disabled:cursor-not-allowed">
  Сохранить
</button>
```

```html
<input class="w-full px-3 py-2 bg-bg-secondary border border-white/10 rounded-lg text-text-light placeholder-white/40 focus:outline-none focus:border-brand-teal transition-colors">
```

### Phase 7 - Observability / Ops

Goal: deploy and debug without guessing.

Tasks:

1. Replace route-level `console.*` with `logger`.
2. Add request log context:
   - requestId;
   - runtimeHost;
   - runtimeMode;
   - authType;
   - userId/staffId when available;
   - route;
   - status;
   - latency.
3. Add metrics endpoints or structured logs for:
   - auth failures by reason;
   - rate limit hits;
   - queue depth;
   - oldest pending withdrawal;
   - resolve latency;
   - Redis availability;
   - DB busy timeout events.
4. Add startup smoke checks for:
   - `/health` on `sndct.cc`;
   - `/health` on `app.sndct.cc`;
   - `/api/v1/health`;
   - DB migration status.

---

## 7. Testing Strategy

Current tests:

- `src/tests/api.test.js` covers `/health`, `/api/v1/health`, `/api/v1/auth/status`.
- `src/tests/domain_split.test.js` covers host -> runtime mode and unknown host rejection.

Missing high-value tests:

1. `auth.web.test.js`
   - register creates unverified user;
   - verify email creates session;
   - login rejects unverified;
   - forgot password does not leak account existence;
   - reset password consumes token.
2. `auth.cookies.test.js`
   - `sndct.cc` sets `syndicate_web_sid`;
   - `app.sndct.cc` sets `syndicate_mini_sid`;
   - logout clears both split cookies.
3. `staff.withdrawals.test.js`
   - manager can see own shop only;
   - manager cannot resolve foreign shop;
   - admin can resolve any;
   - reject refunds balance;
   - audit row is written.
4. `shops.manage.test.js`
   - admin can create shop;
   - manager can update assigned shop;
   - manager cannot update foreign shop;
   - cache invalidates on write.
5. `security.csrf.test.js`
   - web unsafe method without CSRF fails after CSRF implementation;
   - Telegram initData policy remains valid.

Minimum test commands before handing to production:

```bash
npm run check
npm test
npm run build:css
```

---

## 8. Implementation Notes For Next Agent

Work order:

1. Do not start with refactor.
2. First fix the schema mismatch for `withdrawal_audit_logs`.
3. Add a regression test that fails before the migration and passes after.
4. Fix `api.js` error propagation.
5. Fix password hashing path in `shops.js`.
6. Then move into CSRF/security.
7. Only after P0/P1 work extract service layer.

Files to read before touching each area:

Runtime/domain:

- `src/server.js`
- `src/middleware/runtimeMode.js`
- `public/js/utils/runtime.js`
- `src/tests/domain_split.test.js`

Auth:

- `src/routes/api/v1/auth.js`
- `src/middleware/currentUser.js`
- `src/middleware/telegramAuth.js`
- `public/js/pages/login.js`
- `public/js/utils/api.js`

Staff/withdrawals:

- `src/routes/api/v1/staff.js`
- `src/middleware/staffAuth.js`
- `src/routes/api/v1/bonuses.js`
- `public/js/pages/constructor.js`
- `src/config/migrations/008_staff_and_withdrawals.js`
- `src/config/migrations/011_web_auth_and_security.js`

Shops/constructor:

- `src/routes/api/v1/shops.js`
- `public/js/pages/constructor.js`
- `DATA_FLOW.md`

Tailwind/design:

- `public/css/tailwind.input.css`
- `tailwind.config.js`
- `postcss.config.js`
- `src/views/*.html`
- `public/css/global/main.css`

---

## 9. Known Sharp Edges

- `src/config/database.js` runs migrations on import. Tests importing `app` can mutate the configured DB.
- `src/config/migrations/017_add_database_indexes.js` exports default `migrate()` instead of `up(db)`, so the migration runner will import it but not execute it unless this has been handled manually. The doc claims indexes are added; verify in DB before relying on it.
- `src/routes/api/v1/auth.js` contains duplicate `POST /logout`.
- `src/routes/api/v1/auth.js` mixes public auth, web auth, admin legacy auth, profile endpoints, and helper functions in one file.
- `src/routes/api/v1/staff.js` and `src/routes/api/v1/shops.js` duplicate password hashing concerns.
- `src/routes/api/v1/shops.js` allows `application/octet-stream` uploads if extension is allowed; acceptable only if downstream serving policy is safe.
- Redis `invalidatePattern()` uses `KEYS`, fine for small dev, bad for large production keyspaces.
- `public/js/pages/admin-dashboard.js` passes `/api/v1/...` into `window.API.apiCall()`, causing `/api/api/v1/...` because `window.API_BASE` already includes `/api`.
- `.github/workflows/*` is absent, so there is no repository CI pipeline for `npm test` / `npm run check`.
- `.env.example` contains weak illustrative secrets (`ADMIN_PASSWORD=246810`, placeholder `JWT_SECRET`), and `JWT_SECRET` is not used by the current JS auth path.
- Google OAuth routes in `auth.js` can run in stub mode with dummy config; disable or feature-flag this in production.
- `src/views/work.html` is outside the shared runtime shell pattern: it has Tailwind only and no `window.__APP_RUNTIME__`.
- `public/js/framework/telegram.js` falls back to browser `alert/confirm`. For Telegram-specific wrapper this is less severe, but still UX debt.
- `public/js/pages/constructor.js` uses `window.prompt` and `window.confirm` in operational flows.
- `forgot-password.js` uses direct `fetch`, not `window.API.apiCall`; this means future CSRF header logic must cover direct calls or migrate it.
- `npm run check` currently fails inside `src/scripts/check-git-secrets.sh`: line with `local candidate=...` is executed outside a function.

---

## 10. Verification Run

Commands attempted after creating this file:

```bash
npm test
npm run check
```

Observed:

- `npm test` printed `Test Suites: 2 passed, 2 total` and `Tests: 7 passed, 7 total`, then the shell command did not exit cleanly before being aborted. Treat the tests as functionally passing but investigate lingering open handles from `--detectOpenHandles`.
- `npm run check` failed before JS/Python checks because `src/scripts/check-git-secrets.sh` uses `local` outside a function.
- `npm run build:css` was not run because no CSS was changed while producing this state document.

Fix before relying on check pipeline:

```bash
# in src/scripts/check-git-secrets.sh
- local candidate="${line#add \'}"
+ candidate="${line#add \'}"
```

---

## 11. Department Recommendations

### Design Department

- Freeze current cyber/dark/teal visual language, but formalize tokens.
- Stop adding arbitrary hex utilities in new pages.
- Keep `main.css` as shell layer until a full design-system pass.
- Convert repeated cards/buttons/inputs into documented snippets first.
- Replace native dialogs with project modals.
- Build consistent empty/loading/error states.

### Flow / Userflow Department

- Treat web and miniapp as separate entry experiences, not separate products.
- Web default: email login/register/profile/password recovery.
- Miniapp default: Telegram auth and Telegram-safe recovery.
- Add redeem history/status for users.
- Add resend verification.
- Add safe redirect preservation across login/register/verify.

### Admin Panel / Control Department

- Make `/constructor` an operations console, not just shop CRUD.
- Withdrawal queue is core business flow — approve/reject работает; UI карточки + модалка (2026-06-02).
- [x] Scoped manager UX: `/shop-cabinet`, whitelist PUT, форма без admin-полей.
- [ ] SLA, filters, notes, audit view в UI.
- [ ] Remove prompt/confirm from resolve/delete flows.
- [x] `/admin-panel` vs `/constructor` — разные страницы; API paths исправлены.

### Development Department

- P0 schema + tests first.
- P1 security: ~~CSRF, Argon2id, rate limit reset~~ ✅; CORS no-Origin policy — open.
- Then service layer extraction.
- Keep route compatibility while moving internals.
- Every production-sensitive change gets regression tests.
- Do not remove old docs until this file has been reviewed and accepted.

---

## 12. Final Checklist

Before next production deploy:

- [x] Fix `withdrawal_audit_logs` INSERT (`staff.js` → schema 011).
- [x] Verify withdrawal approve/reject (test + prod reject sample).
- [x] Fix API error propagation.
- [x] Fix `/admin-panel` API paths + `config.js`.
- [x] Fix manager password hash (`password.js` / PBKDF2).
- [x] Argon2id (`lib/password.js`) + upgrade on login.
- [x] Forgot/reset rate limiters.
- [x] CSRF web-only (§3.2.1).
- [x] Wheel spin atomic transaction (`wheel.js` + `wheel.spin.test.js`).
- [x] `resolveDatabasePath` для `database.js` + `dbAsyncWorker.js`.
- [x] Dead code cleanup (§14.4).
- [x] Mailer SendGrid clicktrack off (`mailer.js` + test).
- [x] Web login UX: bot code → `POST /web/bot-login` (`bonuses.js`, `login.js`).
- [x] Observability middleware order.
- [ ] Replace critical `console.*` with `logger` (частично: `wheel.js`, `index.js` API log).
- [x] `npm run verify` locally.
- [x] `npm run build:css` (в verify/check pipeline).
- [x] Docs: этот файл обновлён 2026-06-03 (§14 debug session).
- [x] CI workflow `.github/workflows/ci.yml`.
- [x] `018_apply_missing_indexes.js`; 017 `export function up`.
- [x] Turnstile hardcoded secret removed.
- [x] `link/complete` POST для сессии; GET `link/status` read-only.
- [x] Google OAuth stub gated (`isGoogleOAuthLive`).
- [x] Stop leaking manager passwords in `manage/list`.
- [x] `/browser/link/*` удалён; web login через `POST /web/bot-login`.
- [x] `config.js` + `authManager` on constructor/shop-cabinet/admin-dashboard.
- [x] Constructor: `shop-editor-form.html` partial, manager scope, category removed from editor.
- [x] (§13 P1) `dbAsyncWorker.js` → `resolveDatabasePath.js`.
- [x] (§13 P2) Dead files removed (§14.4); **`csrf_tokens` table — живая**, используется CSRF.
- [ ] (§13 P2) Drop optional dead tables `user_achievements` (после бэкапа).

The project is implementable and salvageable. The immediate priority is not another big visual rewrite; it is aligning schema, auth/security claims, and operational queue behavior with the code that already exists.

---

## 13. Code & DB Audit (4 агента + живая БД, 2026-06-02 21:00)

Этот раздел — результат сплошного аудита: API endpoints, БД-слой, фронтенд-вызовы, cross-cutting код. Источник схемы — **живая `syndicate.db`** (снято через `sqlite3`, не миграции). Где код расходится с живой схемой — побеждает БД.

**После аудита (вечер 2026-06-02):** код и тесты изменены по таблице §5; статусы P0 ниже помечены ✅/❌. Snapshot row counts в §13.1 — на момент 21:00, не переснимался.

### 13.0. Метод

- Снят дамп `syndicate.db`: таблицы, колонки, индексы, row counts, `_migrations`.
- 4 параллельных агента (read-only) + ручная верификация двух P0 по строкам.
- Все находки — с `file:line`. P0 = ломает прод/безопасность, P1 = латентные баги/долг, P2 = чистота/DX.
- Bugfix pass: `npm run verify`, правки `staff.js`/`shops.js`/FE staff pages, восстановление constructor UI.

### 13.1. Живая схема (snapshot)

Таблицы (row counts): `users(12)`, `shops(9)`, `services(5)`, `bonuses(7)`, `wheel_spins(32)`, `referrals(0)`, `achievements(0)`, `user_achievements(0)`, `auth_sessions(1)`, `auth_link_codes(8)`, `ui_events(610)`, `admin_sessions(1)`, `staff_accounts(10)`, `staff_sessions(37)`, `bonus_withdrawal_requests(2)`, `web_oauth_states(0)`, `csrf_tokens(41)`, `auth_audit_logs(5)`, `withdrawal_audit_logs(0)`, `password_reset_tokens(3)`.

`_migrations`: 001–017 на snapshot; добавлена **018** `apply_missing_indexes` (повтор индексов 017). На старых БД 017 мог быть «пустым» — 018 обязателен после deploy.

API-поверхность: **58 API endpoints** + 12 page routes + 2 `/health`. Из них **7 P0, 10 P1, 8 P2** на уровне маршрутов.

### 13.2. P0 — критично (ломает прод / безопасность)

**P0-1. `withdrawal_audit_logs`: INSERT ≠ живая схема → resolve withdrawal падает в 500.** ✅ **Fixed 2026-06-02** — INSERT под 011; prod reject + audit проверены. (Было: подтверждено лично + 2 агентами + `sqlite3`)
- Живая таблица (из `011_web_auth_and_security.js:133-149`): `withdrawal_id NOT NULL, request_code, actor_staff_id, actor_role, action, reason_code, reason_note, ip_address, user_agent, request_id, metadata_json`.
- Код `staff.js:461-498` пишет: `withdrawal_request_id, request_code, user_id, shop_slug, previous_status, next_status, action, reason_code, reason_note, actor_staff_id, actor_role, actor_ip, actor_user_agent, request_id, metadata_json`.
- Колонок `withdrawal_request_id/user_id/shop_slug/previous_status/next_status/actor_ip/actor_user_agent` НЕТ; `withdrawal_id NOT NULL` не заполняется. `prepare()` → `no such column`. INSERT внутри `db.transaction()` → **полный rollback** (статус, refund, бонусы). Подтверждение: `withdrawal_audit_logs = 0` строк при 2 заявках.
- **Fix (вариант A, рекомендуется, без миграции):** переписать INSERT в `staff.js` под схему 011 — колонки `(withdrawal_id, request_code, actor_staff_id, actor_role, action, reason_code, reason_note, ip_address, user_agent, request_id, metadata_json)`, а `withdrawal_request_id/user_id/shop_slug/previous_status/next_status` сложить в `metadata_json`. `withdrawal_id = requestRow.id`.

**P0-2. Миграция 017 помечена applied, но НЕ исполнялась → индексов нет.** ✅ **Fixed** — `018_apply_missing_indexes.js` + 017 переписан с `export function up(db)`.
- `migrate.js:39-43` вызывает только `up(db)`. `017_add_database_indexes.js:10` экспортирует `export default function migrate()` + тянет глобальный `database.js` при импорте.
- `up === undefined` → тело не выполняется, но `INSERT INTO _migrations` (`migrate.js:46`) всё равно срабатывает. В живой БД индексов из 017 нет (есть только `idx_users_email_unique/google_id_unique/username_unique` из 011/014 — это другие).
- **Fix:** новая `018_apply_missing_indexes.js` с `export function up(db)` (скопировать `CREATE INDEX IF NOT EXISTS` из 017, кроме дублей) — **не** удаляя строку 017 из `_migrations`. Параллельно переписать 017 на `up(db)` для чистых установок.

**P0-3. Staff manager-пароли хешируются голым SHA-256 (без соли).** ✅ **Fixed** — `shops.js` → `hashPasswordStrong` (`src/lib/password.js`). Legacy SHA-256 по-прежнему verify при login.

**P0-4. Захардкоженный Turnstile secret в репозитории.** ✅ **Fixed** — `validateTurnstileToken` удалена; секрет в repo убран (ротация в Cloudflare — ops).

**P0-5. `/browser/link/*` miniapp pairing.** ✅ **Removed** — маршруты удалены; web вход через бот `/login` + `POST /api/v1/auth/web/bot-login` (`bonuses.js`, `login.js`).

**P0-6. Google OAuth callback — stub без обмена code→token.** ❌ **Open** — gated `isGoogleOAuthLive()`; при включении без live credentials — 503/redirect `oauth_disabled`.

**P0-7. Утечка паролей менеджеров в API-ответе.** ✅ **Fixed** — `manage/list` отдаёт только `manager_login` + shop fields, без plaintext password.

**P0-8. `POST /auth/bot/generate-token` — полный контроль над `user` в body при знании bot-token, без rate limit.** `auth.js:291-306` — upsert любого telegram-пользователя. Плюс bot-token гоняется в теле POST (`bot.py:72` + plain compare `auth.js:294`).

**P0-9 (FE). `admin-dashboard.js` → `/api/api/v1`.** ✅ **Fixed** — `/v1/staff/...` + `config.js`.

**P0-10 (FE). `link/start`.** ✅ **Closed** — flow заменён на `POST /v1/auth/web/bot-login` (6-digit code from bot).

**P0-11 (FE). Staff pages без `config.js`.** ✅ **Fixed** — constructor, shop-cabinet, admin-dashboard. Доп.: восстановлен `shop-editor-form.html`; JS modal/`setVisible`; cache `?v=20260602e`.

### 13.3. Dead tables / dead columns (живая схема vs код)

**Мёртвые таблицы:**
| Таблица | Статус | Доказательство |
|---|---|---|
| `csrf_tokens` | **живая** | reader/writer в `src/lib/csrf.js` + `createBrowserSession` / revoke (2026-06-03); старые строки-сироты можно чистить ops |
| `user_achievements` | мёртвая + битый FK | 0 обращений; FK `REFERENCES achievements(type)`, а в `achievements` колонки `type` нет (есть `achievement_type`) |
| `withdrawal_audit_logs` | живая | INSERT исправлен (P0-1 ✅); строки появляются после resolve/reject |
| `achievements` | полу-мёртвая | только READ `auth.js:699`; нет writer'а, прогресс не ведётся |

Живые, но пустые (просто не использовали фичу, НЕ мёртвые): `web_oauth_states` (код в `auth.js:519-544`), `referrals` (INSERT `telegramAuth.js:194`).

**Дублирующая модель достижений:** `achievements` (per-user, 001) vs каталог + `user_achievements` (010). 010 не пересоздал `achievements` из-за `IF NOT EXISTS`. Нужно выбрать одну модель.

**Мёртвые/неиспользуемые колонки (нет ни read, ни write в бизнес-логике):**
`users.total_spent` (read-only, никогда не UPDATE), `users.referrer_id` (FK на `users(telegram_id)` — семантически неверно, не пишется), `users.status` (read через `SELECT *`), `shops.commission_rate` (только схема), `shops.rating`/`review_count` (только ORDER BY, нет writer), `services.secondary_action_url`/`secondary_action_label`, `wheel_spins.spin_duration_ms`, `referrals.reward_claimed` (read-only, всегда 0), `web_oauth_states.code_verifier_hash`/`user_id`, `staff_sessions.last_seen_at`, `admin_sessions.last_seen_at`, `achievements.completed_at`/`reward_claimed`/`updated_at`/`created_at`.

### 13.4. Дублирующаяся логика (консолидировать в `src/lib/`)

**Backend хелперы (копии в 4-5 файлах):**
| Хелпер | Копии (file:line) |
|---|---|
| `parseCookies` | `auth.js:131`, `staff.js:14`, `staffAuth.js:7`, `currentUser.js:27` |
| `hashToken` (SHA-256) | `auth.js:146` (+`hashOpaque` 119), `staff.js:28`, `staffAuth.js:21`, `currentUser.js:16` |
| `safeEqual` | `auth.js:161`, `staff.js:64` |
| password hashing | `auth.js:82` (pbkdf2), `staff.js` + `shops.js` → `lib/password.js` (pbkdf2+legacy verify) |
| `nowIso`/`addMinutes`/`nowPlusMinutes` | `auth.js:54,56,115,150`, `telegramAuth.js:18`, `currentUser.js:7`, `staff.js:113` |
| `parseJson` | `bonuses.js:16`, `shops.js:60`, `services.js:7` |
| `getAdminSession` | `auth.js:168`, `staffAuth.js:25` (почти 1:1) |
| spin-elapsed SQL | `user.js:11-21`, `wheel.js:8-16` (идентичный запрос) |

**Целевые модули:** `src/lib/cookies.js`, `src/lib/crypto.js`, `src/lib/password.js` (единый источник для users+staff+shops, pbkdf2/argon2id + verify legacy), `src/lib/datetime.js`, `src/lib/sessions.js`.

**Frontend дубли:** `isMiniApp()` (вместо `utils/runtime.js`) в `bonuses.js`, `authManager.js`, `login.js`, `profile.js`; `escapeHtml`/`escapeAttr` в 5 файлах; HTTP: `utils/api.js` + `constructor.js` local `request()` (~~`framework/api.js`~~ удалён); два staff-login UI (`constructor.js` + `settingsManager.js`).

### 13.5. Мёртвый код

**Удалено 2026-06-03 (§14.4):** `errorHandler.js`, `redisClient.js`, `bonusWorker.js`, `reproduce_register.js`, `build-css.js`, `public/js/framework/*`, `public/js/pages/work.js`, `public/js/utils/cache.js` (и `<script>` из `bonuses.html`), dep `lusca`.

**Всё ещё мёртвое / долг:**
- `getLegacyManagerPassword` (`shops.js`) — если не вызывается.
- `seedAchievements()` body только log; мёртвые exports `telegramAuth.js`; `bot.py` `run()`; env `JWT_SECRET`.
- Таблица `user_achievements` — мёртвая схема (§13.3).

**Исправлено:** ~~дубль `POST /auth/logout`~~; ~~`argon2` unused~~; ~~`lusca`~~.

### 13.6. Латентные баги и bad patterns (P1)

- ~~`utils/api.js` без парсинга JSON~~ ✅ fixed; ветки `login.js` (`EMAIL_NOT_VERIFIED`/`CREDENTIALS_EXPIRED`) — проверить вручную в UI.
- `forgot-password.js:31`, `profile.js:11`, `authManager.js:12` — игнор `res.ok` (успех UI при 4xx/5xx); `bonuses.js:356` глотает ошибку polling.
- ~~`dbAsyncWorker.js` хардкод пути~~ ✅ `config/resolveDatabasePath.js`.
- ~~`wheel.js` spin race~~ ✅ `syncDailySpinInTx` + cooldown + deduct + prize в одной `db.transaction()`; test `wheel.spin.test.js`.
- ~~`ensureBootstrapStaff` лишний UPDATE~~ ✅ `verifyPasswordAsync` + upgrade только при `needsPasswordUpgrade` / неверном hash.
- `redis.js:69` — `KEYS` в `invalidatePattern` (O(N), блокирует Redis).
- Несогласованные контракты ошибок: `{errors:[]}` (validator) vs `{error}` vs `{success:false,error}` (user routes) vs staff без `message`.
- Валидации нет на: `forgot-password`, `staff/login` limiter, `bonuses/claim/:id`, `POST /ui/events`, `GET /ui/summary` без auth. ~~`PUT /shops/:slug` manager меняет всё~~ — manager whitelist на cities/features/links/visual (2026-06-02).
- `console.*` вместо `logger` в `staff.js`, `bonuses.js`, `shops.js`, `user.js`, ...; частично: `wheel.js`, `routes/api/v1/index.js` → `logger`.
- `constructor.js:444,463,663` — `window.prompt/confirm` в прод-флоу reject/delete.
- `bot.py:87` MarkdownV2 без escape (6-значный code может сломать parse_mode); `telegram_bot/ui.py:33-34` обещает команды `/bonus`/`/manager`, которых нет.

### 13.7. Сводная матрица приоритетов (этот аудит)

| P0 (прод/безопасность) | P1 (латентные/долг) | P2 (чистота/DX) |
|---|---|---|
| ~~audit INSERT~~ ✅ | dbAsyncWorker хардкод пути | framework/*, work.js |
| ~~017 indexes~~ ✅ | ~~wheel spin гонка~~ ✅ | prompt/confirm в constructor |
| ~~SHA-256 shops~~ ✅ | redis KEYS invalidate | logger vs console.* |
| ~~Turnstile secret~~ ✅ | ~~bootstrap hash~~ ✅ | dead columns |
| ~~link/* removed~~ ✅ | res.ok игнор на фронте | achievements |
| Google OAuth stub ❌ | контракты ошибок | |
| ~~password leak~~ ✅ | ~~rate limit reset~~ ✅ | |
| bot/generate-token ❌ | | |
| ~~admin-dashboard~~ ✅ | | |
| ~~CSRF web~~ ✅ | | |
| ~~Argon2~~ ✅ | | |
| ~~config.js staff~~ ✅ | | |

### 13.8. Рекомендованные миграции

```sql
-- 018_apply_missing_indexes.js  (export function up(db))
--   скопировать CREATE INDEX IF NOT EXISTS из 017, кроме дублей
--   (idx_admin_sessions_token_hash — уже 007; idx_withdrawals_status — уже 008).

-- 019_cleanup_dead_schema.js  (опционально, после бэкапа)
-- НЕ удалять csrf_tokens — используется CSRF (2026-06-03)
DROP TABLE IF EXISTS user_achievements;
-- ALTER TABLE users DROP COLUMN referrer_id;   -- после проверки данных

-- P0-1: выполнено в staff.js (без ALTER audit table).
```

> §13.2 snapshot — на 21:00 (counts не переснимались). Актуальные код-изменения — §14 (2026-06-03).

---

## 14. Debug Session Changelog (2026-06-02 вечер — 2026-06-03)

Полный список правок из сессии отладки/харденинга. Источник истины для deploy и code review.

### 14.1. Security & Auth

| Тема | Файлы | Что сделано |
|------|--------|-------------|
| **CSRF web-only** | `lib/csrf.js`, `middleware/csrf.js`, `currentUser.js`, `routes/api/v1/index.js`, `server.js` (`renderWebPage`), `views/partials/runtime-head.html`, `public/js/utils/api.js`, `profile.js`, `index.js`, `auth.js` | Токен при сессии; cookie `syndicate_csrf`; middleware только `authType===session'`; miniapp `initData` exempt; тест `csrf.session.test.js` |
| **Argon2id** | `lib/password.js`, `auth.js`, `staff.js`, `shops.js` | Новые пароли Argon2; verify PBKDF2+SHA256; upgrade on login |
| **Rate limits** | `auth.js`, `forgot-password.js`, `reset-password.js` | forgot 5/15min, reset 10/15min; UI 429 |
| **DATABASE_PATH** | `config/resolveDatabasePath.js`, `database.js`, `dbAsyncWorker.js` | Единый путь sync/async SQLite |
| **Mailer** | `lib/mailer.js`, `mailer.sendgrid-headers.test.js` | SendGrid: clicktrack/opentrack/ganalytics off; `clicktracking="off"` на ссылках; plain-text URL; `APP_CANONICAL_URL` |
| **Auth flow** | `auth.js`, `bonuses.js`, `login.js` | Удалён `/browser/link/*`; web login = bot `/login` + `POST /web/bot-login` |
| **Duplicate logout** | `auth.js` | Убран второй `POST /logout` |

### 14.2. Staff / Constructor UI (2026-06-02, сохранено в сессии)

| Тема | Файлы |
|------|--------|
| Восстановлен редактор | `views/partials/shop-editor-form.html` (был пустой stub) |
| Constructor JS | `public/js/pages/constructor.js` — `setVisible`, modal scroll, cards, manager PUT whitelist |
| Manager scope | `shops.js` — PUT только cities/features/links/visual; без category в форме |
| Страницы | `constructor.html`, `shop-cabinet.html` (`data-dashboard-mode="manager"`), cache `?v=20260602f/g` |
| Withdrawals staff | `staff.js` audit INSERT 011; `staff.withdrawals.test.js` |
| Admin FE | `admin-dashboard.js` → `/v1/staff/...`; `config.js` на staff pages |

### 14.3. Wheel & Backend

| Тема | Файлы | Что сделано |
|------|--------|-------------|
| **Spin race** | `wheel.js` | `syncDailySpinInTx` + cooldown + deduct + insert + award в одной `db.transaction()`; 409 при concurrent deduct |
| **SQLite concurrency** | `database.js` | `journal_mode=WAL`, `busy_timeout=5000` |
| **Test** | `wheel.spin.test.js` | Второй spin при `spin_count=0` → 400 `NO_SPINS`; один row в `wheel_spins` |

### 14.4. Dead Code Cleanup (2026-06-03)

**Удалённые файлы:**

- `src/lib/errorHandler.js`
- `src/config/redisClient.js`
- `src/workers/bonusWorker.js`
- `reproduce_register.js`
- `build-css.js`
- `public/js/framework/api.js`, `public/js/framework/telegram.js`
- `public/js/pages/work.js`
- `public/js/utils/cache.js`

**package.json:** удалён `lusca` (CSRF через свой middleware).

### 14.5. Tests & CI (19 total)

`api.test.js`, `domain_split.test.js`, `password.lib.test.js`, `csrf.session.test.js`, `wheel.spin.test.js`, `auth.email-register.test.js`, `staff.withdrawals.test.js`, `mailer.sendgrid-headers.test.js`.

```bash
npm run verify   # check + lint + test:ci
```

### 14.6. Deploy Notes

1. Hard refresh на `https://sndct.cc` после выкладки (CSRF token в HTML / `auth/status`).
2. Miniapp `https://app.sndct.cc` — без CSRF; проверить wheel spin + redeem.
3. Существующие пользователи с PBKDF2 — Argon2 после следующего логина.
4. SendGrid: Link Branding на `url8682.sndct.cc` снят в dashboard (вне кода).
5. Миграция `018` на проде если ещё не применена.

### 14.7. Still Open (не входило в «закрыто» сессии)

- Google OAuth stub (real token exchange or keep disabled).
- `POST /auth/bot/generate-token` hardening.
- CORS no-Origin production policy.
- `constructor.js` prompt/confirm → modal (если остались ветки).
- Service layer extraction; `user_achievements` table drop.
- Массовый `console.*` → `logger`.
- Achievements model unification.
