# Zerodha Product Dissection & SQL Schema Design

**Domain:** FinTech / Stock Broking  
**Tools:** SQL · Schema Design · ER Diagram · Product Analysis  
**Company:** Zerodha — India's largest retail discount stockbroker (13M+ active clients)

---

## Project Overview

A comprehensive product dissection and database schema design for Zerodha's core trading platform. This project analyses the real-world problems Zerodha solves for retail traders, dissects the product features that address those problems, and designs a normalised SQL schema that reflects Zerodha's actual operational and regulatory requirements as a SEBI-registered stockbroker.

The case study is grounded in personal experience as an active F&O and intraday trader on Zerodha's Kite platform.

---

## Scope

- Product feature analysis: Kite, GTT Orders, Console, Coin, Kite Connect API
- Real-world problems identified and how Zerodha's product design addresses each
- SQL schema design for 6 core entities
- ER diagram with full relationship mapping
- Rationale for every design decision — regulatory and operational

---

## Real-World Problems Addressed

### Problem 1: Inaccessibility of Professional Trading Tools
Traditional brokers required heavy desktop software, phone-based orders, and had interfaces too slow for intraday use. Kite's browser/mobile platform reduced F&O order placement to 3 clicks with real-time streaming data and TradingView-integrated charting — no separate tools needed.

### Problem 2: Absence of Persistent Conditional Orders
Indian exchanges do not natively support long-duration conditional orders. Zerodha's GTT (Good Till Triggered) system built a conditional automation layer on top of exchange infrastructure — allowing traders to set trigger-based orders that fire without screen monitoring. Operationally transformative for positional traders.

### Problem 3: Opacity in Brokerage Reporting and Tax Compliance
F&O income is treated as business income under Indian tax law (ITR-3), requiring precise turnover computation. Console generates segment-wise P&L, absolute turnover (as required by the IT Act), capital gains breakdowns, and brokerage charge summaries — what previously took days of manual reconstruction now takes seconds.

---

## SQL Schema Design

### Entities

| Entity | Purpose |
|---|---|
| `User` | Client account — KYC, PAN, segment access (SEBI-mandated) |
| `Instrument` | Shared catalogue — equities, F&O contracts, ETFs, indices |
| `Order` | All exchange orders — market, limit, stop-loss, GTT-triggered |
| `Holding` | Demat holdings — quantity, avg cost, pledged quantity |
| `GTT_Trigger` | Conditional order rules — single trigger and OCO |
| `Position` | Open intraday and overnight F&O positions |

### Key Design Decisions

**Order vs Position separation** — Orders represent intent (may be rejected/cancelled). Positions represent confirmed open exposure. Merging them would corrupt the order lifecycle and misrepresent trade confirmation flow.

**GTT as a standalone entity** — A GTT is a rule, not an order. It may remain active for weeks without generating any exchange order. Modelling it inside the Order table would pollute order history with inactive records.

**Instrument as shared catalogue** — All four trading entities (Orders, Holdings, GTT Triggers, Positions) reference a single Instrument table. Centralising F&O attributes (expiry, strike, lot size) prevents duplication and ensures consistency when exchange authorities revise contract specifications.

**Pledged_Qty on Holding** — SEBI regulations require pledged shares to be tracked separately. The system must always compute available-to-sell quantity to prevent regulatory violations from over-selling pledged securities.

**Segment_Access as SET on User** — SEBI mandates explicit client activation with separate risk disclosures per segment (EQ, F&O, Commodity, Currency). Storing as SET allows a single-field permission check at order entry without an additional join — critical for minimising order placement latency.

---

## Schema (Key Tables)

```sql
-- User Entity
CREATE TABLE User (
    UserID        VARCHAR(20)  PRIMARY KEY,
    Full_Name     VARCHAR(100) NOT NULL,
    Email         VARCHAR(100) UNIQUE NOT NULL,
    Mobile        VARCHAR(15)  UNIQUE NOT NULL,
    PAN           CHAR(10)     UNIQUE NOT NULL,
    Account_Type  ENUM('Individual', 'HUF', 'Corporate') NOT NULL,
    KYC_Status    ENUM('Verified', 'Pending', 'Rejected') NOT NULL,
    Registration_Date DATE     NOT NULL,
    Segment_Access SET('EQ','FO','CDS','COM') NOT NULL
);

-- Order Entity
CREATE TABLE Order (
    OrderID          VARCHAR(30) PRIMARY KEY,
    UserID           VARCHAR(20) NOT NULL,
    InstrumentID     VARCHAR(30) NOT NULL,
    Order_Type       ENUM('Market','Limit','Stop-Loss','SL-Market') NOT NULL,
    Transaction_Type ENUM('BUY','SELL') NOT NULL,
    Quantity         INT NOT NULL,
    Price            DECIMAL(10,2),
    Status           ENUM('Open','Executed','Cancelled','Rejected') NOT NULL,
    Product_Type     ENUM('CNC','MIS','NRML') NOT NULL,
    Timestamp        DATETIME NOT NULL,
    Exchange         ENUM('NSE','BSE','NFO','MCX','CDS') NOT NULL,
    GTT_Flag         BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (UserID) REFERENCES User(UserID),
    FOREIGN KEY (InstrumentID) REFERENCES Instrument(InstrumentID)
);

-- GTT_Trigger Entity
CREATE TABLE GTT_Trigger (
    GTT_ID           VARCHAR(30) PRIMARY KEY,
    UserID           VARCHAR(20) NOT NULL,
    InstrumentID     VARCHAR(30) NOT NULL,
    Trigger_Type     ENUM('Single','OCO') NOT NULL,
    Trigger_Price    DECIMAL(10,2) NOT NULL,
    Limit_Price      DECIMAL(10,2) NOT NULL,
    Quantity         INT NOT NULL,
    Transaction_Type ENUM('BUY','SELL') NOT NULL,
    Status           ENUM('Active','Triggered','Cancelled','Expired') NOT NULL,
    Created_At       DATETIME NOT NULL,
    Expires_At       DATE NOT NULL,
    FOREIGN KEY (UserID) REFERENCES User(UserID),
    FOREIGN KEY (InstrumentID) REFERENCES Instrument(InstrumentID)
);
```

---

## ER Diagram

```
User (1) ──────────── (M) Order
User (1) ──────────── (M) Holding
User (1) ──────────── (M) GTT_Trigger
User (1) ──────────── (M) Position

Instrument (1) ─────── (M) Order
Instrument (1) ─────── (M) Holding
Instrument (1) ─────── (M) GTT_Trigger
Instrument (1) ─────── (M) Position
```

Full ER diagram with attributes available in `Zerodha_Product_Dissection.pdf`

---

## Files

| File | Description |
|---|---|
| `Zerodha_Product_Dissection_Shiv.pdf` | Full case study with ER diagram and schema tables |

---


