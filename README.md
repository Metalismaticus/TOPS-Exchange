# TOPS Exchange

**Auction price analytics for the TOPS Vintage Story server.**

Live dashboard: https://YOUR_USERNAME.github.io/tops-exchange

---

## What it does

Displays price history, top sellers, and item trends collected by the [AuctionFilter mod](https://mods.vintagestory.at/auction).  
Data is snapshot-based: the mod captures the auction listing when a player opens the Auction House. Duplicate snapshots of the same lot are deduplicated server-side.

---

## Repo structure

```
tops-exchange/
├── index.html        ← dashboard (GitHub Pages entry point)
├── icons/            ← item textures exported from Vintage Story (pixelated PNGs)
│   ├── knife-flint.png
│   ├── ingot-copper.png
│   └── ...
└── README.md
```

---

## Setup

### 1. GitHub Pages

1. Fork or clone this repo
2. Go to **Settings → Pages**
3. Source: **Deploy from branch** → `main` → `/ (root)`
4. Save — site is live at `https://YOUR_USERNAME.github.io/tops-exchange`

### 2. Connect Supabase (when mod data is ready)

Open `index.html` and edit the `CONFIG` block at the top of the `<script>`:

```js
const CONFIG = {
  useMockData: false,                         // ← switch off mock
  supabaseUrl: 'https://xxxx.supabase.co',    // ← your project URL
  supabaseAnonKey: 'eyJ...',                  // ← anon public key
  tableName: 'auction_listings',
};
```

### 3. Supabase table schema

```sql
create table auction_listings (
  id            bigserial primary key,
  item_code     text not null,          -- e.g. 'game:knife-flint'
  item_name     text,                   -- display name, sent by mod
  category      text,                   -- Tools / Items / Construction / ...
  price         integer not null,
  quantity      integer default 1,
  seller        text not null,
  expire_date   date,                   -- auction expiry, used for dedup
  collected_at  timestamptz default now(),
  server_id     text default 'tops'
);

-- Allow mod to insert, deny everything else from anon
alter table auction_listings enable row level security;

create policy "mod can insert"
  on auction_listings for insert
  to anon with check (true);

create policy "anyone can read"
  on auction_listings for select
  to anon using (true);

-- Index for fast date-range queries
create index on auction_listings (collected_at);
create index on auction_listings (item_code);
```

### 4. Item icons

Export textures from Vintage Story (look for `/debug exportatlas` or similar command in-game dev tools), then place individual PNGs in the `icons/` folder named exactly as the item code without the `game:` prefix:

```
icons/knife-flint.png
icons/ingot-copper.png
```

The dashboard will automatically find them by item code. Missing icons fall back gracefully to a `?` placeholder.

---

## Data flow

```
Player opens Auction House
        ↓
AuctionFilter mod captures listing snapshot
        ↓
HTTP POST → Supabase REST API (anon key, RLS insert-only)
        ↓
auction_listings table
        ↓
Dashboard fetches via Supabase REST (read, filtered by date/category)
        ↓
Client-side dedup + aggregation → charts rendered
```

---

## License

MIT
