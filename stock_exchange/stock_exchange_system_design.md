# Stock Exchange Question

## Problem Statement

Build a stock exchange system that matches buyers and sellers of an asset efficiently.

---

## Product Requirements

### Personas

Personas are ways to categorize and define our users. They provide attributes that help us to understand who they are, what skills they have, what they are trying to accomplish, and what problems they are trying to solve.

#### Persona: Retail Investor

An individual who buys and holds a cryptocurrency for 1 year or longer. Generally interested in the fundaments of a cryptocurrency, not the short-term fluctuations.

#### Persona: Retail Trader

An individual who actively buys and sells cryptocurrencies, attempting to capture short-term profit. Interested in charts and technical indicators.

#### Persona: Market Maker

An institution that actively trades both sides of the market, attempting to make the spread between the bid and the ask price.

### Functional Requirements

1. Place a market order to buy or sell a cryptocurrency.
1. Place a limit order to buy or sell a cryptocurrency.
1. Cancel a placed order that is not currently filled.
1. Get the current status of an order.
1. Get the last traded price for a cryptocurrency.
1. Get the current bid and ask prices for a cryptocurrency.
1. Subscribe to the market data stream for a cryptocurrency

### Non-functional Requirements

1. The system should be able to handle up to 1 million transactions per second.
1. The system should process orders as quickly as possible.
1. The system should maintain strong consistency for a customer's account balance.

### Out-of-Scope Requirements

1. Transfer money into a customer's Exchange account.
1. Generate a user's trading history report.
1. Generate a user's tax statement.

---

## Assumptions and Capacity Estimation

### Assumptions

## High-Level System Design

The following is an initial basic system design prior to any kind of optimization or scaling.

![High-level System Design Diagram](/docs/diagrams/out/high_level_system_design/high_level_system_design.png)

## Links
[Coinbase Exchange APIs](https://docs.cloud.coinbase.com/exchange/docs)
