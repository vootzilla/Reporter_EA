# Project Specification: Cloud Multi-Tenant MT5 Portfolio Dashboard

## 1. Architecture Overview
- **Hosting:** Static Single-Page Application (`index.html`) hosted on GitHub Pages / Cloudflare Pages (Zero backend server required).
- **Database & API:** Supabase PostgreSQL RPC endpoints accessed directly via `fetch()` from the browser.
- **Multi-Tenancy:** Users authenticate on the dashboard by entering their `Supabase Project URL`, `Supabase Anon Key`, `Owner Email`, and `Sync Key`. Credentials are saved in browser `localStorage`.

## 2. Supabase RPC Endpoints

### A. Fetch Portfolio Data
- **Endpoint:** `POST {SUPABASE_URL}/rest/v1/rpc/get_my_dashboard`
- **Headers:**
  - `apikey: {ANON_KEY}`
  - `Authorization: Bearer {ANON_KEY}`
  - `Content-Type: application/json`
- **Payload:** `{ "p_email": "user@example.com", "p_sync_key": "secret123" }`
- **Response Array Item Schema:**
  - `account_login` (number)
  - `broker_server` (string)
  - `port_alias` (string)
  - `raw_currency` (string, e.g., "USD", "USC")
  - `is_cent` (boolean)
  - `balance_usd` (number)
  - `equity_usd` (number)
  - `floating_pnl_usd` (number)
  - `floating_dd_pct` (number, negative value indicates drawdown)
  - `profit_day_usd` (number)
  - `profit_week_usd` (number)
  - `profit_month_usd` (number)
  - `open_positions` (number)
  - `updated_at` (ISO timestamp string)

### B. Delete Inactive Account
- **Endpoint:** `POST {SUPABASE_URL}/rest/v1/rpc/delete_mt5_account`
- **Headers:** Same as above
- **Payload:** `{ "p_email": "...", "p_sync_key": "...", "p_login": 123456, "p_server": "..." }`

## 3. UI/UX & Drag-and-Drop Reordering Requirements (`index.html`)
- **Tech Stack:** Single file `index.html` using Tailwind CSS (CDN), SortableJS (CDN: `https://cdn.jsdelivr.net/npm/sortablejs@latest/Sortable.min.js`), and Vanilla JS or Alpine.js.
- **Theme:** Mobile-first responsive Dark Mode (`bg-slate-950`, cards `bg-slate-900`, crisp typography for financial figures).

### A. Top Summary Cards (Aggregated across all active accounts of the user)
1. **Total Equity ($) & Total Balance ($)** (with total active ports count).
2. **Net Floating P/L ($) & Portfolio Floating DD (%)** = `((Total Equity - Total Balance) / Total Balance) * 100`.
3. **Total Realized Profit:**
   - Today ($ and % of Total Balance)
   - This Week ($ and % of Total Balance)
   - This Month ($ and % of Total Balance)

### B. Drag-and-Drop Portfolio Ordering & Sort Modes
1. **SortableJS Integration:**
   - Render accounts as a responsive grid of cards (or sortable rows) where each card has a unique composite ID: `${account_login}__${broker_server}`.
   - Include a dedicated, touch-friendly **Drag Handle (`⠿`)** on each card (`handle: '.drag-handle'`) so mobile users can scroll the page normally without accidentally dragging cards.
   - Add visual feedback during drag (`ghostClass: 'opacity-40'`, `chosenClass: 'ring-2 ring-indigo-500'`).
2. **Order Persistence & Auto-Refresh Merge Logic (CRITICAL):**
   - Save the user's custom array of composite IDs (`${account_login}__${broker_server}`) in `localStorage` under the key `mt5_custom_order_${owner_email}` whenever `onEnd` fires in SortableJS.
   - When the 5-second auto-refresh poll receives data from Supabase, **DO NOT reset the user's custom card order**. Map and sort the incoming data according to the saved `mt5_custom_order_${owner_email}` array.
   - Any newly discovered accounts not yet in the saved order array must be automatically appended to the bottom of the list and saved.
   - **Drag Lock:** Set a state flag `isDragging = true` on Sortable `onStart` and `false` on `onEnd`. While `isDragging === true`, skip DOM re-rendering from the 5-second polling timer so the card never jumps out of the user's finger mid-drag.
3. **Sort Mode Selector:**
   - Provide a segmented control / dropdown above the portfolio list with 4 modes:
     - `Custom Order (Drag & Drop)` (Default — enables drag handle)
     - `Worst Drawdown (%)` (Sorts by `floating_dd_pct` ascending)
     - `Highest Equity ($)` (Sorts by `equity_usd` descending)
     - `Profit Today ($)` (Sorts by `profit_day_usd` descending)

### C. Per-Account Card Details
- **Header:** Drag handle (`⠿`), `port_alias` (editable or clear title), `account_login`, `broker_server`, and a badge `USC->USD` if `is_cent === true`.
- **Heartbeat Status:** Calculate `seconds_ago = (Date.now() - new Date(updated_at).getTime()) / 1000`. If greater than 120 seconds, show a red `OFFLINE` badge with last seen time; otherwise show a pulsing green `LIVE` indicator.
- **
