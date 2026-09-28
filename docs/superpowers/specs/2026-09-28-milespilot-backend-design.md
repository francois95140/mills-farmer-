# MilesPilot — Backend + Scraper Design (sub-project 1 of 2)

Date: 2026-09-28

## Context / Intent

MilesPilot is a Chrome extension that tells a shopper, on a merchant site, whether
buying through the Flying Blue Store portal (+ paying with a specific Amex card)
earns more Miles than a direct purchase, and shows the estimated Miles total.

This spec covers the **backend + data pipeline** sub-project: the NestJS API and
the scraper that keeps offer data fresh. The Chrome extension is a separate
sub-project (spec to follow), built against this API's contract.

### Scope for v1

- Reward programs: **Flying Blue Store** (merchant miles-per-€10 rate, scraped)
  and **Amex card miles** (flat rate per card, manually seeded) only.
- Other programs (Amex Membership Rewards portal offers, Revolut RevPoints,
  cashback) are **not built now**, but the schema must not require rework to
  add them later (see `loyalty_programs` table — new program rows, new scraper
  modules, no schema migration needed for the core reward model).
- `flying_blue_status` (Explorer/Silver/Gold/Platinum) is stored on the user
  profile but **must not** influence the calculated Miles — no verified rule
  ties portal status to portal Miles rate. It's profile metadata only.
- No user auth. Extension generates a local anonymous UUID; backend creates a
  `users` row lazily on first profile update.

### Success criteria

- `GET /offers?merchant=apple&country=FR` returns an offer with a Miles number
  the team can trust is either (a) freshly scraped and unexpired, or (b)
  explicitly flagged `UNVERIFIED` — never a fabricated/stale number presented
  as current.
- Offer data refreshes automatically every week without manual intervention.
- Adding a second loyalty program later requires new rows/scraper code, not a
  schema rewrite.

## Architecture

```
                    ┌─────────────────────────┐
                    │   NestJS backend (repo)  │
                    │                          │
  cron (weekly) ───▶│  ScraperModule           │
                    │   └─ Playwright scrape   │──▶ upserts
                    │      of flyingblueplus-  │
                    │      shopping.flyingblue │
                    │      .com catalog         │
                    │                          │
  Chrome ext  ─────▶│  OffersModule (REST API) │◀── reads
  (GET /offers)     │  UsersModule             │
                    │  CardsModule              │
                    └───────────┬──────────────┘
                                │ TypeORM
                                ▼
                        PostgreSQL (Docker)
```

Single NestJS repo, single Docker Compose (`postgres` + `app`). No separate
scraper service — runs as a scheduled job (`@nestjs/schedule`) inside the same
process, plus a protected manual-trigger endpoint for on-demand refresh.

## Data model (PostgreSQL / TypeORM)

```
merchants
  id, name, slug, logo_url, created_at

merchant_domains
  id, merchant_id → merchants, domain (e.g. "apple.com"), match_type (exact|suffix)

countries
  id, code (FR, US, ...), name

loyalty_programs
  id, code ("FLYING_BLUE"), name
  -- future rows: "AMEX_MR", "REVOLUT", "CASHBACK" — no schema change needed

offers                          -- Flying Blue Store offers, scraped
  id, merchant_id → merchants, loyalty_program_id → loyalty_programs,
  country_id → countries,
  base_miles_per_10eur numeric,
  tracking_url,
  source ("flyingblue_scraper"),
  scraped_at, valid_from, valid_until nullable,
  is_active boolean,
  raw_snapshot jsonb            -- raw scraped payload, for audit/debug

offer_tiers                     -- bonus tiers, e.g. "+2500 miles if spend > 500€"
  id, offer_id → offers,
  min_spend_eur nullable, max_spend_eur nullable,
  bonus_miles nullable, bonus_multiplier nullable,
  description

offer_conditions                -- exclusions/conditions shown to the user
  id, offer_id → offers, label, type (exclusion|condition)

cards
  id, name ("Amex KLM Gold"), issuer, program_id → loyalty_programs

card_reward_rules               -- flat rate, not merchant-specific
  id, card_id → cards,
  miles_per_10eur numeric,
  category nullable,            -- e.g. "travel" vs "general", for later use
  valid_from, valid_until nullable

users
  id, email nullable, country_id → countries,
  selected_card_id → cards nullable,
  flying_blue_status enum(EXPLORER|SILVER|GOLD|PLATINUM) nullable
```

`offers` is append-only: a new scrape inserts a new row and flips
`is_active = false` on the previous active offer for that
merchant+program+country. This gives offer history for free, and lets
`/offers` always answer from `is_active = true` without a separate history
table.

## Scraper module

- `ScraperModule` → `FlyingBlueScraperService`, using Playwright (site at
  `flyingblueplus-shopping.flyingblue.com` is server-rendered but the catalog
  list loads/paginates dynamically — confirmed via manual inspection — so a
  headless browser is more robust than plain HTTP + HTML parsing).
- Weekly cron (`@Cron`, Sunday 03:00 Europe/Paris) → `runFullScrape()`.
- Flow: launch headless browser → paginate/scroll the catalog → for each
  merchant card, extract name, slug, miles-per-€10 → visit the merchant detail
  page for the tracking URL and any tier/condition text → resolve/guess the
  merchant's domain (for `merchant_domains`) → upsert `merchants` /
  `merchant_domains`, insert new `offers` row + `offer_tiers` /
  `offer_conditions` if present.
- Manual trigger: `POST /admin/scrape/run` (no public auth in v1, but bind to
  localhost/internal network only — do not expose publicly).
- Per-merchant scrape errors are logged and skipped; they don't abort the run.
  Each run logs a summary (merchants updated/failed count).

## Reward calculation

Pure function, isolated from the HTTP layer (`RewardCalculatorService`):

```
calculate(offer, cardRule, priceEur):
  fbMiles = priceEur * offer.base_miles_per_10eur / 10
          + sum of matching offer_tiers bonuses for priceEur
  cardMiles = cardRule ? priceEur * cardRule.miles_per_10eur / 10 : 0
  total = fbMiles + cardMiles
```

Cumulating Flying Blue Store miles and card miles is treated as an
**unverified assumption** (no confirmed exclusivity rule found) — the API
response must carry this as an explicit condition/disclaimer, not assert it
silently as guaranteed.

If no active, unexpired offer exists for the merchant+country: respond with
`status: "UNVERIFIED"` and no Miles numbers. Never fabricate a value.

## REST API

- `GET /offers?merchant=:slug&country=:code&cardId=:id?` → active offer +
  tiers + conditions + (if `cardId` given) card rule + computed reward
  breakdown + freshness metadata (`source`, `scraped_at`, `valid_until`).
- `GET /merchants/resolve?domain=apple.com` → merchant slug (used by the
  extension's content script to identify the current site).
- `GET /cards` → list of supported cards, for the options page dropdown.
- `POST /users` / `PATCH /users/:id` → save card/country/status selection;
  `users` row created lazily on first write for a new anonymous id.
- `POST /admin/scrape/run` → manual scrape trigger (internal use).

## Testing

- Unit: `RewardCalculatorService` — flat rate, tiers, missing offer, expired
  offer. Pure functions, TDD-friendly.
- Integration: scraper run against saved HTML/DOM fixtures (1–2 real merchant
  pages captured as fixtures), not the live site on every test run.
- E2E: supertest against the NestJS app + a test Postgres (Docker) for the
  `/offers` endpoint end to end.

## Out of scope for this spec

- The Chrome extension (content script, popup, price/merchant detection) —
  separate sub-project spec.
- Admin UI for editing offers/merchants — v1 relies on the scraper + manual
  DB/seed edits for Amex card rates.
- Any loyalty program other than Flying Blue Store + Amex card miles.
- User accounts / authentication.
