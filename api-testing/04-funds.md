# 04. Funds: list, filters, fund page, compare, watchlist

Any signed-in user can use these (onboarding does not need to be finished). Every request needs
`Authorization: Bearer {{token}}`. None of them changes money, so you can repeat them freely.

**All requests in this file were run on the sandbox on 2026-11-03.**

| Call | Purpose |
|---|---|
| `GET /funds` | Fund list, with filters |
| `GET /funds/collections` | The collection tiles (Mid Cap, Commodity, ...) |
| `GET /funds/:isin` | One fund's page |
| `GET /funds/compare?isins=A,B` | Compare 1 or 2 funds |
| `GET /funds/compare/popular` | Ready-made compare pairs |
| `POST /watchlist/:isin` | Add / remove a fund from the watchlist (toggle) |
| `GET /watchlist` | The watchlist |

---

## 1. Fund list

```
GET {{baseUrl}}/funds
Authorization: Bearer {{token}}
```

```bash
curl https://risips-demo-api.azurewebsites.net/funds -H "Authorization: Bearer <TOKEN>"
```

**Expected (200):** `{ "title": null, "funds": [ ...35 funds... ] }`. One fund looks like this (trimmed):

```json
{
  "isin": "INF109KC1Y64",
  "name": "ICICI Prudential Silver ETF FOF - Regular Plan - Growth",
  "short_name": "ICICI Pru Silver ETF FOF",
  "amc": "ICICI Prudential Mutual Fund", "amc_short": "ICICI Pru", "amc_id": 15,
  "category": "Commodity",
  "sub_category": "FoF Domestic",
  "risk": "Very High Risk",
  "tags": ["commodity"],
  "nav": 26.7905,
  "nav_date": "2026-11-02",
  "returns": { "1D": -1.3, "1M": -18.01, "6M": -25.34, "1Y": 19.36, "3Y": 33.76, "5Y": null },
  "lumpsum": { "allowed": true, "min": 100, "additional_min": 100, "multiples": 1, "max": 999999999 },
  "sip_min": 100,
  "stp": { "in": true, "out": true, "frequency": "monthly", "min": 1000, "multiples": 1, "installments_min": 6, "dates": [1, 2, "...", 28] },
  "swp": { "frequency": "monthly", "min": 1, "multiples": 1, "installments_min": 2, "dates": [1, 2, "...", 28] },
  "switch_in_min": 100,
  "top_performer": true,
  "watchlisted": false
}
```

**What to check on every fund:**

| Field | Check |
|---|---|
| `nav_date` | **Recent**: the last working day (on 2026-11-03 every fund showed `2026-11-02`). An old date (e.g. September) is a bug (see RSP-FND-002) |
| `nav` | A positive number |
| `category` | One of `Equity`, `Debt`, `Hybrid`, `Index`, `Commodity`, `Solution`. Gold and silver funds are `Commodity`, not `Index` |
| `risk` | One of `Low Risk`, `Low to Moderate Risk`, `Moderate Risk`, `Moderately High Risk`, `High Risk`, `Very High Risk`. Debt / liquid funds should **not** be "Very High Risk" |
| `returns` | Numbers in %, or `null` when the fund is too young (e.g. no 5Y) |
| `lumpsum.allowed` | `false` for SIP-only funds (ICICI Pru Mid Cap `INF109K01AN2`) |
| `watchlisted` | `true` only for funds you added to your watchlist |

On 2026-11-03 the sandbox list had 35 funds: Equity 15, Debt 6, Index 5, Commodity 4, Hybrid 3, Solution 2.

---

## 2. Filters

Add these to `/funds` as query parameters. Each parameter may appear only once.

| Example | What it does | Result on 2026-11-03 |
|---|---|---|
| `/funds?category=Commodity` | Only that category | 4 funds (ICICI Pru Silver ETF FOF, ABSL Silver ETF FOF, ABSL Gold Fund, ICICI Pru Gold ETF FOF) |
| `/funds?category=equity,hybrid` | Several categories, any case | 18 funds |
| `/funds?risk=Low%20to%20Moderate%20Risk` | By risk label (spaces as `%20`) | 3 funds (ICICI Pru Money Market, Liquid, Gilt) |
| `/funds?q=icici%20liquid` | Search: all words must match | 1 fund (ICICI Pru Liquid Fund) |
| `/funds?top=1` | Top performers only | 11 funds |
| `/funds?sort=1Y&category=Debt` | Sort by a return (`1M`, `6M`, `1Y`, `3Y` default, `5Y`), highest first | ABSL Liquid 6.36, ICICI Pru Liquid 6.31, ... ICICI Pru Gilt 2.06 |
| `/funds?collection=mid` | One collection | `"title": "Mid Cap"`, 2 funds |
| `/funds?collection=tax&picks=1` | "Our picks" in a collection | `"title": "Tax Saver (ELSS)"`, 1 fund |
| `/funds?amc_id=15` | One fund house (15 = ICICI Pru, 3 = ABSL) | 21 ICICI Pru funds (`amc_id=3`: 14 ABSL funds) |
| `/funds?collection=foryou` | The "For YOU!" collection | 16 funds |
| `/funds?switch_in=1` / `/funds?stp_in=1` | Funds that accept switches / STPs in | 35 / 34 funds (Nifty50 Value 20 has no STP) |

**Negative tests:**

| Send | Expected |
|---|---|
| `/funds?collection=xyz` | `404` `{ "error": "Unknown collection" }` |
| `/funds?category=Debt&category=Equity` (parameter twice) | `400` `{ "error": "Invalid filter.", "reason": "validation", "fields": { "category": "Invalid filter." } }` |
| `/funds?category=Nothing` | `200` with an empty `funds` list |

Note: the **Commodity collection** (`?collection=commodity`) has **5** funds: the 4 Commodity-category funds plus ICICI
Pru Multi Asset Allocation Fund (category Hybrid, tagged `commodity`). The **category** filter has 4. This is how it
works today; ask the product owner if the Multi Asset fund should be in that collection.

## 3. Collections

```
GET {{baseUrl}}/funds/collections
```

**Expected (200):**

```json
{ "collections": [
  { "key": "high", "title": "High Return" }, { "key": "commodity", "title": "Commodity" },
  { "key": "mid", "title": "Mid Cap" }, { "key": "large", "title": "Large Cap" },
  { "key": "small", "title": "Small Cap" }, { "key": "tax", "title": "Tax Saver (ELSS)" },
  { "key": "debt", "title": "Debt & Liquid" }, { "key": "index", "title": "Index Funds" }
] }
```

`foryou` also works in `/funds?collection=foryou` but is not in this list.

---

## 4. Fund page

```
GET {{baseUrl}}/funds/INF109KC1Q80?period=3Y
Authorization: Bearer {{token}}
```

`period` is `1M`, `6M`, `1Y` (default), `3Y` or `5Y`: it picks the chart range.

**Expected (200):** `{ "fund": {...}, "period": "3Y", "period_return": 14.03, "chart": [[1790294400000, 20.15], ...] }`

The `fund` has everything from the list, plus (real values for ICICI Pru Flexi Cap):

```json
{
  "nav": 19.69, "nav_date": "2026-11-02", "risk": "Very High Risk", "category": "Equity",
  "lumpsum": { "allowed": true, "min": 500, "additional_min": 500, "multiples": 1, "max": 999999999 },
  "sip": [ { "frequency": "monthly", "min": 100, "multiples": 1, "installments_min": 6, "dates": [1, "...", 28] } ],
  "lock_in": "N.A",
  "exit_load": "1% (<1Yr)",
  "tax": { "stcg": "20%", "ltcg": "12.5%*" },
  "expense_ratio": { "regular": 1.73, "direct": 0.88, "date": "2026-10-29", "source": "AMFI TER disclosure" },
  "managers": ["Rajat Chandak", "Sharmila D'Silva"],
  "objective": "To generate long-term capital appreciation ...",
  "portfolio": { "as_of": "2026-09-30", "aum_cr": 25141, "count": 77, "mix": { "equity": 97.16, "debt": 0.52, "funds": 0 },
                 "holdings": [ { "name": "TVS Motor Company Ltd.", "industry": "Automobiles", "kind": "equity", "pct": 9.51 }, "..." ] },
  "peers": [ { "name": "ICICI Pru Flexi Cap fund", "r1": -0.91, "r3": 14.03, "r5": 12.53 } ],
  "redemption_allowed": true, "switch_in_allowed": true, "switch_out_allowed": true,
  "watchlisted": false
}
```

**Check:**
- `nav_date` is recent and matches the list.
- The last point of `chart` is the latest NAV (here `19.69`).
- `period` in the answer is the one you asked for.
- `lumpsum.min` is the minimum for a first purchase; once the user has a folio at that fund house it shows the
  additional-purchase minimum.

**Negative tests:**

| Send | Expected |
|---|---|
| `/funds/INF109KZZZZZ` (not in the list) | `404` `{ "error": "Fund not found" }` |
| `/funds/INF109KC1Q80?period=2Y` (unknown period) | `200` with `"period": "1Y"`: an unknown period silently falls back to 1Y (no error) |

---

## 5. Compare

```
GET {{baseUrl}}/funds/compare?isins=INF109KC1Q80,INF209K01AJ8
Authorization: Bearer {{token}}
```

**Expected (200):** `{ "funds": [ {...}, {...} ], "unavailable": [] }`

Each compared fund has the list fields plus `exit_load`, `lock_in`, `min_sip`, `history`, `fund_house`, `mandate`,
`expense_ratio`, `managers`, `objective`, `portfolio`, `pros`, `cons`. `unavailable` lists data that is missing for
these funds (`expense_ratio`, `holdings`, `managers`); empty means everything is there.

**Negative tests:**

| Send | Expected |
|---|---|
| 3 ISINs: `?isins=INF109KC1Q80,INF209K01AJ8,INF109K01VQ1` | `400` `{ "error": "You can compare up to 2 funds.", "reason": "validation", "fields": { "isins": "You can compare up to 2 funds." } }` |
| `?isins=INF109KZZZZZ` (well-formed, unknown) | `404` "Fund not found" |
| `?isins=bad` | `400` `{ "error": "Fund not found", "reason": "validation", "fields": { "isins.0": "Fund not found" } }` |

### Popular pairs

```
GET {{baseUrl}}/funds/compare/popular
```

**Expected (200):** `{ "pairs": [[fund, fund], ...] }`. On 2026-11-03: 4 pairs (ICICI Pru vs ABSL Large Cap, Flexi Cap,
Liquid, and Mid Cap).

---

## 6. Watchlist

`POST /watchlist/:isin` is a **toggle**: the first call adds, the second removes.

```
POST {{baseUrl}}/watchlist/INF109KC1Q80
Authorization: Bearer {{token}}
```

**Expected (200):** `{ "watchlisted": true }`, then on the next call `{ "watchlisted": false }`.

```
GET {{baseUrl}}/watchlist
```

**Expected (200):** `{ "funds": [ { "isin": "INF109KC1Q80", "watchlisted": true, "...": "..." } ] }`

**Check:** the fund also shows `"watchlisted": true` in `GET /funds` and `GET /funds/INF109KC1Q80`.

**Negative test:** `POST /watchlist/INF109KZZZZZ` -> `404` "Fund not found".
