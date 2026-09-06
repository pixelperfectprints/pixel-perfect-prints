# Pixel Perfect Prints — Agent API Brief

> For Hermes agents (Gerry, Rick, cron jobs, etc.): how to read and write the
> business inventory. No scraping, no logins — the Firebase Realtime Database
> **is** the API.

## Endpoint

Base URL (no auth needed for reads once rules allow — see Security note):

```
https://<DATABASE_URL>/ops/<path>.json
```

`<DATABASE_URL>` comes from `C:\Users\jammi\print-ops\site\firebase-config.js`
(`databaseURL` field). Strip the trailing `/` if present.

## Data layout (Realtime Database, under `ops/`)

| Path | Contents | Units |
|---|---|---|
| `stock/<ProductName>` | `{qty: <int>}` — finished goods on hand | units |
| `stock/<ProductName>/price` | unit price (read-only reference, seeded) | £ |
| `filament/<spoolId>` | `{material, color, brand, stock_jammin, stock_james}` | kg |
| `products/<id>` | public catalogue: `{name, price, desc, img, hidden}` | — |
| `enquiries/<id>` | customer quote requests: `{name, email, msg, created, status}` | — |
| `jobs/<id>`, `assign/<id>`, `orders/<id>`, `notes/<id>` | operational logs | — |

## Key facts agents must know

- **Stock tab in `ops.html` is the MASTER** for finished goods. Never trust
  `E:\Hermes Brain\Hermes Brain\1-wiki\operations\stock-list.md` for current
  numbers — it is a deprecated 2026-08-30 snapshot (1,645 units / £13,671.50).
- Stock quantities are **whole units**, not kg. Filament is **kg** with two
  separate counters: `stock_jammin` (at the workshop) and `stock_james`
  (with James). Total spool stock = sum of both.
- Prices in `stock/` are seeded references; revenue maths = qty × price.
- Customer enquiries arrive in `enquiries/` with `status:"new"`.
- The dashboard UI reads the same paths — anything an agent writes appears
  live for Jammin & James, and vice versa.

## Read examples (curl)

```bash
# whole finished-goods ledger
curl -s "https://<DATABASE_URL>/ops/stock.json"

# one product
curl -s "https://<DATABASE_URL>/ops/stock/Pikachu.json"

# filament library (all spools, both shelves)
curl -s "https://<DATABASE_URL>/ops/filament.json"

# open customer enquiries
curl -s "https://<DATABASE_URL>/ops/enquiries.json"
```

## Write examples (curl)

```bash
# Gerry sold 3 Pikachus (decrement)
curl -s -X PATCH "https://<DATABASE_URL>/ops/stock/Pikachu.json" \
  -d '{"qty": 24}'

# record a new enquiry status
curl -s -X PATCH "https://<DATABASE_URL>/ops/enquiries/<id>.json" \
  -d '{"status": "quoted"}'
```

Rules for agents writing stock: **read-modify-write** (GET current qty, compute
new qty, PATCH it). Never blind-set a qty you didn't just read — another actor
may have changed it between your read and write. Keep qty ≥ 0.

## Security note

- Current rules may be test-mode (30-day expiry). Before expiry, Jammin must
  publish locked rules: public read ONLY on `ops/products`; everything else
  requires Firebase Auth. If agent writes are wanted post-lockdown, the clean
  path is a per-agent Firebase Auth user + auth token in the rules, or a small
  Cloud Function proxy with its own token.
- Until rules are locked, the database URL is the only secret — do not commit
  it to public repos.

## Product line reference (seeded 2026-08-30)

Categories & prices: Pokémon figures £16 · Pokéballs £12 · Large dragons £20 /
XL £25 / flex rex £20 · Keyrings £3 · Minis/fidgets £2.50 (3 for £5) ·
Pokéscenes (large) £12 · Hueforges £20 · Moon lamps £20.
Full baseline list: the deprecated vault note. Live truth: `ops/stock`.
