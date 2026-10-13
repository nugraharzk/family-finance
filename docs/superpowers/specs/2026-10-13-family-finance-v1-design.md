# Family Finance — v1 Design Spec

**Date:** 2026-10-13
**Status:** Approved for planning
**Project dir:** `~/Projects/family-finance`

---

## 1. Intent

A self-hosted household finance app for **one family**, built because the existing
options (Firefly III, Actual Budget, Securo, Econumo, Expensave) have the features
but a UI nobody enjoys opening.

**Who it's for:** the household — 2 adults plus optionally kids, on phones and a
laptop, over the home LAN.

**Success criteria (v1):**

1. Recording an expense on a phone takes **under 10 seconds** and is pleasant enough
   that it actually happens.
2. A month's bank CSV imports **without double-counting** and without hand-editing
   the file first.
3. Any household member can answer "how much did we spend on X this month" and
   "where did Dad's money go" from the dashboard alone.
4. Backup is **copying one file**.
5. The UI is one a partner or a kid can read **across the room**.

**Explicitly not a product.** No multi-tenancy, no billing, no onboarding, no
marketing, no internet-facing hardening.

---

## 2. Scope

### In v1

- Members, sessions, login, roles (adult / kid), first-run setup wizard.
- Accounts (bank / cash / wallet / credit) with opening balances.
- Transactions: manual add/edit/delete, transfers between accounts.
- Categories: flat or nested, emoji + colour, archivable.
- Attribution tags: `paid_by`, `for_whom`.
- Generic CSV import with saved per-bank mapping profiles, dedupe, batch rollback.
- Auto-categorisation rules, re-appliable to existing transactions.
- Reports: net worth, monthly cash flow, spending by category.
- Single currency (IDR), single household.

### Out of v1 (later slices, each with its own spec)

| Slice | Contents |
|---|---|
| 2 | Budgets / envelopes, recurring bills, goals |
| 3 | Live bank sync (SimpleFIN / GoCardless), PDF statement parsing |
| 4 | Investments & asset valuation |
| 5 | Kid allowance pockets, gamification |
| 6 | TLS via Caddy, multi-household, event-sourced audit log |
| — | Multi-currency, FX rates — **not planned**, revisit only if it becomes real |

**Never in v1:** budgets, goals, investments, FX, event log, multi-tenancy,
email (SMTP), public signup, PDF parsing.

---

## 3. Architecture

**Shape: modular monolith.** One container, one SQLite file. Chosen because it's
structurally different from all five reference apps (which are split-service
PHP/Node/Go) while being the *right* answer at family scale — and because
deployment and backup collapse to one file.

```
family-finance/
├── backend/
│   ├── app/
│   │   ├── main.py              # app factory: mounts /api, serves SPA, exception handlers
│   │   ├── core/
│   │   │   ├── config.py        # settings (pydantic-settings)
│   │   │   ├── db.py            # engine, session, PRAGMAs, WAL
│   │   │   ├── security.py      # argon2, session tokens, CSRF/origin checks
│   │   │   ├── money.py         # parse/format rupiah; no floats, ever
│   │   │   ├── errors.py        # domain exceptions + handlers
│   │   │   └── deps.py          # auth dependencies, role guards
│   │   └── modules/
│   │       ├── auth/            # members, sessions, login, rate limit
│   │       ├── accounts/        # router / service / models / schemas
│   │       ├── transactions/
│   │       ├── categories/
│   │       ├── imports/
│   │       └── reports/
│   │   └── migrations/          # Alembic
│   └── tests/
├── frontend/                    # Nuxt 3 (Vue 3) — SPA, static build
│   ├── assets/css/main.css      # @theme tokens + brutalist utilities
│   └── ...
├── Dockerfile                   # multi-stage: node build → python runtime
├── docker-compose.yml           # 1 service + 1 volume (app.db)
└── docs/superpowers/specs/
```

**Module contract.** Every module is exactly:

| File | Responsibility |
|---|---|
| `router.py` | HTTP only — parse, call service, serialise. No logic. |
| `service.py` | All business logic. The only cross-module entry point. |
| `models.py` | SQLAlchemy tables. Never imported by another module. |
| `schemas.py` | Pydantic request/response shapes. |

Cross-module calls go through `service` functions only. This is what keeps a
monolith from becoming a ball of mud, and it's the one architectural rule that
must be enforced in review.

**Deploy.** `docker compose up -d` → one container on port 8000, one volume
holding `app.db`. Multi-stage Dockerfile: node stage builds Nuxt to static
files, python stage installs deps and copies them into `app/static/`; FastAPI
serves the SPA at `/` and the API at `/api/*`.

**Money.** IDR has no practical subunit. Amounts are signed `INTEGER` rupiah.
**Floats are forbidden** — enforced by a test that greps the codebase and by
`hypothesis` never seeing a float return.

---

## 4. Data model

### DDL

```sql
CREATE TABLE members (
  id            INTEGER PRIMARY KEY,
  name          TEXT    NOT NULL,
  username      TEXT    NOT NULL UNIQUE,
  password_hash TEXT    NOT NULL,
  role          TEXT    NOT NULL CHECK (role IN ('adult','kid')),
  avatar_color  TEXT    NOT NULL DEFAULT '#118AB2',
  is_active     INTEGER NOT NULL DEFAULT 1,
  created_at    TEXT    NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE sessions (
  id           TEXT    PRIMARY KEY,              -- 32 random bytes, hex
  member_id    INTEGER NOT NULL REFERENCES members(id) ON DELETE CASCADE,
  created_at   TEXT    NOT NULL DEFAULT (datetime('now')),
  expires_at   TEXT    NOT NULL,
  last_seen_at TEXT    NOT NULL DEFAULT (datetime('now')),
  user_agent   TEXT
);
CREATE INDEX ix_sessions_member ON sessions(member_id);

CREATE TABLE accounts (
  id              INTEGER PRIMARY KEY,
  name            TEXT    NOT NULL,
  type            TEXT    NOT NULL CHECK (type IN ('bank','cash','wallet','credit')),
  opening_balance INTEGER NOT NULL DEFAULT 0,
  sort_order      INTEGER NOT NULL DEFAULT 0,
  archived        INTEGER NOT NULL DEFAULT 0,
  created_at      TEXT    NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE categories (
  id        INTEGER PRIMARY KEY,
  name      TEXT    NOT NULL,
  kind      TEXT    NOT NULL CHECK (kind IN ('income','expense')),
  parent_id INTEGER REFERENCES categories(id),
  emoji     TEXT    NOT NULL DEFAULT '📦',
  color     TEXT    NOT NULL DEFAULT '#C9B6FF',
  archived  INTEGER NOT NULL DEFAULT 0
);
CREATE UNIQUE INDEX ux_categories_ident
  ON categories(kind, name, IFNULL(parent_id, -1));

CREATE TABLE import_profiles (
  id          INTEGER PRIMARY KEY,
  name        TEXT    NOT NULL,
  bank_name   TEXT,
  delimiter   TEXT    NOT NULL DEFAULT ',',
  encoding    TEXT    NOT NULL DEFAULT 'utf-8',
  has_header  INTEGER NOT NULL DEFAULT 1,
  skip_rows   INTEGER NOT NULL DEFAULT 0,
  date_format TEXT,                              -- strptime pattern; NULL = ask per import
  amount_mode TEXT    NOT NULL
              CHECK (amount_mode IN ('signed','debit_credit','amount_type','flip_outflow')),
  column_map  TEXT    NOT NULL,                  -- JSON: field -> column index
  created_at  TEXT    NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE imports (
  id              INTEGER PRIMARY KEY,
  profile_id      INTEGER NOT NULL REFERENCES import_profiles(id),
  account_id      INTEGER NOT NULL REFERENCES accounts(id),
  filename        TEXT    NOT NULL,
  uploaded_by     INTEGER NOT NULL REFERENCES members(id),
  uploaded_at     TEXT    NOT NULL DEFAULT (datetime('now')),
  status          TEXT    NOT NULL CHECK (status IN ('committed','rolled_back')),
  row_count       INTEGER NOT NULL,
  imported_count  INTEGER NOT NULL,
  duplicate_count INTEGER NOT NULL,
  possible_duplicate_count INTEGER NOT NULL DEFAULT 0,
  error_count     INTEGER NOT NULL
);
CREATE INDEX ix_imports_account ON imports(account_id, uploaded_at);

CREATE TABLE transactions (
  id                INTEGER PRIMARY KEY,
  account_id        INTEGER NOT NULL REFERENCES accounts(id),
  date              TEXT    NOT NULL,            -- YYYY-MM-DD
  amount            INTEGER NOT NULL,            -- signed rupiah: outflow < 0
  description       TEXT    NOT NULL DEFAULT '',
  category_id       INTEGER REFERENCES categories(id),   -- nullable: import first
  paid_by           INTEGER REFERENCES members(id),      -- attribution tags
  for_whom          INTEGER REFERENCES members(id),
  note              TEXT,
  transfer_group_id TEXT,                        -- two rows share one UUID
  import_id         INTEGER REFERENCES imports(id) ON DELETE CASCADE,
  import_hash       TEXT,
  created_at        TEXT    NOT NULL DEFAULT (datetime('now')),
  updated_at        TEXT    NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX ix_tx_account_date ON transactions(account_id, date);
CREATE INDEX ix_tx_date         ON transactions(date);
CREATE INDEX ix_tx_category     ON transactions(category_id);
CREATE INDEX ix_tx_transfer     ON transactions(transfer_group_id);
CREATE UNIQUE INDEX ux_tx_import_hash
  ON transactions(import_hash) WHERE import_hash IS NOT NULL;   -- partial: NULLs allowed

CREATE TABLE rules (
  id              INTEGER PRIMARY KEY,
  priority        INTEGER NOT NULL DEFAULT 100,
  match_type      TEXT    NOT NULL CHECK (match_type IN ('contains','starts_with','equals','regex')),
  match_value     TEXT    NOT NULL,
  set_category_id INTEGER REFERENCES categories(id),
  set_paid_by     INTEGER REFERENCES members(id),
  is_active       INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX ix_rules_order ON rules(is_active, priority);

CREATE TABLE settings (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
```

### Decisions

| # | Decision | Rationale |
|---|---|---|
| 1 | **Single-entry + paired transfer legs**, not double-entry | Balance = `opening_balance + SUM(amount)`. Firefly's double-entry is ceremony at family scale. Both legs written in one DB transaction. |
| 2 | Signed `INTEGER` rupiah | No floats, no cents, no rounding bugs. |
| 3 | Attribution as **nullable tag FKs**, not separate ledgers | One wallet, but "what did Dad spend on the kids" is still answerable. Nullable → you may not care. |
| 4 | `category_id` **nullable** | An unmapped CSV row must still import. Never block on a missing category. |
| 5 | Dedupe via **partial unique `import_hash`** | Re-importing the same statement is a no-op. Partial index keeps manual entries (`NULL`) unlimited. |
| 6 | `archived` flags, never hard deletes | Transactions reference these rows; history must not break. |
| 7 | Balances **computed**, not stored | Family scale. A stored running balance is an optimisation with no buyer. |
| 8 | Money ban on floats | Asserted in tests. |

### `import_hash` definition

```
import_hash = sha256( f"{account_id}|{date:%Y-%m-%d}|{amount}|{normalise(description)}" )
normalise(description) = collapse internal whitespace → trim → casefold
```

The running-balance column, if present in the CSV, is **excluded** from the hash.

**Known limitation:** some banks embed a changing reference number in the
description, which makes the hash differ for a genuine duplicate. So the import
review runs a **second pass**: any row matching an existing transaction on
`(account_id, date, amount)` but with a different hash is flagged
`possible_duplicate` for human review — flagged, never silently blocked.

The flag exists **only in the review step and is not persisted on the
transaction**. `possible_duplicate` rows arrive in the review list **unchecked by
default**; if the user accepts one, it is imported as an ordinary transaction,
indistinguishable from a manual entry. Counts are on separate axes:
`imported_count` = rows actually inserted, `possible_duplicate_count` = rows that
were *detected* as possible duplicates regardless of the decision, so a single
accepted row appears in both — intentionally.

---

## 5. Auth & family model

- **Session cookies**, not JWT. Same-origin SPA served by FastAPI → httpOnly
  cookie is simpler *and* stronger; JWTs would cost refresh machinery and lose
  revocation. A `sessions` row is revocable instantly.
- **Argon2id** password hashing.
- **Roles:** `adult` = full read/write, manage members, import, edit rules.
  `kid` = read-only plus a "my money" view filtered to `for_whom`; cannot edit,
  import, or manage accounts.
- **Login:** adults need a ≥12-char password. `kid` accounts may use a 6-digit
  PIN — safe *only* because of the rate limit below.
- **First-run setup:** if `members` is empty, only `/setup` is reachable; the
  first adult is created there. **No seeded default user** — that's the hole
  that gets self-hosted apps owned.
- **No registration endpoint, ever.** Adults create members from Settings.
  No password-reset-by-email (no SMTP); an adult resets a member's password.
- **Rate limiting:** per-IP + per-username failed-attempt counter with
  exponential backoff, in-process (no Redis at this scale).
- **CSRF:** `SameSite=Lax` cookie, require `Content-Type: application/json`, and
  verify the `Origin` header on state-changing requests.
- **Sessions:** 30-day sliding expiry, one row per device, "log out everywhere".

### Security posture — explicit trade

LAN-only, **no TLS in v1**. Consequence, stated rather than buried:

> Without TLS, passwords and session cookies cross the home Wi-Fi in cleartext.
> Anyone on the network (guest Wi-Fi, a compromised smart device, a neighbour on
> a weak WPA key) can sniff them. If that matters, terminate TLS even on the LAN
> — a self-signed cert is enough — or put it behind Caddy with a local cert.

**Internet-facing hardening is deliberately out of scope.** Port-forwarding this
to the open internet later would require a separate security pass — a real
project, not a config tweak. Recorded here so nobody assumes otherwise.

---

## 6. Import pipeline

Nothing is written to the DB until the user confirms. Dry-run, then commit.

```
upload → sniff → preview → map columns → parse → dedupe → apply rules
       → review (new/possible_dup/duplicate/error) → commit (one txn)
       → rollback-able batch
```

| Stage | Detail |
|---|---|
| **Sniff** | Detect delimiter (`, ; \t \|`), encoding (`utf-8-sig`, `utf-8`, `cp1252`, `latin-1`), header presence. Stdlib `csv` + `charset-normalizer`. **No pandas.** |
| **Map** | Choose a saved profile or create one. Fields: `date*`, `description*`, amount (4 modes), optional `paid_by`, `category`, `note`. |
| **Amount modes** | `signed` (`-45000`) · `debit_credit` (two columns) · `amount_type` (amount + `DB`/`CR`) · `flip_outflow` (positive means outflow). |
| **Amount normalise** | Strip `Rp`, thousands separators (`.`, `,`, space); handle `(1.234,56)` parens-as-negative, trailing minus, `DR`/`CR` suffix → signed `INTEGER` rupiah. |
| **Dates** | Parsed per the profile's `date_format`. **If absent and a value like `03/04/2025` is valid as both dd/mm and mm/dd, the wizard blocks and asks.** Never guess — guessing silently files a year of transactions in the wrong month. |
| **Dedupe** | `import_hash` vs existing → `duplicate`. Second pass `(account, date, amount)` → `possible_duplicate`. |
| **Rules** | Rule engine in `priority` order sets `category_id` / `paid_by`. |
| **Review** | Per-row status (`new` / `possible_duplicate` / `duplicate` / `error`) with the matched rule shown; counts at top; rows individually toggleable. `possible_duplicate` rows start **unchecked**. |
| **Commit** | One DB transaction: create the `imports` row, insert all transactions with `import_id`. Report: *imported 142, skipped 8 duplicates, 3 possible duplicates, 2 errors*. |
| **Rollback** | Delete an import batch → one query removes exactly its transactions via `import_id ON DELETE CASCADE`. |

**Rules are re-appliable to existing transactions** ("apply rules to all") — needed
the first time a rule is added after importing.

**Input validation:** 20 MB file cap, 50 000 row cap, reject binary/PDF with a
clear message.

**Row errors are data, not failures.** A malformed row returns a row-level error;
one bad line never 500s a 400-row import.

---

## 7. UI system — "Family Play"

Neo-brutalist but friendly. Flat colour, hard black border, hard offset shadow.

**Tokens** — declared **once** in `frontend/assets/css/main.css` via Tailwind v4's CSS-first `@theme` block. Screens never repeat these hex codes.

| Token | Value | Use | Tailwind class |
|---|---|---|---|
| `--color-paper` | `#FDFCF7` | page background | `bg-paper` |
| `--color-ink` | `#111111` | all text, all borders | `text-ink` `border-ink` |
| `--color-in` | `#06D6A0` | money in | `text-in` `bg-in` |
| `--color-out` | `#EF476F` | money out | `text-out` `bg-out` |
| `--color-hi` | `#FFD166` | highlight / CTA | `bg-hi` |
| `--color-info` | `#118AB2` | informational | `bg-info` |
| `--color-cat` | `#C9B6FF` | category accents | `bg-cat` |
| `--color-muted` | `#8D99AE` | secondary text | `text-muted` |

No gradients. No blur. No glassmorphism. **No component kit** — no shadcn, no
PrimeVue; those would fight the aesthetic and drag in their own design language.

**Styling: Tailwind CSS v4** via `@nuxtjs/tailwindcss`. Tailwind handles layout,
spacing, typography, and — the reason it won — responsive breakpoints
(`md:` / `lg:`) for the mobile→desktop sidebar switch.

The usual brutalist-Tailwind mess is repeated arbitrary values
(`border-[2.5px] shadow-[4px_4px_0_#111]` on every element). That's neutralised by
declaring the primitives once:

```css
@theme {
  --color-paper: #FDFCF7;  --color-ink:  #111111;
  --color-in:    #06D6A0;  --color-out:  #EF476F;
  --color-hi:    #FFD166;  --color-info: #118AB2;
  --color-cat:   #C9B6FF;  --color-muted:#8D99AE;
  --radius-brutal: 12px;
}

@utility border-brutal { border: 2.5px solid var(--color-ink); }
@utility shadow-brutal { box-shadow: 4px 4px 0 var(--color-ink); }
@utility press        { box-shadow: none; translate: 4px 4px; }
```

Screens then read as ordinary utilities — `border-brutal shadow-brutal bg-hi
text-ink rounded-brutal active:press` — with **no arbitrary values and no
repeated hex**. Changing the shadow means editing one `@utility`, not grepping
40 files.

**Tradeoff accepted:** one dependency plus a config file, in exchange for free
responsive utilities. The risk to watch is Tailwind conventions leaking into the
*design* — tokens remain the source of truth, never inline hex.

**Signature interaction.** Press = the offset shadow collapses to `0` and the
element translates `4px` into the gap. Physical, cheap, impossible to get from a
stock UI kit. It's what makes this read as designed rather than themed.

**Five UX rules** — the concrete answer to "the other five apps aren't eye-catching":

1. **Money is the hero** — huge tabular numerals (`font-variant-numeric: tabular-nums`), always. Columns align; nothing shifts while loading.
2. **One tap to record** — the `＋` is always on screen, never behind a menu.
3. **Read across the room** — state shown by colour + bar length, not a small number you must lean in to read.
4. **Uncategorised is visible, not silent** — dashed card + "belum dikategorikan", so it gets fixed.
5. **Emoji, not monochrome icon sets** — instantly recognisable, zero learning curve.

**Screens (v1):** Login (avatar tiles) · Dashboard · Accounts + account ledger ·
Transactions (filterable list, inline edit) · Add/Edit transaction (modal, built-in
calculator, quick category picker) · Import wizard (4-step stepper) · Categories ·
Rules · Members/Settings · Reports.

**Layout:** mobile-first. Desktop = sidebar; mobile = chunky bottom nav.

### Accessibility requirement

**Ink on colour, always. Never white on colour.**

- White on `#EF476F` = **3.62:1 → fails WCAG AA** for body text; passes only as
  large text (≥19px bold). Use `#111` ink on pink instead → **5.2:1, passes**.
- `#111` on `#06D6A0` and `#FFD166` = 10:1+, fine.

Enforced by a Playwright accessibility assertion in slice 2 and by manual review
in v1.

---

## 8. Testing & error handling

**Testing.** pytest against real SQLite (temp file). **Never a mocked DB** —
SQLite *is* production, and a test that passes on a mock and fails on the real
engine is worse than no test.

| Layer | Content |
|---|---|
| Unit (bulk) | Amount parsing (4 modes + all edge cases), date-ambiguity detection, `import_hash` stability, rule priority order, `balance = opening + SUM(amount)` |
| Property (`hypothesis`) | Amount + date parsing — thousands of malformed strings; parser never crashes, never returns a float |
| Golden-file | One fixture CSV per amount mode → assert exact resulting transactions |
| API integration | setup → login → import → report, via httpx ASGI transport |
| E2E | One cold-boot test walking the whole v1 flow |

**v1 has no frontend tests.** Slice 2 adds 3 Playwright smoke tests (login, add
transaction, import). Tested logic over untested chrome.

**Error handling.**

- One envelope: `{"error": {"code", "message", "details"}}`.
- No stack traces or SQL to the client; those go to the log with a request id from middleware.
- Domain exceptions (`NotFound`, `ValidationFailed`, `Conflict`, `PermissionDenied`) → 404 / 422 / 409 / 403, mapped centrally.
- Import row errors are per-row data, never a 500.
- Import commit is one DB transaction; any failure rolls the whole batch back.

**SQLite settings (mandatory):**

- `PRAGMA foreign_keys=ON` **per connection** — OFF by default; FKs are silently ignored otherwise.
- `PRAGMA journal_mode=WAL`, `PRAGMA busy_timeout=5000`.
- Short write transactions.

**Ops:** Alembic auto-runs on boot; `/api/health` for the compose healthcheck;
nightly `app.db` copy via host cron; structured logs carrying ids, **not**
descriptions or amounts.

---

## 9. Known risks

1. **SQLite single-writer.** Fine for 3 people. Would need Postgres only if you ever added a second household.
2. **CSV drift.** Banks change exports. The generic mapper absorbs most of it; a changed encoding mid-file fails loudly rather than silently.
3. **No TLS on LAN.** Accepted trade (see §5). Revisit if ever exposed.
4. **Kid PINs** are only safe because of the rate limit. Removing the limit makes PINs a liability.
5. **Description-embedded reference numbers** break naive dedupe; mitigated by the `possible_duplicate` second pass (§4).

---

## 10. Assumptions to confirm

1. **UI copy is in Bahasa Indonesia** (the mockups used it). Say if you want English or bilingual.
2. Household is **2 adults**, kids optional.
3. Reference apps were reviewed for **feature ideas only** — no code is copied from them, so their licences (AGPL/GPL/MIT) don't apply to this project.
4. Project lives at `~/Projects/family-finance`, separate from the TELKOM work monorepo.

---

## 11. Next step

Write the implementation plan (`writing-plans`): ordered tasks with file
references and executable QA scenarios, derived from this spec.

v1 is one deliverable but too large for a single flat task list, so the plan
should be phased — each phase ending in something runnable:

1. **Skeleton** — repo, Dockerfile, compose, FastAPI app factory, SQLite PRAGMAs, Alembic, `/api/health`, Nuxt scaffold with Tailwind + `@theme` tokens and the `border-brutal` / `shadow-brutal` / `press` utilities.
2. **Members & auth** — setup wizard, login, sessions, roles, rate limit, role guards.
3. **Accounts & categories** — CRUD, archive, emoji/colour, nesting.
4. **Transactions** — manual add/edit/delete, transfers, attribution, filters.
5. **Import** — sniff, profiles, all four amount modes, date ambiguity gate, dedupe, rules, review, commit, rollback.
6. **Reports** — net worth, cash flow, by category.
7. **UI pass** — Family Play components, mobile-first screens, accessibility check.

Phases 2–4 are the vertical slice that makes it usable; 5 is the highest-risk
and should not start until the ledger is trustworthy.
