# Case Study: Payments

## Requirements

### Functional Requirements (Authorize, Capture, Refund, Void)

### Non-Functional Requirements (Consistency, Auditability, Durability)

## Core Ledger Model

### Double-Entry Ledger Model

### Audit Log and Immutability

## Correctness Guarantees

### Idempotency Keys for Every Operation

### Exactly-Once Semantics via Idempotency

### Saga Pattern for Multi-Step Payments

### Retry and Reconciliation

## Payment Methods and Security

### Payment Methods (Card, Bank, Wallet, Crypto)

### PCI DSS Compliance

### Tokenization of Sensitive Data

### 3D Secure and SCA

## Money Movement

### Currency Conversion and FX

### Settlement and Clearing

### Refund Handling

### Chargebacks and Disputes

## Risk and Routing

### Risk and Fraud Checks

### Multi-Processor Routing

## See Also
- [[idempotency]] — idempotency keys for retries
- [[distributed-transactions]] — saga and multi-step consistency
- [[transactions-and-acid]] — ledger consistency guarantees
- [[event-sourcing]] — immutable audit log model
