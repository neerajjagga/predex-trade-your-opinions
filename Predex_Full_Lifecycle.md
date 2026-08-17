# Predex — Full Platform Lifecycle

> **Trade Your Opinions.**
>
> Predex is a binary prediction-market exchange where users trade **YES/NO contracts** through a centralized order book and matching engine.

---

## 1. Goal of This Document

This document explains the complete lifecycle of data in Predex:

```text
User Registration
        ↓
Financial Account
        ↓
Deposit
        ↓
Market Creation
        ↓
Outcomes (YES / NO)
        ↓
Market Opens
        ↓
Order Placement
        ↓
Order Book
        ↓
Matching Engine
        ↓
Trade Execution
        ↓
Balance + Position Updates
        ↓
Market Closes
        ↓
Market Resolution
        ↓
Settlement
        ↓
Payout
```

The goal is to understand **why each database table exists, when it is created/updated, and how the tables work together**.

---

# 2. Core Database Tables

Predex currently has these core tables:

| # | Table | Main Responsibility |
|---|---|---|
| 1 | `users` | Stores who is using Predex |
| 2 | `accounts` | Stores the user's current money state |
| 3 | `ledger_entries` | Stores the history/reason for money movements |
| 4 | `markets` | Stores the prediction question |
| 5 | `outcomes` | Stores the possible answers: YES / NO |
| 6 | `orders` | Stores what users want to trade |
| 7 | `trades` | Stores what was actually traded |
| 8 | `positions` | Stores what contracts users currently own |
| 9 | `settlements` | Stores what users receive after market resolution |

> **Fees are intentionally excluded for now.**

---

# 3. The Most Important Mental Model

Before understanding the lifecycle, remember what each table represents:

```text
users
"What user is this?"

accounts
"How much money does the user have right now?"

ledger_entries
"Why did the user's money change?"

markets
"What prediction are we trading?"

outcomes
"What are the possible answers?"

orders
"What does the user WANT to do?"

trades
"What ACTUALLY happened?"

positions
"What contracts does the user OWN?"

settlements
"What did those contracts PAY after resolution?"
```

---

# 4. Market vs Outcome

This is one of the most important concepts in Predex.

## Market

A market is the **question**.

Example:

```text
Will India win the match?
```

## Outcome

An outcome is a **possible answer to that question**.

Because Predex is binary:

```text
YES
NO
```

Therefore:

```text
MARKET
"Will India win the match?"
        │
        ├── YES
        │
        └── NO
```

Database representation:

```text
markets

M001
"Will India win the match?"
```

and:

```text
outcomes

O001 | M001 | YES
O002 | M001 | NO
```

So:

```text
M001
 │
 ├── O001 → YES
 └── O002 → NO
```

An order can therefore reference `O001` instead of repeatedly storing `"YES"`.

---

# 5. Complete Lifecycle

We will follow one example throughout the document.

### Market

```text
Will India win the match?
```

### Outcomes

```text
YES
NO
```

### User

```text
Neeraj
```

---

# 6. Step 1 — User Registration

Neeraj registers:

```text
username = neeraj
email    = neeraj@example.com
password = ********
```

Predex creates a row in:

```text
users
```

Example:

```text
U001 | neeraj | neeraj@example.com | ACTIVE
```

### Why do we need `users`?

It answers:

> **Who is using the platform?**

Other tables can reference:

```text
user_id = U001
```

For example:

```text
orders.user_id
accounts.user_id
positions.user_id
trades.buyer_id
trades.seller_id
```

---

# 7. Step 2 — Financial Account Is Created

After registration, Predex creates a trading account for the user.

```text
accounts

A001 | U001 | INR | ₹0 available | ₹0 locked
```

### Why do we need `accounts`?

`users` represents identity.

`accounts` represents financial state.

```text
USER
 │
 └── ACCOUNT
       │
       ├── Available balance
       └── Locked balance
```

We don't want to mix user identity and trading balance into one table.

---

# 8. Step 3 — User Deposits Money

Neeraj deposits:

```text
₹10,000
```

The current account state becomes:

```text
Available = ₹10,000
Locked    = ₹0
```

This is stored in:

```text
accounts
```

But Predex also records the financial event in:

```text
ledger_entries
```

Example:

```text
DEPOSIT | CREDIT | ₹10,000
```

### Why both?

`accounts` answers:

> How much money do I have now?

`ledger_entries` answers:

> How did I get here?

For example:

```text
LEDGER

+ ₹10,000  DEPOSIT
- ₹1,260   TRADE
+ ₹3,000   SETTLEMENT
```

The ledger provides an audit trail.

---

# 9. Step 4 — Admin Creates a Market

The admin creates:

```text
Will India win the match?
```

A row is created in:

```text
markets
```

Example:

```text
M001
title = "Will India win the match?"
status = DRAFT
```

The market contains information such as:

```text
opens_at
closes_at
resolution_at
status
resolution_id
```

### Why do we need `markets`?

It represents the actual prediction event being traded.

Think:

```text
MARKET = QUESTION
```

---

# 10. Step 5 — Outcomes Are Created

Because the market is binary, Predex creates:

```text
O001 | M001 | YES
O002 | M001 | NO
```

in:

```text
outcomes
```

The relationship is:

```text
MARKET
   │
   ├── YES
   └── NO
```

### Why do we need `outcomes`?

The market tells us:

> What is the question?

The outcomes tell us:

> What are the possible answers?

This also gives other tables a clean reference:

```text
orders.outcome_id
trades.outcome_id
positions.outcome_id
settlements.outcome_id
```

---

# 11. Step 6 — Market Opens

At the configured opening time:

```text
markets.status = OPEN
```

Users can now submit orders.

Example:

```text
opens_at  = 10:00 AM
closes_at = 07:00 PM
```

Before 10 AM:

```text
Trading not allowed
```

After 10 AM:

```text
Trading allowed
```

---

# 12. Step 7 — User Places an Order

Neeraj wants to buy:

```text
100 YES
at ₹63
```

An order is created:

```text
orders

OR001
user_id      = U001
market_id    = M001
outcome_id   = O001
side         = BUY
order_type   = LIMIT
price        = ₹63
quantity     = 100
filled       = 0
status       = OPEN
```

### Why do we need `orders`?

An order represents:

> **What the user wants to do.**

It is not yet a trade.

```text
ORDER
"I want to buy 100 YES at ₹63."
```

Nothing has necessarily happened yet.

---

# 13. Step 8 — Order Enters the Order Book

The order is sent to the order book.

Example:

```text
ASK / SELL SIDE

₹65 → 50 YES
₹64 → 30 YES
₹63 → 20 YES


BID / BUY SIDE

₹62 → 40 YES
₹61 → 80 YES
₹60 → 100 YES
```

Neeraj's:

```text
BUY 100 YES @ ₹63
```

is placed on the bid side.

The order book is the live collection of active orders.

> The order book is primarily a trading-engine data structure; the `orders` table stores the persistent order records.

---

# 14. Step 9 — Matching Engine Checks the Order

The matching engine asks:

> Is there a compatible opposite order?

Neeraj:

```text
BUY YES @ ₹63
```

Seller:

```text
SELL YES @ ₹63
```

They are compatible.

So:

```text
BUY 100
     ↕
SELL 20
```

can produce:

```text
TRADE = 20
```

The buyer still wants:

```text
100 - 20 = 80
```

So his order becomes:

```text
quantity        = 100
filled_quantity = 20
remaining       = 80
status          = PARTIALLY_FILLED
```

---

# 15. Step 10 — Trade Is Created

Once matching happens, something has actually occurred.

Predex creates a row in:

```text
trades
```

Example:

```text
T001

buy_order_id  = OR001
sell_order_id = OR002
buyer_id      = U001
seller_id     = U002

outcome_id    = O001
price         = ₹63
quantity      = 20
```

### Why do we need `trades`?

Because:

```text
ORDER
"What did the user request?"
```

while:

```text
TRADE
"What actually executed?"
```

One order can create multiple trades.

Example:

```text
BUY 100 YES

        ↓

Trade 1 → 20
Trade 2 → 30
Trade 3 → 50
```

Therefore:

```text
1 ORDER
   ↓
MANY TRADES
```

---

# 16. Step 11 — Money Is Updated

The trade means money has changed hands.

Trade:

```text
20 YES @ ₹63
```

Value:

```text
20 × ₹63
= ₹1,260
```

The appropriate account balances are updated.

The financial movement is also recorded in:

```text
ledger_entries
```

For example:

```text
TRADE_DEBIT
DEBIT
₹1,260
```

and the seller receives the corresponding amount according to the platform's accounting model.

### Why is the ledger important?

Because the account stores the current state:

```text
Available = ₹8,740
```

while the ledger stores the event:

```text
₹1,260 used in trade T001
```

---

# 17. Step 12 — Position Is Updated

Neeraj bought:

```text
20 YES
```

Therefore his ownership changes.

Before:

```text
YES = 0
```

After:

```text
YES = 20
```

A row exists in:

```text
positions
```

Example:

```text
U001 | M001 | O001 | 20
```

### Why do we need `positions`?

Because neither orders nor trades directly answer:

> What does Neeraj own right now?

`positions` answers exactly that.

```text
Neeraj
  ↓
Market M001
  ↓
YES
  ↓
20 contracts
```

---

# 18. Step 13 — More Trading Happens

Suppose Neeraj later buys another:

```text
30 YES
```

The process repeats:

```text
ORDER
  ↓
ORDER BOOK
  ↓
MATCHING ENGINE
  ↓
TRADE
  ↓
ACCOUNT / LEDGER
  ↓
POSITION
```

His position becomes:

```text
20 + 30 = 50 YES
```

So:

```text
positions

U001 | M001 | O001 | 50
```

We normally don't create another position row for the same user/market/outcome.

Instead:

```text
quantity = 50
```

because of:

```text
UNIQUE(user_id, market_id, outcome_id)
```

---

# 19. Step 14 — User Sells

Suppose Neeraj owns:

```text
50 YES
```

and wants to sell:

```text
20 YES
```

He creates:

```text
SELL 20 YES
```

in:

```text
orders
```

If it matches:

```text
trade
```

is created.

Then his position becomes:

```text
50 - 20 = 30 YES
```

So:

```text
SELL ORDER
     ↓
TRADE
     ↓
POSITION DECREASES
```

---

# 20. Step 15 — Market Closes

At:

```text
closes_at = 7:00 PM
```

Predex changes:

```text
markets.status
```

from:

```text
OPEN
```

to:

```text
CLOSED
```

Now new orders should no longer be accepted.

The handling of remaining open orders is a separate exchange-rule decision.

---

# 21. Step 16 — Market Is Resolved

Suppose India wins.

Predex determines:

```text
YES = WINNER
NO  = LOSER
```

The market stores:

```text
resolution_id = O001
status        = RESOLVED
```

Remember:

```text
O001 → YES
```

So:

```text
M001
 │
 ├── O001 → YES → WINNER
 └── O002 → NO  → LOSER
```

### Why `resolution_id`?

It answers:

> Which outcome won this market?

Instead of storing:

```text
resolution = "YES"
```

we store:

```text
resolution_id = O001
```

which points to:

```text
O001 → YES
```

---

# 22. Step 17 — Settlement

Suppose Neeraj currently owns:

```text
30 YES
```

Predex uses the fixed-payout model:

```text
Winning YES contract = ₹100
Losing YES contract  = ₹0
```

Therefore:

```text
30 × ₹100
= ₹3,000
```

Predex creates a:

```text
settlements
```

record:

```text
user_id             = U001
market_id           = M001
outcome_id          = O001
position_id         = P001

quantity            = 30
payout_per_contract = ₹100
payout_amount       = ₹3,000

status              = COMPLETED
```

---

# 23. Why Do We Need `settlements`?

The position says:

```text
Neeraj owns 30 YES.
```

The settlement says:

```text
Those 30 YES contracts were settled
and generated ₹3,000.
```

So:

```text
POSITION
"What did I own?"

        ↓

SETTLEMENT
"What did that ownership pay?"
```

---

# 24. Step 18 — Payout Goes to Account

The ₹3,000 payout is recorded financially.

A ledger entry is created:

```text
SETTLEMENT
CREDIT
₹3,000
```

Then the account is updated:

```text
Available balance += ₹3,000
```

So:

```text
SETTLEMENT
     ↓
LEDGER
     ↓
ACCOUNT
```

---

# 25. Complete Lifecycle

```text
┌──────────────────────┐
│   USER REGISTERS     │
└──────────┬───────────┘
           ↓
        USERS
           ↓
      ACCOUNTS
           ↓
       DEPOSIT
           ↓
   LEDGER_ENTRIES
           │
           │
           │
┌──────────▼───────────┐
│ ADMIN CREATES MARKET │
└──────────┬───────────┘
           ↓
        MARKETS
           ↓
      OUTCOMES
      /       \
    YES       NO
      \       /
       \     /
        MARKET OPENS
             ↓
       USER PLACES ORDER
             ↓
           ORDERS
             ↓
        ORDER BOOK
             ↓
     MATCHING ENGINE
             ↓
           TRADES
          /     \
         /       \
        ↓         ↓
    LEDGER     POSITIONS
        ↓         ↓
     ACCOUNT    OWNERSHIP
                   │
                   │
             MARKET CLOSES
                   ↓
              RESOLUTION
                   ↓
            YES / NO WINNER
                   ↓
             SETTLEMENTS
                   ↓
                LEDGER
                   ↓
                ACCOUNT
```

---

# 26. Role of Every Table

| Table | Simple Question It Answers |
|---|---|
| `users` | **Who is this?** |
| `accounts` | **How much money does this user have right now?** |
| `ledger_entries` | **Why did their money change?** |
| `markets` | **What prediction are we trading?** |
| `outcomes` | **What are the possible answers?** |
| `orders` | **What does the user want to do?** |
| `trades` | **What actually happened?** |
| `positions` | **What contracts does the user currently own?** |
| `settlements` | **What did those contracts pay after resolution?** |

---

# 27. The Most Important Relationships

## User → Account

```text
USER
 ↓
ACCOUNT
```

One user has a financial account.

---

## Market → Outcomes

```text
MARKET
 ↓
YES
NO
```

One binary market has two possible outcomes.

---

## User → Orders

```text
USER
 ↓
ORDERS
```

A user can place many orders.

---

## Order → Trades

```text
ORDER
 ↓
TRADES
```

One order can be partially filled through multiple trades.

---

## Trade → Position

```text
TRADE
 ↓
POSITION
```

Executed trades change ownership.

---

## Account → Ledger

```text
ACCOUNT
 ↓
LEDGER
```

Account = current financial state.

Ledger = financial history.

---

## Position → Settlement

```text
POSITION
 ↓
SETTLEMENT
```

The user's contracts are settled after market resolution.

---

# 28. The Most Important Distinctions

These are worth memorizing:

### Market vs Outcome

```text
Market  = Question
Outcome = Possible answer
```

Example:

```text
Market: "Will India win?"

Outcome:
YES
NO
```

---

### Order vs Trade

```text
Order = Intention
Trade = Execution
```

Example:

```text
Order:
BUY 100 YES @ ₹63

Trade:
20 YES @ ₹63
```

---

### Account vs Ledger

```text
Account = Current state
Ledger  = Historical events
```

Example:

```text
Account:
₹8,740

Ledger:
+₹10,000 deposit
-₹1,260 trade
```

---

### Trade vs Position

```text
Trade    = What happened
Position = What you own now
```

Example:

```text
Trade:
Bought 20 YES

Position:
Own 20 YES
```

---

### Position vs Settlement

```text
Position   = What you own before resolution
Settlement = What that ownership pays after resolution
```

Example:

```text
Position:
30 YES

Market:
YES wins

Settlement:
₹3,000
```

---

# 29. Final Mental Model

If you remember nothing else, remember this:

```text
                    USER
                     │
                     ↓
                  ACCOUNT
                     │
                   MONEY
                     │
                     ↓
                   MARKET
                     │
                QUESTION
                     │
                     ↓
                 OUTCOMES
                 /       \
               YES       NO
                 \       /
                  \     /
                   ↓   ↓
                   ORDER
                     │
                "I WANT TO"
                     │
                     ↓
               ORDER BOOK
                     │
                     ↓
             MATCHING ENGINE
                     │
                     ↓
                   TRADE
                     │
                "IT HAPPENED"
                     │
             ┌───────┴───────┐
             ↓               ↓
          LEDGER          POSITION
          MONEY           OWNERSHIP
             │               │
             ↓               ↓
          ACCOUNT         CONTRACTS
                              │
                        MARKET RESOLVES
                              │
                              ↓
                         SETTLEMENT
                              │
                         "I RECEIVED"
                              │
                              ↓
                           LEDGER
                              │
                              ↓
                           ACCOUNT
```

---

# 30. Where the Matching Engine Fits

Notice that the matching engine is **not responsible for the entire Predex system**.

Its responsibility is much narrower:

```text
                  ORDERS
                     ↓
                ORDER BOOK
                     ↓
            ┌─────────────────┐
            │ MATCHING ENGINE │
            └────────┬────────┘
                     ↓
                  TRADES
```

It takes valid orders, determines which orders can match according to **price-time priority**, and produces executions.

Everything before and after it has a different responsibility:

```text
Before matching:
User → Market → Outcome → Order

Matching:
Order → Order Book → Matching Engine → Trade

After matching:
Trade → Ledger + Position

After market resolution:
Position → Settlement → Ledger → Account
```

**This is the foundation we should have in place before designing the matching engine.**
