# Smart Shopping List — Project Plan

A Telegram bot that lives in a private group (you + your wife), keeps a shared
shopping list up to date from ordinary messages and photos, shows the list as a
tappable checklist, and tells you which 2–3 supermarkets are best for this
basket based on current prices and distance from home.

> **Assumption:** you're in Israel. The price-comparison part relies on the
> Israeli Price Transparency Law (2015), which requires large chains to
> publish their full price and promotion files every day. If you're somewhere
> else, everything except the price data source stays the same (see
> [Open questions](#open-questions)).

---

## 1. User experience

### Adding items (any message in the group)
| You send | Bot does |
|---|---|
| `milk, eggs, 2 cucumbers` | Adds 3 items, reacts 👍 (no chat noise) |
| `חלב 3%, לחם אחיד` | Hebrew works too |
| 📷 photo of a handwritten list | Reads it, adds all items |
| 📷 photo of an empty package / product | Identifies the product (brand and size), adds it |
| `no need for eggs` / `remove eggs` | Removes the item |
| `bought the milk` | Marks it as done |
| Ordinary chat ("what time are you home?") | Ignored |

Confirmation stays small: an emoji reaction, or a short "Added: milk, eggs,
cucumbers ×2" with an **Undo** button when the bot wasn't sure.

### Shopping mode — `list`
The bot posts **one message** with an inline-keyboard button for each item,
grouped by aisle (produce, dairy, bakery…):

```
🛒 Shopping list (12 items)
[ ⬜ Milk 3% ×2 ]  [ ⬜ Eggs L ×12 ]
[ ✅ Cucumbers ×2 ]  [ ⬜ Bread ]
...
[ 🏁 Done shopping ]
```

- Tapping an item toggles ⬜/✅ by **editing the same message**, so you both
  see the cart update live if you split up in the store.
- **Done shopping** archives the checked items. Unchecked items stay for next
  time.
- Later upgrade: a **Telegram Mini App** (a small web page that opens inside
  Telegram) for a richer checklist: swipe to check, notes, item photos. The
  inline keyboard is good enough for the MVP.
  (Telegram's built-in checklists can't be used for this: bots can only send
  them through Business accounts, not in normal groups.)

### Store recommendation — `where` (also shown under `list`)
```
💰 Best options for your 12 items:
1. Rami Levy (Talpiot)      ₪243  · 6.1 km  · 12/12 items
2. Shufersal Deal (Center)  ₪258  · 1.2 km  · 12/12 items   ← closest, +₪15
3. Victory (Malha)          ₪251  · 4.0 km  · 11/12 (no Bamba Nougat)
```
You get three angles: the **cheapest**, the **closest one that's reasonably
priced**, and a **balanced** option. Missing items are marked clearly.

### Other commands
`clear`, `undo`, `history` (repeat a past basket), `staples` (auto-suggest
items you buy every week).

---

## 2. Architecture

```
Telegram group ──► Bot (webhook / long-poll)
                     │
                     ├─► Message parser (LLM: text + vision) ──► List service ──► SQLite
                     │                                              ▲
                     ├─► "list" renderer (inline keyboard) ─────────┘
                     │
                     └─► Recommender ──► Price DB (SQLite) ◄── Daily price ingester (cron)
                                              ▲                     │
                                         Product matcher      Chains' public price files
```

| Component | Choice | Why |
|---|---|---|
| Language | **Python 3.12** | Best ecosystem for the Telegram, LLM and price-scraping parts |
| Telegram | `python-telegram-bot` v21+ (async) | Mature, handles inline keyboards and callbacks well |
| Parsing | **Claude API**: a small/fast model for text, a vision-capable model for photos | Handles messy Hebrew and English, quantities and intent (add/remove/done), and reads handwriting |
| Storage | **SQLite** (one file) | Two users; no server to run |
| Price data | Israeli chains' transparency files, via the open-source [`il-supermarket-scraper`](https://github.com/OpenIsraeliSupermarkets) packages | Already handles each chain's quirky portal and format |
| Hosting | Small VPS (Hetzner/Oracle free tier) or a home Raspberry Pi | Always-on bot + nightly cron |
| Deploy | Docker Compose: `bot` and `ingester` | Simple to update |

### Telegram setup notes
- Create the bot with @BotFather, then **disable privacy mode**
  (`/setprivacy` → Disable). Otherwise the bot only sees commands, not
  ordinary messages.
- Allow only your group's `chat_id`, plus your two user IDs, so nobody else
  can use it.

---

## 3. The hard parts (and how to handle them)

### 3.1 Understanding messages
Send each message to the LLM with a strict JSON output schema:
```json
{"intent": "add|remove|check|ignore",
 "items": [{"name": "milk", "qty": 2, "unit": "L", "variant": "3%", "brand": null}]}
```
- Include the **current list** in the prompt so "remove the second one" and
  "make it 3 milks" work.
- Merge duplicates: "milk" plus "another milk" becomes qty 2.
- Photos: the same schema, with a vision call.

### 3.2 Matching "milk" to a real product (the key difficulty)
Price files list products by **barcode and chain-specific name**, for example
`חלב טרי 3% 1 ליטר תנובה`. "milk" doesn't map to a single product.

Approach:
1. **Learn your preferences.** The first time "milk" appears, the bot asks once
   with buttons: *Tnuva 3% 1L / Tara 3% 1L / Yotvata 1% 1L / any*. It remembers
   your answer (`item_alias` table), so after a few weeks it rarely asks.
2. **"Any" means a generic category.** For produce (cucumbers, tomatoes),
   compare the cheapest product in that category per kg at each store.
3. **Fuzzy and semantic search** over product names (rapidfuzz first;
   embeddings only if needed), with the LLM picking the best candidates.
4. **Barcodes are the join key across chains.** Branded products share an EAN
   barcode everywhere, which makes comparing chains exact.

### 3.3 Basket pricing per store
For each candidate store within *R* km of home:
- `total = Σ price(item) × qty`, applying simple promotions ("2 for ₪10",
  club prices flagged separately).
- Missing item: add a penalty (for example the median price at other stores)
  and show "11/12 items".
- Rank and pick three:
  - **Cheapest**: the lowest total.
  - **Closest**: the nearest store whose total is within X% of the cheapest.
  - **Balanced**: the best score from `total + distance_km × ₪/km` (a weight
    you set, for example ₪3/km).

Store addresses come from each chain's `Stores` file. Geocode them once
(OpenStreetMap Nominatim) and cache the coordinates.

### 3.4 Keeping price data small
Full national files are large. Only ingest stores within ~15 km of home, plus
the online stores (Shufersal Online, Rami Levy Online). That's a few dozen
stores, refreshed nightly, which fits easily in SQLite.

---

## 4. Data model (SQLite)

```
list_item(id, name, qty, unit, variant, status[open|checked|archived],
          added_by, source_msg_id, created_at, checked_at)
item_alias(alias, product_barcode NULL, category NULL, preferred_brand)  -- learned prefs
chain(id, name)
store(id, chain_id, store_code, name, address, lat, lon, distance_km, is_online)
product(barcode, name, manufacturer, unit_qty, unit_of_measure, category)
price(store_id, barcode, price, unit_price, updated_at)
promo(store_id, barcode, description, min_qty, promo_price, club_only, valid_to)
list_message(chat_id, message_id)   -- the live "list" message to edit on toggle
```

---

## 5. Milestones

| # | Milestone | Done when | Effort |
|---|---|---|---|
| **M1** | Bot skeleton | Bot joins the group, whitelists users, echoes messages | ½ day |
| **M2** | Text → list | Free text (Hebrew and English) adds, removes and checks items; stored in SQLite | 1–2 days |
| **M3** | `list` checklist | Inline keyboard with live toggling, grouped by category, "Done shopping" archives | 1 day |
| **M4** | Photos | Handwritten list and product photos add items | ½–1 day |
| **M5** | Price ingester | Nightly job loads prices and promotions for 3–4 chains near home | 2–3 days |
| **M6** | Product matching | Alias learning with the "which milk?" buttons; fuzzy search | 2–3 days |
| **M7** | Recommender | `where` shows the top 3 stores with total, distance and coverage | 1–2 days |
| **M8** | Polish | Staples suggestions, history, Mini App checklist, promotion handling | ongoing |

**M1–M4 give you a useful shared list in under a week.** Price
recommendations (M5–M7) are the larger, independent second phase.

---

## 6. Running costs

| Item | Cost | Notes |
|---|---|---|
| Telegram bot | Free | |
| Supermarket price files | Free | Published by law |
| Geocoding (OpenStreetMap) | Free | Store addresses are geocoded once, then cached |
| **Claude API** | **Under $1 a month** | Prepaid credits; the minimum top-up lasts months |
| **Hosting** | **$0–5 a month** | Oracle free tier = $0; Hetzner VPS ≈ €4–5; Raspberry Pi ≈ $60–100 once, plus electricity |
| Domain / HTTPS | Free | The bot uses long polling, so it needs no public URL |

**Claude API estimate** (assuming about 100 text messages and 20 photos a month):
- Text goes to Claude Haiku 5.5 ($0.10 / $0.50 per million input/output
  tokens). About 1.5K tokens in and 0.7K out per message comes to roughly
  $0.0005 each, or about **$0.05 a month**.
- Photos go to Claude Sonnet 5.5 ($2 / $10 per million tokens). About 3K
  tokens in and 1K out per photo comes to roughly $0.016 each, or about
  **$0.30 a month**. Haiku can also read photos; try it first and switch to
  Sonnet only if handwriting recognition is poor.
- Product matching adds a few cents a month, and less over time as the bot
  learns your preferences.
- Set a monthly spend limit in the Anthropic Console as a safety net.

---

## 6b. Zero-cost variant (no Claude API, no paid hosting)

**Hosting on the home PC that is always on.** The bot uses long polling: it
connects out to Telegram and asks for new messages. Nothing has to connect
in, so there is no port forwarding, public IP, domain or certificate.
- Run it with Docker Desktop (restart policy `always`), or as a service
  (systemd on Linux; NSSM or Task Scheduler on Windows).
- Turn off sleep and hibernate, and make sure the bot starts after a reboot
  (Windows Update reboots happen).
- Back up the SQLite file every night to a cloud folder (OneDrive or Google
  Drive).
- For a Mini App later, a free Cloudflare Tunnel provides HTTPS without
  opening any ports.

**Understanding messages without Claude: three tiers that can be combined**
1. **Rule-based parser (always on, free).** Split messages on commas and new
   lines, and read quantities ("2 milk", "milk x2"). Fixed prefixes handle
   the other actions: `-milk` or "remove milk" removes an item, and "bought
   milk" or ✅ marks it done. Item names are fuzzy-matched (rapidfuzz)
   against a dictionary built from the supermarket price files themselves,
   which also catches typos and handles Hebrew. This covers most messages if
   you both follow the simple format.
2. **Barcode photos instead of "AI vision".** Photograph the barcode on the
   empty package. `pyzbar`/`zxing` decodes it locally, and the barcode gives
   an exact product match in the price database. This is more accurate than
   an LLM for products.
   OCR (Tesseract, which supports Hebrew) can read *printed* lists, but it is
   poor at handwriting.
3. **Local LLM on the PC (optional).** Ollama running an open model handles
   free-form messages the rules can't parse. Hebrew quality depends on the
   model, and reading photos requires a vision model plus enough RAM, ideally
   with a GPU. Use it only as a fallback for unparsed messages.

What you lose without Claude: understanding of loose phrasing ("we're out of
the stuff for the kids' sandwiches") and reading of handwritten photos.

## 7. Open questions
1. **Country.** Is this Israel? If not, the price source changes: there might
   be a public API, a price-comparison site to scrape, or we may need to
   collect prices from your own receipts.
2. **Home location and radius.** Which chains do you actually consider? Are
   online orders an option?
3. **Club memberships.** Shufersal, Rami Levy, etc. Should club-only prices
   count?
4. **Hosting.** Do you have a VPS or Pi, or should we pick one?
5. **Language.** Should bot replies be in Hebrew, English, or match the
   sender's language?
