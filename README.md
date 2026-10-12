# Cartly: Intelligent Shopping Assistant

A single-file, zero-dependency e-commerce storefront with a built-in AI-style shopping concierge. Everything (catalog, search, cart, price alerts, comparison, chat assistant) runs in the browser from one `index.html`.

![status](https://img.shields.io/badge/build-none%20required-brightgreen) ![deps](https://img.shields.io/badge/dependencies-0-blue) ![stack](https://img.shields.io/badge/stack-HTML%20%7C%20CSS%20%7C%20Vanilla%20JS-9c7c44)

---

## Table of contents

1. [Highlights](#highlights)
2. [Quick start](#quick-start)
3. [Project structure](#project-structure)
4. [Architecture](#architecture)
5. [Cartly Concierge (chat assistant)](#cartly-concierge-chat-assistant)
6. [Draggable launcher](#draggable-launcher)
7. [Data model](#data-model)
8. [Scoring and pricing logic](#scoring-and-pricing-logic)
9. [Persistence](#persistence)
10. [Customization guide](#customization-guide)
11. [Keyboard shortcuts](#keyboard-shortcuts)
12. [Browser support](#browser-support)
13. [Known limitations](#known-limitations)
14. [Roadmap](#roadmap)

---

## Highlights

| Area | What you get |
|---|---|
| Storefront | Hero banner slider, category strip, discount tiers, coupon cards, horizontal product rows, filterable grid |
| Catalog | 90+ products across 13 categories with brand, rating, MRP, discount and tags |
| Search and filters | Header search, category, brand, rating, max price slider, 6 sort modes |
| Cartly Concierge | Natural-language chat: search, budget bundles, gift finder, deals, compare, price alerts, cart control |
| Voice | Speech-to-text input and read-aloud replies (where the browser supports it) |
| Product details | 12-week price trend sparkline, buy/wait advice, pros and cons, rating breakdown, delivery check, similar items |
| Cart | Quantity controls, 3 coupons, free-delivery progress bar, personal spending limit tracker, mock checkout |
| Live prices | Random price drops every 25 s with toasts and user-set alerts |
| Personalization | Wishlist, compare tray (up to 3), recently viewed, light/dark theme |
| UI | Draggable "Ask Cartly AI" launcher, movable chat window, cute mascot, responsive down to phones |

---

## Quick start

No build step, no server and no package install.

```bash
# Option 1: just open it
open index.html            # macOS
xdg-open index.html        # Linux
start index.html           # Windows

# Option 2: serve locally (recommended for sharing on your network)
python3 -m http.server 8080
# then visit http://localhost:8080
```

> Keep `index.html` and the `images/` folder together. If an image fails to load, the card falls back to a drawn SVG illustration automatically.

---

## Project structure

```
cartly/
├── index.html            # the whole app: markup, CSS, JS, catalog
├── README.md             # this file
└── images/
    ├── banner-appliances.jpg
    ├── banner-phones.jpg
    ├── banner-beauty.jpg
    ├── banner-gaming.jpg
    └── banner-cartly-ai.jpg
```

Inside `index.html` the code is laid out in this order:

```
<style>      design tokens -> base -> layout -> components -> overrides -> chat + launcher
<body>       header | category strip | chat window | banner | rows | shop grid | cart drawer | modal | FAB | toast
<script>
  1. Catalog (P) + derived fields (discount d, MRP o)
  2. State (cart, wish, cmp, alerts, view, dsc, coupon, budget)
  3. NLU: parse() -> search() / combo()
  4. Chat: ask() -> reply() -> run() -> say() -> follow()
  5. Rendering: rb() banners, rrows() home rows, rg() grid, det() detail, rc() cart, showCmp()
  6. Event delegation (single click listener keyed by data-a)
  7. Draggable launcher + movable chat
</script>
```

---

## Architecture

### 1. System block diagram

```mermaid
flowchart LR
    subgraph Browser["Browser (single page)"]
        direction TB

        subgraph UI["Presentation layer"]
            HDR["Header<br/>search | offers | theme | bag"]
            CAT["Category strip"]
            BNR["Banner slider"]
            ROWS["Home rows<br/>deals | picks | steals"]
            GRID["Product grid<br/>+ filter bar"]
            DRW["Cart drawer"]
            MOD["Modal<br/>detail | compare | order"]
            CHAT["Concierge chat window"]
            FAB["Draggable FAB"]
        end

        subgraph CORE["Logic layer"]
            STATE[("App state<br/>cart | wish | cmp<br/>alerts | view | dsc<br/>coupon | budget")]
            NLU["NLU parser<br/>parse()"]
            SRCH["Search and rank<br/>search() | combo()"]
            INTENT["Intent router<br/>reply() | run()"]
            PRICE["Price engine<br/>tick() | sc()"]
            EVT["Event delegation<br/>data-a dispatcher"]
        end

        subgraph DATA["Data layer"]
            CATALOG[("Catalog P[]<br/>90+ products")]
            LS[("localStorage<br/>cart | wish | alerts<br/>fab pos | theme")]
            IMGS[("images/ + loremflickr<br/>+ SVG fallback")]
        end

        subgraph WEBAPI["Browser APIs"]
            SR["SpeechRecognition"]
            SS["speechSynthesis"]
            PTR["Pointer Events"]
            CLIP["Clipboard"]
        end
    end

    FAB --> CHAT
    CHAT --> INTENT --> NLU --> SRCH --> CATALOG
    INTENT --> STATE
    HDR --> EVT
    GRID --> EVT
    ROWS --> EVT
    DRW --> EVT
    EVT --> STATE
    STATE --> LS
    PRICE --> CATALOG
    PRICE --> GRID
    STATE --> GRID
    STATE --> DRW
    CATALOG --> GRID
    CATALOG --> ROWS
    IMGS --> GRID
    IMGS --> BNR
    CHAT <--> SR
    CHAT --> SS
    FAB --> PTR
    ROWS --> CLIP
```

### 2. Page view state machine

The same grid serves four modes. `rx()` decides whether the home rows or the results view is shown.

```mermaid
stateDiagram-v2
    [*] --> Home
    Home: Home<br/>banner + rows + grid
    Results: Search or category results
    Discount: Discount view (dsc)
    Assistant: Assistant results (view)

    Home --> Results: type in search / pick category
    Home --> Discount: tap discount tier / Offers / See all
    Home --> Assistant: chat returns products
    Results --> Home: Back to home
    Discount --> Home: clear / Back to home
    Assistant --> Home: clear / Back to home
    Results --> Discount: tap discount tier
    Discount --> Results: pick category
    Assistant --> Results: change a filter
```

### 3. Chat request lifecycle

```mermaid
sequenceDiagram
    actor U as Shopper
    participant FAB as Launcher
    participant C as Chat UI
    participant R as reply() router
    participant N as parse()
    participant S as search()/combo()
    participant G as Grid and state

    U->>FAB: tap (or drag first)
    FAB->>C: openChat() anchors near FAB
    U->>C: "gaming laptop and mouse under 80000"
    C->>C: show typing dots (650 ms)
    C->>R: reply(query)
    R->>R: check fixed intents (cart, deals, gift, compare...)
    R->>N: no match, parse(query)
    N-->>R: cats[], tags[], max, min, brand, sort
    alt more than one category and a budget
        R->>S: combo() builds a bundle
    else normal query
        R->>S: search() scores and ranks
    end
    S-->>R: ranked results with match %
    R->>G: set view, rg() refresh grid
    R->>C: say() text + product cards
    C->>C: follow() adds suggestion chips
    U->>C: tap "Add to bag" on a card
    C->>G: addCart(), toast, cart badge update
```

### 4. Intent routing (order matters)

```mermaid
flowchart TD
    A([User message]) --> B{greeting?}
    B -- yes --> B1[Welcome reply]
    B -- no --> C{help?}
    C -- yes --> C1[Show examples]
    C -- no --> D{clear cart?}
    D -- yes --> D1[Empty cart]
    D -- no --> E{show cart or checkout?}
    E -- yes --> E1[Open drawer + summary]
    E -- no --> F{coupon?}
    F -- yes --> F1[List codes]
    F -- no --> G{trending?}
    G -- yes --> G1[Top 4 by Cartly Score]
    G -- no --> H{gift?}
    H -- yes --> H1[Best rated per category under budget]
    H -- no --> I{alert + number?}
    I -- yes --> I1[Save price alert for first result]
    I -- no --> J{deals?}
    J -- yes --> J1[Top 4 discounts]
    J -- no --> K{add all / add N?}
    K -- yes --> K1[Add to cart]
    K -- no --> L{compare?}
    L -- yes --> L1[Open comparison modal]
    L -- no --> M{refine: cheaper / better rated?}
    M -- yes --> M1[Re-run last query with new bound]
    M -- no --> N{">1 category + budget?"}
    N -- yes --> N1[Bundle builder]
    N -- no --> O[Ranked search]
    O --> P{results?}
    P -- none --> P1[Suggest relaxing filters]
    P -- some --> P2[Top pick + reasons + cards]
```

### 5. Natural-language parsing

```mermaid
flowchart LR
    Q["Raw text<br/>'best phone under 20k with good camera'"] --> L[lowercase]
    L --> CATS["Category regex map<br/>CAT{}"]
    L --> PRICEX["Price regex<br/>under / below / above / Rs / k / lakh"]
    L --> TAGS["Tag list + synonym map<br/>cheap -> budget<br/>gamer -> gaming<br/>photo -> camera"]
    L --> BR["Brand match<br/>from catalog"]
    L --> SORT["Sort cues<br/>cheapest | top rated"]
    CATS --> O["Intent object<br/>cats | tags | max | min | brand | sort"]
    PRICEX --> O
    TAGS --> O
    BR --> O
    SORT --> O
    O --> U["'Understood' chips shown in chat"]
    O --> SR["search(o)"]
```

### 6. Cart and pricing flow

```mermaid
flowchart TD
    ADD[Add to bag] --> CART[(cart: id -> qty)]
    CART --> SUB[Subtotal = sum price x qty]
    SUB --> CPN{Coupon applied?}
    CPN -- SAVE10 --> D1[10% off]
    CPN -- WELCOME500 --> D2{subtotal >= 3000?}
    D2 -- yes --> D3[Rs 500 off]
    D2 -- no --> D4[No discount + toast hint]
    CPN -- FEST20 --> D5[20% off, capped at Rs 2000]
    CPN -- none --> D6[No discount]
    D1 --> TOT[Total]
    D3 --> TOT
    D4 --> TOT
    D5 --> TOT
    D6 --> TOT
    TOT --> LIM{Spending limit set?}
    LIM -- yes --> BAR[Budget progress bar<br/>over / under limit]
    LIM -- no --> SKIP[ ]
    SUB --> SHIP{subtotal >= 5000?}
    SHIP -- yes --> FREE[Free delivery unlocked]
    SHIP -- no --> NEED[Add Rs X more for free delivery]
    TOT --> CO[Checkout]
    CO --> ORD[Order confirmed + timeline]
    ORD --> CLR[Clear cart + coupon]
```

### 7. Live price ticker and alerts

```mermaid
sequenceDiagram
    participant T as setInterval 25s
    participant P as tick()
    participant CAT as Catalog
    participant UI as Grid / cart / toast
    participant AL as alerts{}
    participant CH as Chat

    T->>P: fire
    P->>CAT: pick random product, compute 3-8% drop
    alt new price below 70% of MRP or not lower
        P-->>T: skip
    else valid drop
        P->>CAT: update price, discount, mark dn
        P->>UI: rg(), rc(), toast "Price drop"
        P->>AL: target set for this product?
        alt price <= target
            P->>CH: say() "Price alert" + product card
        end
    end
```

---

## Cartly Concierge (chat assistant)

The assistant is rule-based and runs fully offline. It does not call any external AI service.

### Things you can say

| Intent | Example |
|---|---|
| Search with budget | `best phone under 20k with good camera` |
| Budget bundle | `gaming laptop and mouse under 80000` |
| Gift ideas | `gift ideas under 3000` |
| Deals | `show deals` |
| Trending | `what is trending` |
| Compare | `compare` (uses last results or your compare picks) |
| Add to bag | `add 1`, `add first`, `add all` |
| Refine | `cheaper`, `better rated` |
| Price alert | `alert me below 20000` |
| Cart | `show cart`, `clear cart` |
| Coupons | `coupons` |
| Help | `help` |

### What the assistant shows

- **Understood chips**: the filters it extracted from your sentence.
- **Top pick with reasons**: for example "Rs 3,000 under your budget, matches camera, 4.6 star rating".
- **Product cards** with price, discount, Cartly Score, match percentage and quick actions.
- **Follow-up chips**: cheaper, better rated, compare, add to bag, set price alert.

### Voice

- Mic button: speech-to-text (`en-IN`) using the Web Speech API.
- Speaker button: read replies aloud.
- Both are feature-detected and degrade gracefully.

---

## Draggable launcher

The "Ask Cartly AI" button and the chat window are movable.

```mermaid
flowchart TD
    PD[pointerdown on FAB] --> CAP[setPointerCapture]
    CAP --> MV{pointer moved > 6px?}
    MV -- no --> REL1[pointerup]
    REL1 --> CLK[click -> openChat]
    MV -- yes --> DRAG[placeFab clamp to viewport]
    DRAG --> REL2[pointerup]
    REL2 --> SAVE[(save position to localStorage)]
    REL2 --> GUARD[dragged flag blocks the next click for 80 ms]

    CLK --> POS{mobile width?}
    POS -- yes --> SHEET[Bottom sheet, no drag]
    POS -- no --> ANC[Anchor chat to FAB<br/>left or right by half of screen<br/>above or below by half of screen]
    ANC --> HDRD[Header drag moves chat freely]
```

Behavior details:

- Free placement anywhere on screen; position is clamped so the button never leaves the viewport.
- Position survives reloads (`cartly_fab`).
- A tap and a drag are separated by a 6 px movement threshold, so dragging never accidentally opens the chat.
- When the chat is open the launcher fades out; closing the chat brings it back in place.
- On screens 700 px wide or less the chat becomes a bottom sheet.
- `Esc` closes the chat.

---

## Data model

### Catalog entry

Products are defined as compact arrays and expanded at load time.

```js
// [name, category, brand, price, rating, (unused), tags]
["iPhone 15", "Phones", "Apple", 69900, 4.7, "", "premium camera"]
```

Expanded object:

```js
{
  id: 0,                 // array index
  n: "iPhone 15",        // name
  c: "Phones",           // category
  b: "Apple",            // brand
  p: 69900,              // current price (changes with price drops)
  r: 4.7,                // rating
  t: "premium camera",   // space-separated tags
  d: 15,                 // discount % (derived from a price-band pattern)
  o: 82240,              // MRP derived from price and discount
  dn: 1                  // set to 1 after a live price drop
}
```

### Entity relationship overview

```mermaid
erDiagram
    PRODUCT ||--o{ CART_LINE : "added as"
    PRODUCT ||--o{ WISHLIST_ITEM : "saved as"
    PRODUCT ||--o| PRICE_ALERT : "watched by"
    PRODUCT }o--|| CATEGORY : "belongs to"
    PRODUCT }o--o{ TAG : "described by"
    CART ||--o{ CART_LINE : contains
    CART }o--o| COUPON : "may apply"

    PRODUCT {
        int id
        string name
        string brand
        int price
        int mrp
        float rating
        int discount
    }
    CART_LINE {
        int productId
        int qty
    }
    PRICE_ALERT {
        int productId
        int targetPrice
    }
    COUPON {
        string code
        string rule
    }
```

### Runtime state

| Variable | Type | Purpose |
|---|---|---|
| `cart` | `{id: qty}` | Items in the bag |
| `wish` | `number[]` | Saved product ids |
| `cmp` | `number[]` | Up to 3 products to compare |
| `alerts` | `{id: price}` | Price-drop targets |
| `view` | `{ids, pct?}` or `null` | Assistant result set shown in the grid |
| `dsc` | `{m, u, l}` or `null` | Active discount filter (min %, max price, label) |
| `coupon` | string or `null` | Applied coupon |
| `budget` | number | Personal spending limit |
| `last`, `lastO` | arrays, object | Last results and last parsed query, used for follow-ups |

---

## Scoring and pricing logic

### Cartly Score (0-99)

Shown on cards and used by "Trending" and the "Cartly Score" sort.

```
score = min(99, round(
    rating/5 * 62
  + discount% * 1.1
  + (1 - price / maxPriceInCategory) * 14
))
```

It rewards high ratings first, then bigger discounts, then value within the category.

### Match percentage (assistant)

```
raw   = rating * 10
      + 12 per matched tag
      + (price / budget) * 6          // only when a budget is given
max   = 50 + 12 * tagCount + 6
match = min(99, round(raw / max * 100))
```

### Bundle builder

For queries that name more than one category and a budget, `combo()`:

1. Walks the categories in order.
2. Reserves the cheapest item for every category still to come.
3. Picks the highest-rated product that fits the remaining money.
4. Falls back to the cheapest option when nothing fits, and says so.

### Smart compare

Value score is `rating / price * 10000`. The modal highlights the lowest price, best rating and best value, and writes a one-line verdict.

---

## Persistence

| Key | Contents |
|---|---|
| `cartly_cart` | Cart contents |
| `cartly_wish` | Wishlist |
| `cartly_al` | Price alerts |
| `cartly_fab` | Launcher position `{x, y}` |
| `cartly_theme` | `light` or `dark` |

All access is wrapped in `try/catch`, so the site still works in private windows where storage is blocked.

---

## Customization guide

### Replace product photos

Near the top of the image section:

```js
const IMG = {};
// IMG[productId] = 'images/iphone15.jpg';
IMG[0] = 'images/iphone15.jpg';
```

Lookup order: `IMG[id]` -> loremflickr keyword photo -> drawn SVG fallback.

Tips: use square or 4:3 images around 800x600 px, plain light background, under 300 KB each.

### Replace banners

```js
const BIMG = {
  0: 'images/banner-appliances.jpg',
  1: 'images/banner-phones.jpg',
  2: 'images/banner-beauty.jpg',
  3: 'images/banner-gaming.jpg',
  4: 'images/banner-cartly-ai.jpg'
};
```

Use wide images around 1600x700 px with the subject on the right. The left side carries the headline text and gets a dark gradient overlay.

### Edit banner text

Each entry in the `BN` array is:

```js
[title, subtitle, buttonLabel, category, svgKey, hue, gradient, imageKeyword, badge]
```

An empty category makes the button open the chat instead of a category.

### Add or edit products

Append to the array `P`:

```js
["Product name", "Category", "Brand", 1999, 4.3, "", "budget wireless battery"]
```

- Use an existing category name so it appears in the strip and filters.
- Tags drive assistant matching. Known tags: gaming, battery, camera, budget, premium, wireless, fitness, student, noise, fast, inverter, energy, 4k, oled, luxury, organic, insulated, steel, glass, multiplayer, family, makeup, skincare, fragrance, haircare.
- To add a new category, also add it to the `CAT` regex map (so the chat understands it), the `CK` icon map, and `artKey()`.

### Coupons

Defined in three places: the `CPN` array (cards), the discount logic in `rc()`, and the list inside the `#ap` click handler.

| Code | Rule |
|---|---|
| `SAVE10` | 10% off |
| `WELCOME500` | Rs 500 off at Rs 3,000 or more |
| `FEST20` | 20% off, up to Rs 2,000 |

### Theme and brand colors

All colors are CSS variables on `:root`; the last `:root` block in the stylesheet wins.

```css
--bg --c --t --m --a --b --g --on
```

`--a` is the maroon accent. Dark values live under `@media (prefers-color-scheme: dark)` and `:root[data-theme=dark]`.

### Chat window size

```css
#chat { width: min(440px, calc(100vw - 24px)); height: min(700px, calc(100vh - 110px)); }
```

### Price-drop frequency

```js
setInterval(tick, 25000);   // milliseconds
```

---

## Keyboard shortcuts

| Key | Action |
|---|---|
| `/` | Focus the chat input |
| `Esc` | Close modal, cart drawer, then chat |
| `Enter` | Send chat message / run header search |

---

## Browser support

| Browser | Support |
|---|---|
| Chrome, Edge (current) | Full, including voice input |
| Safari 16.4+ | Full (watch-category regex needs lookbehind support) |
| Firefox (current) | Full except speech recognition |
| Mobile Chrome / Safari | Full, chat becomes a bottom sheet |

Features used: CSS `color-mix()`, `:has()`, CSS grid, Pointer Events, `localStorage`, optional Web Speech API.

---

## Known limitations

- **Prototype data**: prices, ratings and reviews are generated or hard-coded. There is no backend, real checkout or payment.
- **Discounts and MRP** are derived from a deterministic pattern, not stored data.
- **Chat is rule-based**: it understands the supported phrasing, not free-form conversation.
- **Placeholder photos** come from a keyword photo service and may not match the product; replace them via `IMG`.
- **Price ticker** changes in-memory prices only; they reset on reload.
- **Voice input** needs HTTPS (or localhost) and a supporting browser.

---

## Roadmap

```mermaid
timeline
    title Suggested next steps
    Now : Replace placeholder photos with your own product images
        : Tune catalog and coupons
    Next : Move catalog to a JSON file or API
         : Real search index (fuzzy matching)
         : Persist order history
    Later : Backend with auth and real checkout
          : LLM-powered concierge behind the same UI
          : Analytics on chat queries and conversions
```

---

## Credits

Built with plain HTML, CSS and JavaScript. Fonts: Fraunces and Inter (Google Fonts). Placeholder photos: loremflickr.
