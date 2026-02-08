# Chronos Bank 🏦⏳

![Go Version](https://img.shields.io/badge/go-1.21+-00ADD8?logo=go)
![Architecture](https://img.shields.io/badge/Pattern-Transactional%20Outbox-orange)
![Consistency](https://img.shields.io/badge/Consistency-Strong%20Eventual-green)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

**Chronos Bank** is a reference implementation of a distributed Core Banking Ledger designed for correctness and auditability.

Unlike traditional CRUD ledgers that store mutable balances, Chronos uses **Event Sourcing** to reconstruct the state of any account from an immutable history of facts. It solves the critical distributed system challenge of **Dual Write** (writing to the database and publishing to a queue) by implementing the **Transactional Outbox Pattern**.

## 🏗️ Architecture & Design Decisions

The system is built on the **CQRS** (Command Query Responsibility Segregation) principle, splitting the write and read paths to optimize for both consistency and latency.

### 1. The Write Path (Command Side) - "Safety First"
* **Responsibility:** Validates business rules (e.g., "Non-negative Balance") and persists events.
* **Technology:** Go (gRPC Service) + PostgreSQL.
* **Consistency Mechanism:** To ensure **ACID** guarantees, we do not publish directly to Kafka. Instead, we insert the event into an `outbox` table within the *same* PostgreSQL transaction as the business data.
    * *Result:* It is impossible to have a "phantom payment" where the DB is updated but the event is lost.

### 2. The Read Path (Query Side) - "Performance"
* **Responsibility:** Serves high-speed balance inquiries and transaction history.
* **Technology:** Kafka + Elasticsearch / Redis.
* **Projections:** An asynchronous worker consumes events from Kafka and updates the denormalized views (Read Models).

## 🛠️ Tech Stack

* **Core:** Go (Golang)
* **Communication:** gRPC (Protobuf)
* **Event Store:** PostgreSQL (Source of Truth)
* **Message Broker:** Apache Kafka
* **Read Database:** Elasticsearch
* **Observability:** OpenTelemetry (Tracing), Prometheus

## 🚀 Transaction Flow (The Outbox Pattern)

1.  **Initiate:** Client calls `TransferFunds(From: A, To: B, Amount: 100)`.
2.  **Transaction Start:** The Command Service opens a SQL Transaction.
3.  **Persist:**
    * Validate invariants (Is Balance A >= 100?).
    * Insert `MoneyWithdrawn` event to `Events` table.
    * Insert payload to `Outbox` table.
4.  **Commit:** The SQL transaction commits. Data is safe on disk.
5.  **Relay:** A separate **Outbox Poller** process (or CDC connector) reads the `Outbox` table and pushes the message to Kafka.
6.  **Project:** The Query Service consumes the Kafka message and updates the balance in Elasticsearch.

## 📦 Getting Started

### Prerequisites
* Go 1.21+
* Docker & Docker Compose

### Running the Infrastructure

```bash
# Spin up Postgres, Kafka, Zookeeper, and Elasticsearch
docker-compose up -d
