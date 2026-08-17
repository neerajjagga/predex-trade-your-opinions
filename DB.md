# Predex — Database Schema & Data Flow

# Tables

We currently need **9 core tables**:

| # | Table | Purpose |
|---:|---|---|
| 1             | `users`          | Stores user identity and account information |
| 2             | `markets`        | Stores prediction markets/events             |
| 3             | `outcomes`       | Stores YES/NO outcomes for each market       |
| 4             | `orders`         | Stores users' requests to buy/sell contracts |
| 5             | `trades`         | Stores successfully matched executions       |
| 6             | `accounts`       | Stores user's current cash balance           |
| 7             | `ledger_entries` | Stores financial movement history            |
| 8             | `positions`      | Stores contracts currently owned by users    |
| 9             | `settlements`    | Stores final payouts after market resolution |

**Fees are intentionally excluded for now.**

---

# 3. `users`

Stores the identity of people using Predex.

```
CREATE TABLE users (
```

    id UUID PRIMARY KEY,



    username VARCHAR(50) NOT NULL UNIQUE,

    email VARCHAR(255) NOT NULL UNIQUE,

    password_hash TEXT NOT NULL,



    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',



    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP

);

### Columns

| Column | Why we need it |                                                                                    |
|---|---| ---------------------------------------------------------------------------------- |
| `id`                 | Unique identifier for the user and the primary key referenced by other tables.     |
| `username`           | Human-readable unique identity for the user.                                       |
| `email`              | User's login/contact identifier; unique so two accounts cannot use the same email. |
| `password_hash`      | Stores the hashed password rather than the actual password.                        |
| `status`             | Allows us to activate, suspend, or deactivate users.                               |
| `created_at`         | Records when the user account was created.                                         |
| `updated_at`         | Records when the user's information was last updated.                              |

Example:

```
U001
```

neeraj

neeraj\@example.com

ACTIVE

---

# 4. `markets`

A market represents one prediction question.

Example:

> "Will India win the 2026 World Cup?"

```
CREATE TABLE markets (
```

    id UUID PRIMARY KEY,



    title VARCHAR(255) NOT NULL,

    description TEXT,



    status VARCHAR(30) NOT NULL,



    opens_at TIMESTAMP NOT NULL,

    closes_at TIMESTAMP NOT NULL,

    resolution_at TIMESTAMP,



    resolution_id UUID,



    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP

);

### Columns

| Column | Why we need it |                                                                        |
|---|---| ---------------------------------------------------------------------- |
| `id`                 | Unique identifier for the market.                                      |
| `title`              | Short question displayed to users.                                     |
| `description`        | Provides detailed information about the prediction question and rules. |
| `status`             | Tracks the market lifecycle such as `OPEN`, `CLOSED`, `RESOLVED`, etc. |
| `opens_at`           | Defines when users can start trading the market.                       |
| `closes_at`          | Defines when new trading must stop.                                    |
| `resolution_at`      | Records when the market is expected/allowed to be resolved.            |
| `resolution_id`      | Identifies the winning outcome after the market is resolved.           |
| `created_at`         | Records when the market was created.                                   |
| `updated_at`         | Records when market information was last modified.                     |

### Example

```
M001
```



Title:

"Will India win the match?"



Status:

OPEN



Opens:

10:00 AM



Closes:

7:00 PM



Resolution:

After match result

After resolution:

```
resolution_id = O001
```

meaning:

```
O001 = YES
```

---

# 5. `outcomes`

Every binary market has two possible outcomes:

```
YES
```

NO

```
CREATE TABLE outcomes (
```

    id UUID PRIMARY KEY,



    market_id UUID NOT NULL,



    name VARCHAR(20) NOT NULL,



    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',



    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,



FOREIGN KEY (market_id) REFERENCES markets(id)

);

### Columns

| Column | Why we need it |                                                                                 |
|---|---| ------------------------------------------------------------------------------- |
| `id`                 | Unique identifier for the outcome referenced by orders, trades, positions, etc. |
| `market_id`          | Identifies which market this outcome belongs to.                                |
| `name`               | Stores the outcome name, which for Predex is `YES` or `NO`.                     |
| `status`             | Allows us to mark an outcome as active, winner, or loser during resolution.     |
| `created_at`         | Records when the outcome was created.                                           |

Example:

```
O001 | M001 | YES
```

O002 | M001 | NO

### Why not simply store `YES`/`NO` in orders?

We could, but `outcome_id` gives us a proper relationship:

```
ORDER
```

  ↓

OUTCOME

  ↓

MARKET

This becomes useful when connecting orders, trades, positions, and settlements.

---

# 6. `orders`

This is one of the most important tables.

An order means:

> **"I want to buy/sell these contracts at this price."**

```
CREATE TABLE orders (
```

    id UUID PRIMARY KEY,



    user_id UUID NOT NULL,

    market_id UUID NOT NULL,

    outcome_id UUID NOT NULL,



    side VARCHAR(10) NOT NULL,

    order_type VARCHAR(20) NOT NULL,

    time_in_force VARCHAR(20) NOT NULL,



    price DECIMAL(20, 8),

    quantity DECIMAL(20, 8) NOT NULL,

    filled_quantity DECIMAL(20, 8) NOT NULL DEFAULT 0,



    status VARCHAR(30) NOT NULL,



    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,



FOREIGN KEY (user_id) REFERENCES users(id),

FOREIGN KEY (market_id) REFERENCES markets(id),

FOREIGN KEY (outcome_id) REFERENCES outcomes(id)

);

### Columns

| Column | Why we need it |                                                                                             |
|---|---| ------------------------------------------------------------------------------------------- |
| `id`                 | Unique identifier for the order.                                                            |
| `user_id`            | Identifies who placed the order.                                                            |
| `market_id`          | Identifies which prediction market the order belongs to.                                    |
| `outcome_id`         | Identifies whether the order is trading YES or NO.                                          |
| `side`               | Tells whether the user wants to `BUY` or `SELL`.                                            |
| `order_type`         | Defines execution behavior such as `LIMIT` or `MARKET`.                                     |
| `time_in_force`      | Defines how long/how the order should remain active, e.g. `GTC`, `IOC`, `FOK`, `POST_ONLY`. |
| `price`              | Specifies the maximum buy or minimum sell price for limit orders.                           |
| `quantity`           | Specifies how many contracts the user wants to trade.                                       |
| `filled_quantity`    | Records how many contracts have actually been executed.                                     |
| `status`             | Tracks whether the order is `OPEN`, `PARTIALLY_FILLED`, `FILLED`, `CANCELLED`, etc.         |
| `created_at`         | Records when the order entered the system and helps with time priority.                     |
| `updated_at`         | Records the latest modification to the order.                                               |

---

## Example

Neeraj wants:

```
BUY
```

100 YES

at ₹63

Order:

```
id              = O001
```

user_id         = U001

market_id       = M001

outcome_id      = O001



side            = BUY

order_type      = LIMIT

time_in_force   = GTC



price           = ₹63

quantity        = 100

filled_quantity = 0



status          = OPEN

If 40 contracts get matched:

```
quantity        = 100
```

filled_quantity = 40

Remaining:

```
60
```

---

# 7. `trades`

An order represents an **intention**.

A trade represents an **actual execution**.

```
CREATE TABLE trades (
```

    id UUID PRIMARY KEY,



    market_id UUID NOT NULL,

    outcome_id UUID NOT NULL,



    buy_order_id UUID NOT NULL,

    sell_order_id UUID NOT NULL,



    buyer_id UUID NOT NULL,

    seller_id UUID NOT NULL,



    price DECIMAL(20, 8) NOT NULL,

    quantity DECIMAL(20, 8) NOT NULL,



    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,



FOREIGN KEY (market_id) REFERENCES markets(id),

FOREIGN KEY (outcome_id) REFERENCES outcomes(id),

FOREIGN KEY (buy_order_id) REFERENCES orders(id),

FOREIGN KEY (sell_order_id) REFERENCES orders(id),

FOREIGN KEY (buyer_id) REFERENCES users(id),

FOREIGN KEY (seller_id) REFERENCES users(id)

);

### Columns

| Column | Why we need it |                                                                                  |
|---|---| -------------------------------------------------------------------------------- |
| `id`                 | Unique identifier for the execution.                                             |
| `market_id`          | Identifies the market where the execution happened.                              |
| `outcome_id`         | Identifies whether YES or NO was traded.                                         |
| `buy_order_id`       | Identifies the order submitted by the buyer.                                     |
| `sell_order_id`      | Identifies the order submitted by the seller.                                    |
| `buyer_id`           | Identifies the user who bought the contracts.                                    |
| `seller_id`          | Identifies the user who sold the contracts.                                      |
| `price`              | Records the actual execution price.                                              |
| `quantity`           | Records how many contracts were actually executed.                               |
| `created_at`         | Records the exact time of execution and supports trade history/order sequencing. |

---

## Important relationship

One order can produce multiple trades.

Example:

```
BUY 100 YES @ ₹63
```

matches:

```
SELL 30 @ ₹60
```

SELL 20 @ ₹61

SELL 50 @ ₹62

Result:

```
Trade 1 → 30
```

Trade 2 → 20

Trade 3 → 50

Therefore:

```
1 ORDER
```

   ↓

MANY TRADES

---

# 8. `accounts`

Stores the user's current cash state.

```
CREATE TABLE accounts (
```

    id UUID PRIMARY KEY,



    user_id UUID NOT NULL UNIQUE,



    currency VARCHAR(10) NOT NULL DEFAULT 'INR',



    available_balance DECIMAL(20, 8) NOT NULL DEFAULT 0,

    locked_balance DECIMAL(20, 8) NOT NULL DEFAULT 0,



    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,



FOREIGN KEY (user_id) REFERENCES users(id)

);

### Columns

| Column | Why we need it |                                                                        |
|---|---| ---------------------------------------------------------------------- |
| `id`                 | Unique identifier for the financial account.                           |
| `user_id`            | Identifies the user who owns the account.                              |
| `currency`           | Defines the currency held by the account, initially INR.               |
| `available_balance`  | Stores money the user can currently use.                               |
| `locked_balance`     | Stores money reserved for active orders and unavailable for other use. |
| `created_at`         | Records when the financial account was created.                        |
| `updated_at`         | Records when the account balance/state was last updated.               |

---

## Why available and locked?

Suppose:

```
Available = ₹10,000
```

Locked    = ₹0

User places:

```
BUY 100 YES @ ₹60
```

Required:

```
100 × ₹60 = ₹6,000
```

We reserve it:

```
Available = ₹4,000
```

Locked    = ₹6,000

This prevents the user from spending the same ₹6,000 twice.

---

# 9. `ledger_entries`

The account tells us the **current state**.

The ledger tells us the **history**.

```
CREATE TABLE ledger_entries (
```

    id UUID PRIMARY KEY,



    account_id UUID NOT NULL,



    entry_type VARCHAR(30) NOT NULL,

    direction VARCHAR(10) NOT NULL,

    amount DECIMAL(20, 8) NOT NULL,



    reference_type VARCHAR(30),

    reference_id UUID,



    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,



FOREIGN KEY (account_id) REFERENCES accounts(id)

);

### Columns

| Column | Why we need it |                                                                                                                                |
|---|---| ------------------------------------------------------------------------------------------------------------------------------ |
| `id`                 | Unique identifier for the financial event.                                                                                     |
| `account_id`         | Identifies which account was affected.                                                                                         |
| `entry_type`         | Describes why the financial movement occurred, such as `DEPOSIT`, `WITHDRAWAL`, `ORDER_LOCK`, `ORDER_UNLOCK`, or `SETTLEMENT`. |
| `direction`          | Indicates whether the entry is a `CREDIT` or `DEBIT`.                                                                          |
| `amount`             | Records the amount involved in the financial movement.                                                                         |
| `reference_type`     | Identifies what caused the entry, such as `ORDER`, `TRADE`, `SETTLEMENT`, or `DEPOSIT`.                                        |
| `reference_id`       | Identifies the exact order/trade/settlement responsible for the entry.                                                         |
| `created_at`         | Records when the financial event occurred.                                                                                     |

---

## Example

Deposit:

```
DEPOSIT
```

CREDIT

₹10,000

Order lock:

```
ORDER_LOCK
```

₹6,000

Order cancellation:

```
ORDER_UNLOCK
```

₹6,000

The ledger should be treated as **immutable**.

Don't modify historical entries.

---

# 10. `positions`

A position tells us:

> **How many contracts does a user currently own?**

```
CREATE TABLE positions (
```

    id UUID PRIMARY KEY,



    user_id UUID NOT NULL,

    market_id UUID NOT NULL,

    outcome_id UUID NOT NULL,



    quantity DECIMAL(20, 8) NOT NULL DEFAULT 0,



    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,



FOREIGN KEY (user_id) REFERENCES users(id),

FOREIGN KEY (market_id) REFERENCES markets(id),

FOREIGN KEY (outcome_id) REFERENCES outcomes(id),



UNIQUE (user_id, market_id, outcome_id)

);

### Columns

| Column | Why we need it |                                                             |
|---|---| ----------------------------------------------------------- |
| `id`                 | Unique identifier for the position.                         |
| `user_id`            | Identifies who owns the contracts.                          |
| `market_id`          | Identifies which prediction market the position belongs to. |
| `outcome_id`         | Identifies whether the user owns YES or NO contracts.       |
| `quantity`           | Records the number of contracts currently owned.            |
| `created_at`         | Records when the position was first created.                |
| `updated_at`         | Records when the position was last changed.                 |

### Unique constraint

```
UNIQUE (user_id, market_id, outcome_id)
```

This ensures:

```
Neeraj | M001 | YES
```

has only **one position row**.

Instead of:

```
Neeraj | M001 | YES | 60
```

Neeraj | M001 | YES | 40

we maintain:

```
Neeraj | M001 | YES | 100
```

---

# 11. `settlements`

Settlement happens after the market is resolved.

It answers:

> **How much did this user's position receive when the market settled?**

```
CREATE TABLE settlements (
```

    id UUID PRIMARY KEY,



    market_id UUID NOT NULL,

    outcome_id UUID NOT NULL,

    user_id UUID NOT NULL,

    position_id UUID NOT NULL,



    quantity DECIMAL(20, 8) NOT NULL,

    payout_per_contract DECIMAL(20, 8) NOT NULL,

    payout_amount DECIMAL(20, 8) NOT NULL,



    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',



    settled_at TIMESTAMP,



    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,



FOREIGN KEY (market_id) REFERENCES markets(id),

FOREIGN KEY (outcome_id) REFERENCES outcomes(id),

FOREIGN KEY (user_id) REFERENCES users(id),

FOREIGN KEY (position_id) REFERENCES positions(id)

);

### Columns

| Column | Why we need it |                                                                   |
|---|---| ----------------------------------------------------------------- |
| `id`                  | Unique identifier for the settlement record.                      |
| `market_id`           | Identifies which market is being settled.                         |
| `outcome_id`          | Identifies which outcome the user held.                           |
| `user_id`             | Identifies the user receiving the settlement.                     |
| `position_id`         | Identifies the exact position being settled.                      |
| `quantity`            | Records how many contracts were settled.                          |
| `payout_per_contract` | Records how much each winning contract pays.                      |
| `payout_amount`       | Records the total amount paid to the user.                        |
| `status`              | Tracks whether settlement is `PENDING`, `COMPLETED`, or `FAILED`. |
| `settled_at`          | Records when the settlement was actually completed.               |
| `created_at`          | Records when the settlement record was created.                   |

---

# 12. Predex Binary Contract Model

We're assuming the following model:

```
YES contract:
```



YES wins → ₹100

YES loses → ₹0

and:

```
NO contract:
```



NO wins → ₹100

NO loses → ₹0

Therefore:

```
YES price = ₹63
```

means:

```
Cost              = ₹63
```

Maximum payout    = ₹100

Potential profit  = ₹37

And:

```
NO price = ₹42
```

means:

```
Cost              = ₹42
```

Maximum payout    = ₹100

Potential profit  = ₹58

For a binary market, YES and NO are complementary outcomes.

---

# 13. Complete Example

Market:

```
M001
```



"Will India win?"



YES → O001

NO  → O002

Neeraj places:

```
BUY
```

100 YES

₹60

### Order

```
O001
```

BUY 100 YES @ ₹60

Another user sells:

```
SELL
```

40 YES

₹60

Matching engine creates:

```
T001
```



40 YES @ ₹60

Neeraj's order:

```
quantity        = 100
```

filled_quantity = 40

remaining       = 60

Neeraj's position:

```
40 YES
```

Seller's position decreases by:

```
40 YES
```

---

Later, the market resolves:

```
YES = WINNER
```

Neeraj owns:

```
40 YES
```

Payout:

```
40 × ₹100
```

\= ₹4,000

Settlement:

```
S001
```



quantity            = 40

payout_per_contract = ₹100

payout_amount       = ₹4,000

status              = COMPLETED

The ₹4,000 is then recorded through the ledger and credited to the user's account.

---

# 14. Overall Relationship Diagram

```
                         USERS
```

                           │

              ┌────────────┼────────────┐

              │            │            │

              ↓            ↓            ↓

           ACCOUNTS      ORDERS      POSITIONS

              │            │            │

              ↓            │            ↓

        LEDGER_ENTRIES     │       SETTLEMENTS

                           │

                           ↓

                         TRADES





                         MARKETS

                            │

                            ↓

                         OUTCOMES

                            │

              ┌─────────────┼─────────────┐

              ↓             ↓             ↓

           ORDERS         TRADES       POSITIONS

---

# 15. Complete Data Flow

This is probably the **most important part to understand** before we start coding.

### Step 1 — User

```
USER
```

creates an account.

### Step 2 — Deposit

```
USER
```

 ↓

ACCOUNT

 ↓

LEDGER

### Step 3 — Market created

```
MARKET
```

 ├── YES

 └── NO

### Step 4 — User places order

```
USER
```

 ↓

ORDER

### Step 5 — Order enters order book

```
ORDER
```

 ↓

ORDER BOOK

### Step 6 — Matching engine

```
BUY ORDER
```

     ↕

SELL ORDER

     ↓

MATCH

### Step 7 — Trade

```
TRADE
```

### Step 8 — Financial movement

```
TRADE
```

 ↓

LEDGER

 ↓

ACCOUNT

### Step 9 — Ownership

```
TRADE
```

 ↓

POSITION

### Step 10 — Market resolution

```
MARKET
```

 ↓

RESOLVED

 ↓

YES / NO winner

### Step 11 — Settlement

```
POSITION
```

 ↓

SETTLEMENT

 ↓

LEDGER

 ↓

ACCOUNT

---

# 16. Tables at a Glance

```
users
```

│

├── accounts

│     └── ledger_entries

│

├── orders

│     └── trades

│

└── positions

      └── settlements





markets

│

└── outcomes

      │

      ├── orders

      ├── trades

      ├── positions

      └── settlements
