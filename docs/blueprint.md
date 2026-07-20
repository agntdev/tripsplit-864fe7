# TripSplit — Bot specification

**Archetype:** finance

**Voice:** professional and concise — write every user-facing message, button label, error, and empty state in this voice.

TripSplit is a Telegram bot that helps groups split trip expenses in chat. It tracks who paid what, calculates running balances, and suggests minimal settlement payments. Expenses are recorded with payer, amount, description, and split method. The bot supports equal or custom splits, handles participants joining/leaving mid-trip, and maintains privacy within the group. Trip data is visible only to participants, and the organizer has full management controls.

> This is the complete contract for the bot. Implement EVERY entry point, flow, feature, integration, and edge case below. The completeness review checks the bot against this document after each build pass.

## Primary audience

- friends
- families
- travel companions

## Success criteria

- Trip expenses are accurately tracked and settled with minimal manual calculations
- Participants can easily view balances and suggested settlements
- Expenses are immutable after confirmation with explicit correction workflows
- Trip data is private to participants only

## Entry points

Every feature must be reachable from the bot's command/button surface (button-first; only /start and /help are slash commands).

- **/start** (command, actor: user, command: /start) — Open the main menu
- **/newtrip** (command, actor: user, command: /newtrip) — Create a new trip with a name and currency
  - inputs: trip name, currency
  - outputs: trip creation confirmation, participant list
- **/jointrip** (command, actor: user, command: /jointrip) — Join an existing trip
  - outputs: join confirmation
- **/leavetrip** (command, actor: user, command: /leavetrip) — Leave the current trip
  - outputs: leave confirmation
- **/addexpense** (command, actor: user, command: /addexpense) — Add a new expense with payer, amount, description, and split method
  - inputs: payer, amount, description, split method, custom shares (if applicable)
  - outputs: expense confirmation summary, confirmation button
- **/balance** (command, actor: user, command: /balance) — View current balances and suggested settlements
  - outputs: balance summary, private detailed view option
- **/expenses** (command, actor: user, command: /expenses) — List all logged expenses with filters
  - inputs: filter type (all, mine, by date)
  - outputs: expense list with details
- **Confirm expense** (button, actor: user, callback: expense:confirm) — Confirm and save an expense after review
  - outputs: expense saved confirmation
- **Mark as paid** (button, actor: user, callback: settlement:mark_paid) — Mark a suggested or manual settlement as paid
  - inputs: settlement details
  - outputs: payment confirmation, balance update

## Flows

### Create trip
_Trigger:_ /newtrip

1. User enters /newtrip with trip name
2. Bot asks for currency (default is group locale)
3. User confirms trip creation
4. Bot creates trip and adds participants (auto-include group members or manual selection)

_Data touched:_ trip, participant

### Add expense
_Trigger:_ /addexpense

1. User enters /addexpense
2. Bot shows form with payer (default: command sender), amount, description, and split method
3. User selects split method (Equal or Custom shares)
4. For custom shares, user specifies per-person amounts or percentages
5. Bot validates sum equals total with rounding rules
6. User confirms expense details
7. Bot shows confirmation summary and confirmation button
8. User confirms expense
9. Bot records expense and updates balances

_Data touched:_ expense, balance ledger

### View balances
_Trigger:_ /balance

1. User enters /balance
2. Bot shows per-person net balances and minimized settlement suggestions
3. For long lists, bot offers private detailed view option
4. User requests private view if needed

_Data touched:_ balance ledger

### Mark payment as paid
_Trigger:_ settlement:mark_paid

1. User selects 'Mark as paid' for a settlement
2. Bot asks for confirmation
3. User confirms payment
4. Bot updates settlement record and balances

_Data touched:_ settlement record, balance ledger

### Join/leave trip
_Trigger:_ /jointrip or /leavetrip

1. User enters /jointrip or reacts to bot prompt
2. Bot adds user to trip participants
3. User enters /leavetrip
4. Bot removes user from trip participants
5. Bot handles past expenses: new participants not retroactively assigned unless explicitly added

_Data touched:_ participant

## Data entities

Durable data (must survive a restart) uses the toolkit's persistent store, never in-memory maps.

- **trip** _(retention: persistent)_ — A trip with title, currency, start/end times, organizer, participant list, and state
  - fields: title, currency, start_time, end_time, organizer, participants, state
- **participant** _(retention: persistent)_ — A participant in a trip with Telegram user id, display name, join/leave timestamps, and optional share overrides
  - fields: telegram_user_id, display_name, join_time, leave_time, share_overrides
- **expense** _(retention: persistent)_ — An expense with payer, amount, description, split method, participants involved, and recorded-by
  - fields: id, payer, amount, currency, description, timestamp, split_method, participants_involved, recorded_by
- **balance_ledger** _(retention: persistent)_ — Per-trip running balances (who owes whom) computed from expenses
  - fields: trip_id, participant_id, balance
- **settlement_record** _(retention: persistent)_ — Payments marked as paid with proof/comment and confirmation status
  - fields: expense_id, payer, payee, amount, status, proof, timestamp

## Integrations

- **Telegram** (required) — Bot API messaging and group chat interactions
Call external APIs against their real contract (correct endpoints, ids, params); credentials from env. Do not fake responses.

## Owner controls

- Create and close trips
- Add/remove participants
- Edit trip details
- Export trip summary
- Approve expense corrections

## Notifications

- Expense confirmation requests
- Balance updates
- Settlement confirmation requests

## Permissions & privacy

- Trip data visible only to participants
- Private balance details available on request
- Organizer has full management controls
- Participants can only view trip data they're part of

## Edge cases

- Rounding per-person shares to currency units with residual allocation
- Immutable expense records with explicit correction workflows
- Participants joining/leaving mid-trip with retroactive inclusion/exclusion
- Minimal settlement algorithm for fewest transactions

## Required tests

- Verify expense recording with confirmation flow
- Test balance calculations with various splits
- Validate settlement minimization algorithm
- Ensure privacy boundaries between trips
- Test participant join/leave behavior with past expenses

## Assumptions

- Default currency inferred from group locale
- Expenses are immutable after confirmation
- New participants not retroactively assigned to past expenses
- Rounding rules ensure ledger balance
- Minimal settlement algorithm matches largest creditors/debtors
