# Predex

> **Trade Your Opinions.**

A scalable, event-driven **prediction market exchange** where users trade prediction contracts on future events through a centralized order book and a price-time-priority matching engine.

---

# Overview

Predex is inspired by modern financial exchanges like **NSE**, **BSE**, and prediction markets like **Polymarket**. Instead of trading stocks, users trade **prediction contracts** on the outcomes of future events.

Examples:

- Will India win tomorrow's match?
- Will Bitcoin cross $150,000 before Dec 2027?
- Will Apple beat earnings expectations?
- Will Candidate A win the election?
- Will GPT-6 launch before 2028?

Unlike a traditional betting platform where users bet against the house, Predex allows users to **trade with each other**.

The platform acts only as the exchange by providing:

- Market creation
- Order books
- Matching engine
- Wallet management
- Settlement
- Real-time market data

---

# The Idea

People have different opinions about future events.

Someone believes Bitcoin will increase.

Someone else believes it will decrease.

Someone thinks Team India will win.

Someone thinks Australia will win.

Instead of simply placing bets against the platform, users should be able to **trade those opinions** with each other.

Predex transforms opinions into tradable contracts.

The exchange never predicts outcomes.

The exchange never takes the opposite side of a trade.

The exchange simply connects buyers and sellers.

---

# Traditional Betting vs Prediction Exchange

## Traditional Betting

```
User
   │
Place Bet
   │
Platform accepts bet
   │
Wait for Result
   │
Platform pays winner
```

- Platform sets odds
- Platform is the counterparty
- No order book
- No matching engine

---

## Predex

```
Buyer
      \
       \
     Exchange
       /
      /
Seller
```

- Users decide prices
- Users trade with each other
- Exchange matches orders
- Exchange charges transaction fees

---

# Core Concepts

---

## Market

A market represents a future event.

Examples:

```
Will India beat Australia?

YES
NO
```

```
Will Bitcoin reach $200,000?

YES
NO
```

Every market has only one final outcome.

---

## Prediction Contract

Instead of trading stocks, users trade prediction contracts.

Example

```
YES Contract

Current Price

₹0.63
```

If the event occurs

```
YES Contract

Value

₹1
```

Otherwise

```
YES Contract

Value

₹0
```

---

## Order

An order is a request to buy or sell.

Example

```
BUY

100 Contracts

@ ₹0.63
```

or

```
SELL

50 Contracts

@ ₹0.65
```

Orders remain open until

- matched
- cancelled
- expired

---

## Order Book

Every market has its own order book.

```
BUY (Bids)

₹0.63     100

₹0.62     250

₹0.61     500


--------------------


SELL (Asks)

₹0.64     80

₹0.65     300

₹0.66     120
```

The order book contains all pending buy and sell orders.

---

## Matching Engine

The matching engine is the heart of the exchange.

Its job is very simple.

```
Highest Buy

>=

Lowest Sell
```

If true

↓

Trade executes.

Otherwise

↓

Order waits.

The matching engine follows **Price-Time Priority**.

Priority is given to

1. Better price
2. Earlier order

---

## Price Discovery

The exchange never decides prices.

Prices are discovered naturally through trading.

Example

```
BUY

₹0.64
```

```
SELL

₹0.64
```

Trade happens.

Latest traded price becomes

```
₹0.64
```

---

## Wallet

Every user has

```
Available Balance

Locked Balance
```

When placing an order

```
Available

₹1000

↓

BUY

₹200

↓

Available

₹800

Locked

₹200
```

---

## Settlement

When a trade occurs

The settlement engine

- updates wallets
- updates positions
- stores trade history
- publishes events

---

## Market Resolution

When the event finishes

Example

```
India Wins
```

Then

```
YES = ₹1

NO = ₹0
```

Every open position is settled automatically.

---

# High Level Architecture

```
                         Next.js Frontend
                                │
                     REST API + WebSockets
                                │
                     Fastify API Gateway
                                │
        ┌───────────────────────┼────────────────────────┐
        │                       │                        │
        │                  PostgreSQL                 Redis
        │               (Source of Truth)        (Cache + Live State)
        │                       │                        │
        └──────────────┬────────┴──────────────┬────────┘
                       │                       │
                 Redis Streams          Socket.IO
                  (Event Bus)          (Real Time)
                       │
      ┌────────────────┼──────────────────────┐
      │                │                      │
 Order Service   Matching Engine      Notification Service
      │                │
      │         Trade Executed
      │                │
      └────────────Settlement───────────────┐
                     │                      │
               Wallet Service         Position Service
```

---

# Event Driven Flow

```
User

↓

Place Order

↓

Order Service

↓

Store Order

↓

Publish

OrderCreated Event

↓

Matching Engine

↓

TradeExecuted Event

↓

Settlement Service

↓

Wallet Updated

↓

Trade Stored

↓

Notification Service

↓

WebSocket Broadcast

↓

Frontend Updates
```

Every component communicates through events instead of directly calling each other.

This allows the system to scale independently.

---

# Backend Modules

```
Authentication

↓

User

↓

Wallet

↓

Market

↓

Order

↓

Matching Engine

↓

Settlement

↓

Notification

↓

Analytics
```

Each module owns one responsibility.

---

# Why Event Driven?

Instead of

```
Place Order

↓

Match

↓

Wallet

↓

Database

↓

Notify

↓

Response
```

Predex uses

```
Place Order

↓

Save Order

↓

Publish Event

↓

Return Response
```

Background workers perform the remaining work asynchronously.

Benefits

- Faster APIs
- Better scalability
- Loose coupling
- Easier recovery
- Independent services

---

# Why Redis?

Redis is used because order books require extremely fast access.

Redis stores

- Active Order Books
- Live Prices
- Cache
- Sessions
- Pub/Sub
- Distributed Locks

Redis is **not** the permanent database.

---

# Why PostgreSQL?

PostgreSQL stores permanent data.

- Users
- Wallets
- Markets
- Orders
- Trades
- Positions
- Transactions

It acts as the source of truth.

---

# Real-Time Communication

Socket.IO broadcasts

- Latest trades
- Price updates
- Order book changes
- Market status
- User notifications

Every connected client receives updates instantly.

---

# Scalability

Initially Predex is built as a **Modular Monolith**.

Internally every module communicates using events.

Later these modules can be deployed independently.

```
API Gateway

↓

Order Service

↓

Matching Engine Cluster

↓

Settlement Service

↓

Notification Service
```

The architecture is designed to evolve into microservices without changing business logic.

---

# Technology Stack

## Frontend

- Next.js
- TypeScript
- Tailwind CSS
- Zustand
- TanStack Query
- Socket.IO Client

---

## Backend

- Fastify
- TypeScript
- Prisma ORM
- Zod

---

## Database

- PostgreSQL

---

## Cache

- Redis

---

## Event Bus

- Redis Streams

---

## Real Time

- Socket.IO

---

## Authentication

- JWT
- Refresh Tokens
- HTTP Only Cookies

---

## Background Workers

- Redis Streams Consumers

---

## Deployment

- Docker
- Docker Compose

---

# Future Enhancements

- Market categories
- Limit Orders
- Market Orders
- Stop Orders
- Trading Charts
- Candlestick Data
- Portfolio Analytics
- Leaderboards
- Market Search
- Liquidity Analytics
- Admin Dashboard
- Rate Limiting
- Event Replay
- Snapshot Recovery
- Horizontal Scaling
- Kafka Migration
- Kubernetes Deployment

---

# Learning Objectives

This project is built to understand and implement concepts used in modern exchanges and distributed systems.

- Order Book Design
- Matching Engines
- Event-Driven Architecture
- Message Queues
- Distributed Systems
- Real-Time Systems
- WebSockets
- Transaction Management
- Price Discovery
- Wallet Consistency
- Scalable Backend Design
- Modular Architecture
- High Performance APIs
- Financial System Fundamentals

---

# Vision

Predex is not just another CRUD application.

The goal is to build a backend that demonstrates how modern exchanges work internally using scalable architecture, asynchronous communication, centralized order books, and real-time event processing.

It is an engineering-focused project inspired by financial exchanges, adapted for prediction markets where **opinions become tradable assets**.