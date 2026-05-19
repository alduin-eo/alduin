## Issue #12
# Phase 2: Implement Alduin cancel handler (player self-service)

## Goal

Handle `Alduin::Cancel` (PacketAction: `Spec`) — allow players to cancel their own pending transactions.

## Flow

1. Player sends `Alduin::Cancel { transaction_id }`
2. **Rate limit check**: If less than 5 seconds since last Alduin request → silently ignored
3. Server validates:
   - Transaction exists → else `Alduin::Reply { TransactionNotFound }`
   - Transaction belongs to this character → else `Alduin::Reply { NotYourTransaction }`
   - Transaction is in `pending` status → else `Alduin::Reply { TransactionNotPending }`
4. If deposit: no item changes (nothing was ever added)
5. If withdraw: refund `amount` Alduin items (ID 491) to player **inventory only** (not bank). If inventory is full (2B cap), refund only what fits; record actual amount in `settled_amount` and notify staff via Discord of shortfall.
6. Mark transaction as cancelled
7. Send `Alduin::Reply { Notify: { transaction_id, status: cancelled, total_alduin } }` to client
8. Send Discord notification that player self-cancelled

## Discord Webhook Format

```json
{
  "content": "🔕 **Alduin Transaction Cancelled**",
  "embeds": [{
    "title": "Player cancelled their own pending transaction",
    "fields": [
      {"name": "Character", "value": "<name> (ID: <id>)"},
      {"name": "Amount", "value": "<amount>"},
      {"name": "Type", "value": "<deposit/withdraw>"},
      {"name": "Transaction ID", "value": "<id>"}
    ],
    "color": 16744192
  }]
}
```

## Acceptance Criteria

- [ ] `Alduin::Spec` handler registered and receiving packets
- [ ] 5-second rate limiting between requests (silently ignored on cooldown)
- [ ] Ownership validation (transaction belongs to requesting character)
- [ ] Status validation (only pending transactions can be cancelled)
- [ ] `Alduin::Reply` sent for validation failures (TransactionNotFound, NotYourTransaction, TransactionNotPending)
- [ ] Deposit cancel: no item changes
- [ ] Withdraw cancel: refund items to inventory only
- [ ] `settled_amount` overflow handling for withdraw cancel refunds (cap at 2B, notify staff of shortfall)
- [ ] `Alduin::Reply { Notify }` sent to client with updated `total_alduin`
- [ ] Transaction status updated to cancelled
- [ ] Discord notification fires for player self-cancel (same format as staff-initiated cancel, but indicates 'Player cancelled')



## Issue #11
# Phase 4: Add Alduin Wallet nav button to in-game HUD

## Goal

Add a new nav button to the in-game HUD that opens the Alduin Wallet dialog.

## Location

In-game nav buttons are in `src/ui/containers/in-game.tsx`. The wallet button should be placed alongside existing buttons (inventory, spells, bank, skills, etc.).

## Implementation

1. Add `wallet` to the dialog type list (line ~50 area)
2. Add wallet icon to the nav bar — use a coin/purse icon or similar
3. Wire click handler to open the Alduin Wallet dialog
4. Button should only be visible in `InGame` state

## Icon
- Use an existing icon from the game assets or a custom SVG
- Style consistent with other nav buttons (DaisyUI button classes)

## Integration
- Import `AlduinWalletDialog` component
- Add to the dialog switch statement
- Subscribe to any open/close signals from the Alduin controller if needed

## Acceptance Criteria
- [ ] Nav button visible in in-game HUD
- [ ] Click opens Alduin Wallet dialog
- [ ] Button styled consistently with other nav buttons
- [ ] Only visible when in-game (not on login/character select)



## Issue #10
# Phase 4: Build Alduin Wallet UI dialog

## Goal

Create the Alduin Wallet dialog with balance display, slider-based amount selection for deposit/withdraw, wallet address input, paginated transaction history, and ability to cancel pending transactions.

## UI Components

### `src/ui/in-game/dialogs/alduin-wallet-dialog.tsx`

**Layout (Preact + Tailwind/DaisyUI):**

```
┌──────────────────────────────────────┐
│  🏦 Alduin Wallet                [✕] │
├──────────────────────────────────────┤
│  Balance: 1,234 Alduin               │
│                                      │
│  Amount: [====|==========] 500       │
│  (min: 1, max: 1000, step: 1)        │
│                                      │
│  Wallet Address:                     │
│  ┌────────────────────────────────┐  │
│  │ 0x1234...5678             [✕] │  │
│  └────────────────────────────────┘  │
│  (prepopulated with most recent)     │
│                                      │
│  [ Deposit ]  [ Withdraw ]           │
│                                      │
├──────────────────────────────────────┤
│  Transaction History                 │
│  ┌────────────────────────────────┐  │
│  │ [Approved] +50 | 0xabc...def │  │
│  │ 2026-05-19 14:32               │  │
│  ├────────────────────────────────┤  │
│  │ [Pending] -20  | 0x123...456 │  │
│  │ 2026-05-18 09:15    [Cancel]  │  │
│  └────────────────────────────────┘  │
│  [‹] Page 1/5 [›]                    │
└──────────────────────────────────────┘
```

## Error Display

- Server validation errors (via `Alduin::Reply` with error values like `AmountBelowMin`) will be displayed using **AlertController** on top of the dialog — errors are **NOT** part of dialog state.
- AlertController manages its own lifecycle: when the player closes and reopens the dialog, any previous error is already gone.
- User message mapping handled by controller (see issue #9).

## Cancel Button

- Only shown on pending transactions
- Clicking triggers a confirmation dialog with conditional text based on transaction type:
  - **Deposit:** "Cancel this deposit request? No items were sent."
  - **Withdraw:** "Cancel this withdrawal? Items will be refunded to your inventory."
- Calls `controller.cancelPending(transactionId)`
- On success, transaction updates to cancelled, balance/slider max refresh
- On error (e.g. already resolved), show error alert via AlertController

## Amount Slider

- **Range:** `[deposit_min, deposit_max]` or `[withdraw_min, withdraw_max]` from server config
- **Withdraw max clamped to:** `min(config_withdraw_max, current_inventory_balance)`
- **Deposit max:** `config_deposit_max` (no inventory dependency)
- Slider replaces separate amount input dialog — selection is immediate
- Value displayed next to slider
- No separate "confirm amount" step — clicking Deposit/Withdraw uses current slider value
- **Dynamic updates:** The dialog should listen for `InventoryController` events to dynamically update the withdraw slider max when inventory changes (e.g., player picks up or drops items while the dialog is open).

## Wallet Address Input

- Text input field below slider
- Prepopulated with `lastWalletAddress` from controller state
- Clear button (✕) to empty the field
- Required for both deposit and withdraw actions

## Server Deposit Wallet Display

- Server's deposit wallet address shown as a read-only, copyable text near the deposit button or as an info line
- Visible when user is considering a deposit

## Transaction History
- 20 entries per page
- Color-coded status: green=approved, yellow=pending, red=cancelled
- Newest first
- Page navigation with prev/next buttons
- Each entry shows: status, action (deposit/withdraw with +/- prefix), amount, wallet address (truncated), timestamp
- Pending entries show [Cancel] button

## Integration
- Register dialog in `in-game.tsx` alongside bank, inventory, locker dialogs
- Open/close via controller subscription (like bank/locker pattern)

## Acceptance Criteria
- [ ] Dialog component created with all sections
- [ ] Slider replaces amount input dialog, bounded by server config min/max
- [ ] Withdraw slider max dynamically clamped to current inventory balance
- [ ] Withdraw slider max updates dynamically when inventory changes (via InventoryController events)
- [ ] Wallet address input with prepopulate and clear
- [ ] Server deposit wallet address visible and copyable
- [ ] Deposit/withdraw buttons use current slider value (no popup)
- [ ] Pending transactions show [Cancel] button with confirmation
- [ ] Cancel confirmation text is conditional based on transaction type (deposit vs withdraw)
- [ ] Cancel refunds items (for withdraws), updates UI
- [ ] Error alerts shown via AlertController (not dialog state)
- [ ] Transaction history paginated and styled with wallet addresses
- [ ] Dialog registered and openable from in-game HUD
- [ ] Status colors consistent for transaction states




## Issue #9
# Phase 4: Add Alduin controller to eoweb

## Goal

Create an Alduin controller in eoweb that handles all Alduin wallet packets.

## Controller: `src/controllers/alduin-controller.ts`

### State
- `totalAlduin: number` — total Alduin items in inventory (from server)
- `pendingWithdrawCount: number` — pending outgoing transactions
- `depositWallet: string` — server's deposit wallet address (from config)
- `depositMin: number`, `depositMax: number` — slider bounds for deposits
- `withdrawMin: number`, `withdrawMax: number` — slider bounds for withdrawals
- `lastWalletAddress: string` — most recent wallet address used (for prepopulate)
- `history: TransactionEntry[]` — current page of transaction history
- `currentPage: number`
- `totalPages: number`

### Events (subscribe pattern)
- `subscribeWallet(callback: (totalAlduin, pendingCount, depositWallet, depositMin, depositMax, withdrawMin, withdrawMax) => void)`
- `subscribeHistory(callback: (page, totalPages, entries) => void)`
- `subscribeTransactionUpdate(callback: (transactionId, status, totalAlduin) => void)`
- `subscribeError(callback: (reply: AlduinReply) => void)`

### Methods
- `requestWalletInfo()` — sends `Alduin::Request { page: 0 }`
- `requestHistory(page: number)` — sends `Alduin::Request { page }`
- `deposit(amount: number, walletAddress: string)` — sends `Alduin::Deposit { amount, wallet_address }`
- `withdraw(amount: number, walletAddress: string)` — sends `Alduin::Withdraw { amount, wallet_address }`
- `cancelPending(transactionId: number)` — sends `Alduin::Cancel { transaction_id }`

## Packet Handler: `src/handlers/alduin.ts`

Register handler for `Alduin::Reply` (PacketAction: `Reply`):
- Switch on `reply`:
  - `Wallet` → update balance, pending count, config bounds, then if `page` > 0 update history state
  - `Notify` → update transaction status, emit event, show notification, use `total_alduin` to update inventory display
  - Error values (`AmountBelowMin`, `AmountAboveMax`, etc.) → emit error event, show user-facing message

## Error Message Mapping
| Reply Value | User Message |
|-------------|-------------|
| `AmountBelowMin` | "Amount is below minimum allowed" |
| `AmountAboveMax` | "Amount exceeds maximum allowed" |
| `InsufficientFunds` | "You don't have enough Alduin" |
| `InvalidWalletAddress` | "Please enter a valid wallet address" |
| `TransactionNotFound` | "Transaction not found" |
| `TransactionNotPending` | "This transaction has already been resolved" |
| `NotYourTransaction` | "You cannot cancel this transaction" |
| `AlreadyHasPending` | "You already have a pending transaction of this type" |

## Registration
- Add `registerAlduinHandlers` to `src/handlers/index.ts`
- Add `AlduinController` to `Client` class
- Register in `src/main.tsx` or wherever controllers are initialized

## Acceptance Criteria
- [ ] Controller created with state management including config bounds for sliders
- [ ] Single packet handler for `Alduin::Reply` with switch on reply value
- [ ] `Wallet` handler always updates balance and config bounds
- [ ] Error values handled directly (no separate Error case) — map reply values to user-facing messages
- [ ] Controller added to Client class
- [ ] Subscribe pattern consistent with existing controllers
- [ ] `requestWalletInfo()` called on in-game entry
- [ ] `Notify` handler uses `total_alduin` to sync inventory count
- [ ] Deposit/withdraw methods accept amount (from slider) and walletAddress
- [ ] `cancelPending` method sends cancel packet



## Issue #8
# Phase 3: Create Alduin transaction Discord bot

## Goal

Build a standalone Discord bot that allows staff to manage pending Alduin transactions via slash commands. The bot communicates with reoserv via its HTTP API — it does **not** touch the database directly.

## Commands

| Command | Description |
|---------|-------------|
| `/alduin pending` | List all pending transactions with approve/cancel buttons |
| `/alduin approve <id>` | Approve a transaction by ID |
| `/alduin cancel <id>` | Cancel a transaction by ID |

## Button Interactions

Pending transaction embeds should have **Approve** (green) and **Cancel** (red) buttons that staff can click directly.

## Embed Refresh Behavior (Design Decision)

On approve or cancel, the bot must perform **both** actions:

1. **Edit the original embed** — update it in-place to reflect the result (approved/cancelled + any error message). This keeps the channel history clean and avoids orphaned stale embeds.
2. **Send a new confirmation message** — a fresh message in the channel confirming the action was completed, referencing the transaction ID and outcome.

This dual approach ensures: (a) the original embed is no longer actionable and clearly shows the final state, and (b) staff get an explicit confirmation without needing to scroll back to find the edited embed.

## Player Self-Cancel Notification (Design Decision)

When a player self-cancels their transaction on the server side, staff should be notified in the Discord channel. The notification uses the **same embed format** as staff-initiated cancels, but with the status text set to **"Player cancelled"** instead of "Cancelled by staff" (or similar wording). This keeps staff aware of all transaction lifecycle events without needing a separate notification channel.

## Bot Configuration

The bot reads its own configuration (env vars or config file) with the following keys:

| Key | Description |
|-----|-------------|
| `api_url` | Base URL of the reoserv HTTP API (e.g., `http://localhost:3000`) |
| `api_token` | Bearer token for authenticating API requests |
| `allowed_role_id` | Discord role ID that is permitted to use alduin commands and buttons |
| `channel_id` | Discord channel ID where pending transaction embeds are posted |

**Note on role restriction (Design Decision):** The `allowed_role_id` is configured in the **bot's own config**, not in reoserv's config. The bot performs the role check locally before processing any command or button interaction. This keeps role management decoupled from the game server and allows the bot admin to change permissions without touching reoserv.

## Bot Requirements

- **Language:** Python (discord.py) or TypeScript (discord.js)
- **Communication:** HTTP API calls to reoserv (no database access)
- **Role restriction:** Only users with the configured `allowed_role_id` can interact (checked by the bot locally)
- **API auth:** Includes configured `api_token` in request headers as `Authorization: Bearer ***`
- **Health check on startup:** Before processing any commands, the bot calls `GET /api/alduin/health` to verify the API token is valid and the server is reachable. If the health check fails, the bot logs an error and refuses to process commands until connectivity is restored.

## Startup Health Check (Design Decision)

On startup, the bot calls `GET /api/alduin/health` to verify:
1. The reoserv API is reachable
2. The configured `api_token` is accepted

If the health check fails, the bot should log a warning/error and **not** process any alduin commands or button interactions until the API is confirmed healthy. This prevents confusing error states where commands silently fail due to auth or connectivity issues.

## Architecture

```
Discord Bot ──HTTP API──► reoserv (handles all DB operations)
```

The bot sends API requests to reoserv; reoserv handles database reads/writes and item management. This keeps the database internal to the game server.

## API Endpoints (reoserv provides)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/alduin/health` | Health check — verifies API token and server readiness |
| `GET` | `/api/alduin/pending` | List all pending transactions |
| `POST` | `/api/alduin/:id/approve` | Approve a transaction |
| `POST` | `/api/alduin/:id/cancel` | Cancel a transaction |

All endpoints require `Authorization: Bearer ***` header.

### Response Formats

**`GET /api/alduin/health`**
```json
{ "status": "ok" }
// or
{ "error": "Unauthorized" }
```

**`GET /api/alduin/pending`**
```json
{
  "transactions": [
    {
      "id": 42,
      "character_id": 7,
      "character_name": "Sorokya",
      "action": "deposit",
      "amount": 500,
      "wallet_address": "0x1234...5678",
      "created_at": "2026-05-19T14:32:00Z"
    }
  ]
}
```

**`POST /api/alduin/:id/approve`** — returns success/error
```json
{ "status": "ok" }
// or
{ "error": "Transaction not found" }
```

**`POST /api/alduin/:id/cancel`** — returns success/error
```json
{ "status": "ok" }
// or
{ "error": "Transaction not pending" }
```

## Embed Fields (all pending transactions)
- **Character** — name (ID: character_id)
- **Wallet** — wallet_address
- **Amount** — amount
- **Type** — Deposit or Withdraw
- **Transaction ID** — id

## Flow

1. Bot starts, calls `GET /api/alduin/health` to verify API connectivity and token validity
2. If healthy, bot fetches pending transactions via API
3. Sends embeds with Approve/Cancel buttons to configured channel
4. Staff clicks Approve → bot POSTs to `/api/alduin/:id/approve`
5. reoserv performs all DB + inventory operations, returns result
6. Bot **edits the original embed** to show result AND **sends a new confirmation message**
7. If a player self-cancels on the server, bot is notified and posts a cancel embed with "Player cancelled" status

## Acceptance Criteria
- [ ] Bot connects to Discord and responds to slash commands
- [ ] Role-based access control (checked against bot's `allowed_role_id` config)
- [ ] Pending transactions fetched via HTTP API (no DB access)
- [ ] Embeds show: character id, name, wallet address, amount, type
- [ ] Approve/cancel via HTTP API calls to reoserv
- [ ] Bot edits original embed AND sends confirmation message after action
- [ ] Embeds update to reflect action result
- [ ] Startup health check verifies API token before processing commands
- [ ] Staff notified when player self-cancels (same embed format, "Player cancelled" text)



## Issue #7
# Phase 2: Add Discord webhook config for Alduin notifications

## Goal

Add configuration section in `Config.toml` for Alduin feature, including an HTTP API listener for the Discord bot to communicate with, server deposit wallet address, and configurable min/max amounts for deposit/withdraw sliders.

## Config Addition

```toml
[alduin]
# Server's deposit wallet address shown to players for deposit requests
deposit_wallet = ""

# Slider bounds for deposit requests (items per transaction)
deposit_min = 1
deposit_max = 1000

# Slider bounds for withdrawal requests (items per transaction)
withdraw_min = 1
withdraw_max = 1000

# Discord webhook URL for Alduin transaction notifications
# Leave empty to disable notifications
discord_webhook = ""

# Role or user to mention in notifications (e.g., "@Staff" or role ID)
discord_mention = ""

# HTTP API listener for Discord bot communication
# Leave empty to disable (Discord bot will not work)
api_listen = "127.0.0.1:9999"

# Secret token for authenticating Discord bot API requests
api_token = ""
```

## Implementation

1. Add `[alduin]` section to `config/Config.toml`
2. Add parsing in config structs
3. Create `send_discord_webhook` utility function (reuse existing Discord integration patterns)
4. Call from deposit and withdraw handlers
5. Include `deposit_wallet` and min/max bounds in `Alduin::Reply { Wallet }` for client slider setup
6. Set up HTTP API listener (likely using `axum` or `actix-web`) for Discord bot to call
7. Implement `GET /api/alduin/health` health check endpoint that validates the API token
8. Implement remaining API endpoints (see issue #8)

## API Endpoints

### `GET /api/alduin/health`

Health check endpoint that verifies the API token.
- Returns **200** if token is valid
- Returns **401** if token is invalid or missing
- Used by Discord bot to verify connectivity on startup

## Discord Webhook Embed Fields (all transactions)
- Character name (ID)
- Wallet address (player's external address)
- Amount
- Transaction type (Deposit / Withdraw)
- Transaction ID

## Acceptance Criteria
- [ ] Config section added and parsed with `deposit_wallet`, `discord_webhook`, min/max bounds, `api_listen`, `api_token`
- [ ] Webhook disabled when URL is empty
- [ ] Notifications include: character id, name, wallet address, amount, type
- [ ] Optional mention of staff role configured
- [ ] `deposit_wallet` and min/max bounds included in wallet info reply
- [ ] HTTP API listener starts on configured address
- [ ] API requests require token authentication
- [ ] `GET /api/alduin/health` endpoint returns 200 with valid token and 401 with invalid/missing token



## Issue #6
# Phase 2: Implement Alduin wallet info & transaction history handlers

## Goal

Handle `Alduin::Request` packets, always returning wallet info (derived from inventory) and optionally paginated transaction history in a unified `Alduin::Reply { Wallet }` response.

## Request (`Alduin::Request { page }`)

- `page` (char) — 0 = wallet info only, ≥1 = wallet + history page

## Response (`Alduin::Reply { Wallet }`)

Always includes:
- `balance` — total count of item ID 491 in player's inventory only (bank excluded)
- `pending_count` — number of pending outgoing (withdrawal) transactions
- `deposit_wallet` — server's deposit wallet address (from config), shown in UI for deposit requests

When `page` ≥ 1, also includes:
- `page` — requested page number
- `total_pages` — total available pages
- `transactions` — array of up to 20 entries per page, newest first (empty if no results)

Each `TransactionEntry` contains:
- `id` — unique transaction ID
- `timestamp` — UNIX timestamp
- `action` — deposit or withdraw (`TransactionAction` enum)
- `amount` — number of Alduin
- `wallet_address` — the player's external wallet address
- `status` — pending, approved, or cancelled (`TransactionStatus` enum)

## Pagination
- 20 entries per page
- Newest first (ORDER BY created_at DESC)
- Return empty array if no results

## Acceptance Criteria
- [ ] `Alduin::Request` always returns `Alduin::Reply { Wallet }` with balance, pending_count, deposit_wallet
- [ ] When page ≥ 1, response also includes paginated history
- [ ] Transaction history paginated (20/page, newest first)
- [ ] Total pages calculated correctly
- [ ] Edge cases handled: no transactions, out-of-range page
- [ ] Each `TransactionEntry` includes `action` (deposit/withdraw), `amount`, `wallet_address`, `status`




## Issue #5
# Phase 2: Implement Alduin withdraw handler

## Goal

Handle `Alduin::Withdraw` (PacketAction: `Remove`) packets — remove Alduin items from player inventory and create a pending withdrawal transaction. The amount is selected by the player via a UI slider (bounds sent from server config, clamped to inventory balance), not a separate amount input dialog.

## Rate Limiting

- **5-second cooldown** between withdrawal requests per character.
- Server tracks the timestamp of the last withdrawal request per character.
- If a new `Alduin::Withdraw` arrives within 5 seconds of the previous one → server **silently ignores** the request (no reply sent).

## Single Pending Per Type

- A character can only have **ONE pending withdrawal** at a time.
- Player must cancel their existing pending withdrawal before creating a new one.

## Flow

1. Player selects amount via slider → enters wallet address → clicks Withdraw
2. Server validates (in order):
   - Rate limit: last withdrawal request was > 5 seconds ago → else **silently ignore**
   - `amount` >= configured withdraw_min → else `Alduin::Reply { AmountBelowMin }`
   - `amount` <= configured withdraw_max → else `Alduin::Reply { AmountAboveMax }`
   - Player has >= `amount` Alduin items (ID 491) in inventory → else `Alduin::Reply { InsufficientFunds }`
   - `wallet_address` is non-empty → else `Alduin::Reply { InvalidWalletAddress }`
   - Character has **no existing pending withdrawal** → else `Alduin::Reply { AlreadyHasPending }`
3. Remove `amount` Alduin items from inventory immediately
4. Create pending transaction record in `character_transaction` (action: withdraw, status: pending, wallet_address, amount)
5. Send `Alduin::Reply { Notify: { transaction_id, status: pending, total_alduin } }` to client
6. Send Discord webhook notification to staff channel

## On Approval
- Transaction marked approved (items already removed — no further item changes needed)
- Send `Alduin::Reply { Notify: { transaction_id, status: approved, total_alduin } }` to client

## On Cancellation (staff or player self-service)
- Server refunds `amount` Alduin items (ID 491) to player **inventory only** (not bank)
- **Inventory overflow handling (`settled_amount`):**
  - If player inventory is full and cannot accept the full refund:
    - Refund is **capped at 2,000,000,000** (EO max inventory value).
    - The **actual refunded amount** is recorded as `settled_amount` in the `character_transaction` DB record.
    - If `settled_amount < amount` (shortfall occurred), a **Discord notification is sent to staff** alerting them of the shortfall, including character name, requested amount, settled_amount, and lost amount.
  - If inventory has space for the full refund: `settled_amount == amount`, no shortfall notification needed.
- Transaction marked cancelled
- Send `Alduin::Reply { Notify: { transaction_id, status: cancelled, total_alduin } }` to client

## Discord Webhook Format
```json
{
  "content": "💸 **Alduin Withdrawal Request**",
  "embeds": [{
    "title": "Player requests withdrawal of X Alduin",
    "fields": [
      {"name": "Character", "value": "<name> (ID: <id>)"},
      {"name": "Wallet", "value": "<wallet_address>"},
      {"name": "Amount", "value": "<amount>"},
      {"name": "Type", "value": "Withdraw"},
      {"name": "Transaction ID", "value": "<id>"}
    ],
    "color": 16744192
  }]
}
```

## Discord Shortfall Notification Format
```json
{
  "content": "⚠️ **Withdrawal Refund Shortfall**",
  "embeds": [{
    "title": "Inventory full — partial refund applied",
    "fields": [
      {"name": "Character", "value": "<name> (ID: <id>)"},
      {"name": "Requested Refund", "value": "<amount>"},
      {"name": "Settled Amount", "value": "<settled_amount>"},
      {"name": "Lost Amount", "value": "<lost_amount>"},
      {"name": "Transaction ID", "value": "<id>"}
    ],
    "color": 16711680
  }]
}
```

## Acceptance Criteria
- [ ] `Alduin::Remove` handler registered and receiving packets
- [ ] 5-second cooldown between withdrawal requests per character (silent ignore if within cooldown)
- [ ] Single pending withdrawal per character enforced (AlreadyHasPending reply)
- [ ] Amount validation against configured min/max range and inventory balance
- [ ] Items removed from inventory immediately on valid request
- [ ] Pending transaction record created in `character_transaction` with wallet_address and amount
- [ ] `Alduin::Reply` sent for validation failures (AmountBelowMin, AmountAboveMax, InsufficientFunds, InvalidWalletAddress, AlreadyHasPending)
- [ ] `Alduin::Reply { Notify }` sent to client on all state changes with updated `total_alduin`
- [ ] Discord webhook fires with: character id, name, wallet address, amount, type
- [ ] Approval path: marks approved (items already removed)
- [ ] Cancellation path (staff or player): refunds items to inventory only, marks cancelled
- [ ] `settled_amount` recorded in DB on cancellation (accounts for inventory overflow)
- [ ] Shortfall capped at 2,000,000,000 (EO max inventory) when inventory is full at refund time
- [ ] Discord shortfall notification sent to staff when `settled_amount < amount`



## Issue #4
# Phase 2: Implement Alduin deposit handler

## Goal

Handle `Alduin::Deposit` (PacketAction: `Add`) packets — create a pending deposit request. Items are only added to inventory when staff approves. The amount is selected by the player via a UI slider (bounds sent from server config), not a separate amount input dialog.

## Rate Limiting

- **5-second cooldown** between deposit requests per character.
- Server tracks the timestamp of the last deposit request per character.
- If a new `Alduin::Deposit` arrives within 5 seconds of the previous one → server **silently ignores** the request (no reply sent).

## Single Pending Per Type

- A character can only have **ONE pending deposit** at a time.
- Player must cancel their existing pending deposit before creating a new one.

## Flow

1. Player selects amount via slider → enters wallet address → clicks Deposit
2. Server validates (in order):
   - Rate limit: last deposit request was > 5 seconds ago → else **silently ignore**
   - `amount` >= configured deposit_min → else `Alduin::Reply { AmountBelowMin }`
   - `amount` <= configured deposit_max → else `Alduin::Reply { AmountAboveMax }`
   - `wallet_address` is non-empty → else `Alduin::Reply { InvalidWalletAddress }`
   - Character has **no existing pending deposit** → else `Alduin::Reply { AlreadyHasPending }`
3. Create pending transaction record in `character_transaction` (action: deposit, status: pending, wallet_address, amount)
4. Send `Alduin::Reply { Notify: { transaction_id, status: pending, total_alduin } }` to client
5. Send Discord webhook notification to staff channel

## On Approval
- Server adds `amount` Alduin items (ID 491) to player inventory
- If inventory is full, items go to bank
- Transaction marked approved
- Send `Alduin::Reply { Notify: { transaction_id, status: approved, total_alduin } }` to client

## On Cancellation
- Transaction marked cancelled (no item changes — nothing was ever added)
- Send `Alduin::Reply { Notify: { transaction_id, status: cancelled, total_alduin } }` to client

## Discord Webhook Format
```json
{
  "content": "🏦 **Alduin Deposit Request**",
  "embeds": [{
    "title": "Player requests deposit of X Alduin",
    "fields": [
      {"name": "Character", "value": "<name> (ID: <id>)"},
      {"name": "Wallet", "value": "<wallet_address>"},
      {"name": "Amount", "value": "<amount>"},
      {"name": "Type", "value": "Deposit"},
      {"name": "Transaction ID", "value": "<id>"}
    ],
    "color": 5814783
  }]
}
```

## Acceptance Criteria
- [ ] `Alduin::Add` handler registered and receiving packets
- [ ] 5-second cooldown between deposit requests per character (silent ignore if within cooldown)
- [ ] Single pending deposit per character enforced (AlreadyHasPending reply)
- [ ] Amount validation against configured min/max range
- [ ] Wallet address validation
- [ ] Pending transaction record created in `character_transaction` with wallet_address and amount
- [ ] `Alduin::Reply` sent for validation failures (AmountBelowMin, AmountAboveMax, InvalidWalletAddress, AlreadyHasPending)
- [ ] `Alduin::Reply { Notify }` sent to client with current `total_alduin`
- [ ] Discord webhook fires with: character id, name, wallet address, amount, type
- [ ] Approval path: credits items to inventory/bank
- [ ] Cancellation path: just marks cancelled (no item changes)



## Issue #3
# Phase 2: Add Alduin transaction database schema

## Goal

Add database table to track Alduin pending transactions. No separate wallet table is needed — balance is derived from counting item ID 491 in `character_inventory` only (bank excluded).

## Schema

```sql
CREATE TABLE IF NOT EXISTS character_transaction (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    character_id INTEGER NOT NULL,
    action TEXT NOT NULL CHECK(action IN ('deposit', 'withdraw')),
    amount INTEGER NOT NULL,
    settled_amount INTEGER NOT NULL DEFAULT 0,  -- actual amount settled (handles inventory cap at 2B on withdraw cancel refund)
    wallet_address TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending' CHECK(status IN ('pending', 'approved', 'cancelled')),
    created_at INTEGER NOT NULL,  -- UNIX timestamp
    resolved_at INTEGER,          -- UNIX timestamp (null while pending)
    resolved_by INTEGER,          -- admin character_id (null while pending)
    notified BOOLEAN NOT NULL DEFAULT 0,  -- whether Alduin::Reply { Notify } was sent to client
    FOREIGN KEY (character_id) REFERENCES characters(id)
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_character_transaction_character_id_status
    ON character_transaction (character_id, status);
CREATE INDEX IF NOT EXISTS idx_character_transaction_status_notified
    ON character_transaction (status, notified);
CREATE INDEX IF NOT EXISTS idx_character_transaction_character_id_action_status
    ON character_transaction (character_id, action, status);
```

## Steps

1. Create `CREATE TABLE IF NOT EXISTS` in the DB init code
2. Add transaction CRUD (create, list pending, resolve, list notified)
3. Add helper to get most recent wallet address for a character (for prepopulate)
4. Add helper function to count Alduin items (ID 491) in `character_inventory` only — this is the "balance"
5. Track `notified` flag — on login, send `Alduin::Reply { Notify }` for any resolved transactions where `notified = 0`

## Login Sync

On character login:
```sql
SELECT * FROM character_transaction
WHERE character_id = ? AND status IN ('approved', 'cancelled') AND notified = 0
```
For each row: send `Alduin::Reply { Notify }`, then set `notified = 1`.

## Notes
- Server uses SQLite or MySQL depending on `[database]` config
- Use parameterized queries to prevent injection
- Balance is always computed live from inventory count of item 491 only (bank excluded)
- `wallet_address` format validation is deferred (accept any non-empty string for now)

## Acceptance Criteria
- [ ] `character_transaction` table created on server startup with `wallet_address` and `settled_amount` columns
- [ ] Indexes created for `(character_id, status)`, `(status, notified)`, and `(character_id, action, status)`
- [ ] Transaction creation and listing functions
- [ ] Transaction approval/cancellation functions
- [ ] Helper to get most recent wallet address for a character
- [ ] Helper function to count item 491 in inventory only
- [ ] `notified` flag tracks unsent notifications
- [ ] Login sends pending notifications and marks them sent



## Issue #2
# Phase 1: Define Alduin protocol extension XML

## Goal

Create the protocol extension XML files in `alduin-eo/eo-protocol-extensions` defining the new `Alduin` packet family using the existing `PacketAction` enum and the reply-switch pattern.

## Required directory structure

```
eo-protocol-extensions/
  alduin/
    protocol.xml              ← enums, structs
    net/
      client/
        protocol.xml          ← C→S packets
      server/
        protocol.xml          ← S→C packets
```

## Definitions to create

### New Enum: `TransactionAction`
```xml
<enum name="TransactionAction" type="char">
    <value name="Deposit">0</value>
    <value name="Withdraw">1</value>
</enum>
```

### New Enum: `TransactionStatus`
```xml
<enum name="TransactionStatus" type="char">
    <value name="Pending">0</value>
    <value name="Approved">1</value>
    <value name="Cancelled">2</value>
</enum>
```

### New Enum: `AlduinReply`
```xml
<enum name="AlduinReply" type="char">
    <comment>Server reply types and validation error codes</comment>
    <value name="Wallet">1</value>
    <value name="Notify">2</value>
    <value name="AmountBelowMin">10</value>
    <value name="AmountAboveMax">11</value>
    <value name="InsufficientFunds">12</value>
    <value name="InvalidWalletAddress">13</value>
    <value name="TransactionNotFound">14</value>
    <value name="TransactionNotPending">15</value>
    <value name="NotYourTransaction">16</value>
    <value name="AlreadyHasPending">17</value>
</enum>
```

### New Struct: `TransactionEntry`
```xml
<struct name="TransactionEntry">
    <field name="id" type="int"/>
    <field name="timestamp" type="int"/>
    <field name="action" type="TransactionAction"/>
    <field name="amount" type="int"/>
    <field name="wallet_address" type="string"/>
    <field name="status" type="TransactionStatus"/>
</struct>
```

### Client → Server Packets (`net/client/protocol.xml`)

**`Alduin::Request`** (PacketAction: `Init`) — request wallet info (always) + history page
- `page` (char) — 0 = wallet info only, ≥1 = wallet + history for that page

**`Alduin::Deposit`** (PacketAction: `Add`) — request deposit of Alduin items
- `amount` (int) — selected via UI slider
- `wallet_address` (string) — player's external wallet address

**`Alduin::Withdraw`** (PacketAction: `Remove`) — withdraw Alduin from wallet
- `amount` (int) — selected via UI slider
- `wallet_address` (string) — player's external wallet address

**`Alduin::Cancel`** (PacketAction: `Spec`) — cancel own pending transaction
- `transaction_id` (int)

### Server → Client Packets (`net/server/protocol.xml`)

**`Alduin::Reply`** (PacketAction: `Reply`) — unified reply with switch
```xml
<packet family="Alduin" action="Reply">
    <field name="reply" type="AlduinReply"/>
    <switch field="reply">
        <case value="Wallet">
            <field name="balance" type="int"/>
            <field name="pending_count" type="char"/>
            <field name="deposit_wallet" type="string"/>
            <field name="deposit_min" type="int"/>
            <field name="deposit_max" type="int"/>
            <field name="withdraw_min" type="int"/>
            <field name="withdraw_max" type="int"/>
            <field name="page" type="char"/>
            <field name="total_pages" type="char"/>
            <array name="transactions" type="TransactionEntry"/>
        </case>
        <case value="Notify">
            <field name="transaction_id" type="int"/>
            <field name="status" type="TransactionStatus"/>
            <field name="total_alduin" type="int"/>
        </case>
    </switch>
</packet>
```

**Error handling:** Any `AlduinReply` value ≥ 10 is a validation error — no switch case needed, the client just shows the appropriate message.

| Request | Possible Errors |
|---------|----------------|
| `Alduin::Deposit` | AmountBelowMin, AmountAboveMax, InvalidWalletAddress, AlreadyHasPending |
| `Alduin::Withdraw` | AmountBelowMin, AmountAboveMax, InsufficientFunds, InvalidWalletAddress, AlreadyHasPending |
| `Alduin::Cancel` | TransactionNotFound, TransactionNotPending, NotYourTransaction |

**Notes:**
- `deposit_wallet` — server's deposit wallet address (from config)
- `deposit_min`, `deposit_max`, `withdraw_min`, `withdraw_max` — slider bounds sent from server config
- `TransactionEntry` includes `action` (deposit/withdraw), `wallet_address`, and `amount`
- `Wallet` case **always** includes balance, pending_count, config bounds. When `page` ≥ 1, also includes page, total_pages, and transactions.
- `Alduin::Cancel` uses PacketAction `Spec` (existing enum member)
- Validation errors are sent as `Alduin::Reply` with the error value — client displays the corresponding message
- **Rate limiting:** Server rejects `Deposit`/`Withdraw`/`Cancel` requests made within 5 seconds of the previous Alduin request — client should debounce UI actions accordingly
- **Single pending per type:** A player may have at most one pending deposit AND one pending withdraw simultaneously. Attempting to create a new deposit (or withdraw) while one of that type is already pending returns `AlreadyHasPending`

## Process

1. Create extension XML files with the definitions above
2. Use `eo-proto-merge validate` to check for conflicts against base protocol
3. Once validated, the extensions are ready for code generation in reoserv and eoweb

## Acceptance Criteria
- [ ] Extension XML files created in `eo-protocol-extensions/alduin/`
- [ ] `eo-proto-merge validate --config=extensions.xml` passes without errors
- [ ] All packets use existing `PacketAction` enum members — no new action enum
- [ ] `Alduin::Cancel` packet defined with `transaction_id` (int)
- [ ] Single `Alduin::Request` (page: char), single `Alduin::Reply` with switch (`Wallet`, `Notify`) + error values
- [ ] `AlduinReply` enum covers both success replies and validation errors — no separate error enum
- [ ] `AlduinReply` includes `AlreadyHasPending` (value 17) for single-pending-per-type enforcement
- [ ] `Wallet` reply includes balance, pending_count, deposit_wallet, min/max bounds for slider
- [ ] `TransactionEntry` includes `action` (TransactionAction), `wallet_address` field
- [ ] `TransactionAction` enum: Deposit, Withdraw
- [ ] Deposit and Withdraw packets include `amount` (from slider) and `wallet_address` (string)
- [ ] `Notify` case includes `total_alduin` field for client inventory sync
- [ ] 5-second rate limiting between Alduin requests enforced server-side



## Issue #1
# Alduin Wallet Project — Master Plan

## Alduin Wallet System

The Alduin Wallet allows players to manage a special currency item (Item ID 491 — "Alduin") through a dedicated UI, with staff-approved deposit/withdrawal transactions tracked via Discord. Each transaction records the user's external wallet address (e.g. Ethereum-style).

## Architecture

```
┌──────────┐    Protocol     ┌──────────┐   Discord Webhook   ┌──────────────┐
│  eoweb   │ ◄─────────────► │ reoserv  │ ──────────────────► │  Discord Bot │
│ (Client) │   Alduin Packets│ (Server) │                     │  (Approval)  │
└──────────┘                 └────┬─────┘                     └──────┬───────┘
                                  │                                  │
                            ┌─────┴────┐                             │
                            │   DB     │                             │
                            │ (Txns)   │◄────────────────────────────┘
                            └──────────┘           HTTP API
```

reoserv exposes an HTTP API for the Discord bot to approve/cancel transactions. The bot never touches the database directly — all DB operations go through reoserv.

## Phases

- **Phase 1:** Protocol extensions ([`eo-protocol-extensions`](https://github.com/alduin-eo/eo-protocol-extensions))
- **Phase 2:** Server handlers + HTTP API ([`reoserv`](https://github.com/alduin-eo/reoserv))
- **Phase 3:** Discord approval bot (new service)
- **Phase 4:** Client UI ([`eoweb`](https://github.com/alduin-eo/eoweb))

## Packet Design

### New packet family: `Alduin`

| Packet | Direction | Description |
|--------|-----------|-------------|
| `Alduin::Request` | C→S | Request wallet info (always) + optional history page |
| `Alduin::Deposit` | C→S | Request deposit of Alduin items (amount selected via UI slider) |
| `Alduin::Withdraw` | C→S | Withdraw Alduin from wallet (amount selected via UI slider) |
| `Alduin::Cancel` | C→S | Cancel a pending transaction (player self-service) |
| `Alduin::Reply` | S→C | Unified reply with switch on reply value |

### `TransactionAction` enum

| Value | Description |
|-------|-------------|
| `Deposit` | Player requesting to receive Alduin |
| `Withdraw` | Player requesting to remove Alduin |

### `Alduin::Reply` switch codes

| Reply Code | Data |
|------------|------|
| `Wallet` | balance, pending_count, deposit_wallet, deposit_min, deposit_max, withdraw_min, withdraw_max, page, total_pages, transactions (array) |
| `Notify` | transaction_id, status (TransactionStatus), total_alduin (int) |

Values ≥ 10 are validation errors sent directly as the reply (no switch case needed):

| Value | Description |
|-------|-------------|
| `AmountBelowMin` | Amount below configured minimum |
| `AmountAboveMax` | Amount above configured maximum |
| `InsufficientFunds` | Not enough Alduin in inventory (withdraw) |
| `InvalidWalletAddress` | Empty or malformed wallet address |
| `TransactionNotFound` | Transaction ID does not exist |
| `TransactionNotPending` | Transaction is already approved or cancelled |
| `NotYourTransaction` | Transaction belongs to another character |
| `AlreadyHasPending` | Player already has a pending transaction of this type — cancel it first |

**Note:** `Alduin::Request` **always** returns wallet balance. If `page` ≥ 1, the `Wallet` reply also includes paginated transaction history. The server treats any page < 1 as page 1. `deposit_wallet` is the server's deposit wallet address (from config). Min/max values for deposit/withdraw are sent from server config for UI slider bounds.

## Item Details

- **Item ID:** 491
- **Name:** Alduin
- Balance is derived from counting item 491 in `character_inventory` only (bank excluded)

## Design Decisions

- **Single pending per type:** A player can have at most one pending deposit AND one pending withdraw at any time. To create a new pending transaction of a given type, the player must first cancel their existing one.
- **Rate limiting:** A 5-second cooldown applies between any two Alduin requests from the same player. Requests arriving within the cooldown window are silently ignored — the client implements a local debounce timer and shows a spam warning if the player tries to send again too soon. No server-side error reply is sent for rate-limited requests.
- **`settled_amount` column:** Added to `character_transaction` to track the actual amount that was settled when the original requested amount cannot be fully applied. This handles the overflow case where inventory capacity caps at 2,147,483,647 — e.g. a withdraw cancellation that would refund more items than the inventory can hold records the actual refunded amount in `settled_amount`.
- **No pending transaction timeout:** Pending transactions never expire automatically — they remain pending until a staff member approves/cancels them or the player cancels them.
- **No wallet address format validation:** For now, wallet addresses are stored as free-form text with no format validation (empty check only).
- **Discord bot behavior:** On approve/cancel, the bot edits the original transaction embed to reflect the new status AND sends a separate confirmation message in the channel.
- **Staff notification on player self-cancel:** When a player cancels their own pending transaction, the Discord bot sends a notification to staff so they are aware the transaction was dropped.

## Transaction Flows

### Deposit (request to receive Alduin)

1. Player sets amount via slider (bounded by config min/max)
2. Player enters wallet address → server's deposit wallet address shown for reference → requests deposit
3. **Rate limit check:** If less than 5 seconds since last Alduin request → silently ignored (client debounce prevents this from happening)
4. **Single pending check:** If player already has a pending transaction of this type → `Alduin::Reply { AlreadyHasPending }`. Must cancel existing first.
5. Pending deposit transaction created
6. Staff approves (via Discord bot → HTTP API → reoserv) → items **added** to inventory. Bot edits original embed + sends confirmation.
7. Staff cancels → no item changes. Bot edits original embed + sends confirmation.
8. **Player cancels** → no item changes. Discord bot notifies staff that player self-cancelled.

### Withdrawal (request to remove Alduin)

1. Player sets amount via slider (bounded by config min/max and current inventory balance)
2. Player enters wallet address (prepopulated with most recent) → requests withdrawal
3. **Rate limit check:** If less than 5 seconds since last Alduin request → reject with `RateLimited`
4. **Single pending check:** If player already has a pending withdraw → reject with `PendingWithdrawExists`. Must cancel existing pending withdraw first.
5. Items **removed** from inventory immediately, pending withdraw transaction created
6. Staff approves (via Discord bot → HTTP API → reoserv) → items stay removed (approved). Bot edits original embed + sends confirmation.
7. Staff cancels → items **refunded to inventory** (not bank). If inventory is full (2B cap), only the fittable amount is refunded; the `settled_amount` column records what was actually returned. Bot edits original embed + sends confirmation.
8. **Player cancels** → items **refunded to inventory** (not bank), same overflow handling with `settled_amount`. Discord bot notifies staff that player self-cancelled.

## Login Sync

On character login, the server sends `Alduin::Reply { Notify }` for any resolved transactions where the client was not yet notified.

---

## Phase Issues

### Phase 1: Protocol
- [#2](https://github.com/alduin-eo/alduin/issues/2) — Define Alduin protocol extension XML

### Phase 2: Server (reoserv)
- [#3](https://github.com/alduin-eo/alduin/issues/3) — Database schema
- [#4](https://github.com/alduin-eo/alduin/issues/4) — Deposit handler
- [#5](https://github.com/alduin-eo/alduin/issues/5) — Withdraw handler
- [#6](https://github.com/alduin-eo/alduin/issues/6) — Wallet info & transaction history handlers
- [#7](https://github.com/alduin-eo/alduin/issues/7) — Config + HTTP API setup
- [#12](https://github.com/alduin-eo/alduin/issues/12) — Cancel pending transaction handler

### Phase 3: Discord Bot
- [#8](https://github.com/alduin-eo/alduin/issues/8) — Alduin transaction Discord bot (HTTP API client)

### Phase 4: Client (eoweb)
- [#9](https://github.com/alduin-eo/alduin/issues/9) — Alduin controller & packet handlers
- [#10](https://github.com/alduin-eo/alduin/issues/10) — Wallet UI dialog
- [#11](https://github.com/alduin-eo/alduin/issues/11) — Nav button in in-game HUD



