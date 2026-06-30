# Multi-Tenancy

## Isolation Models

### Tenant Isolation Levels (Silo, Pool, Bridge)

### Data Isolation Strategies (Schema, Row-Level, Database-per-Tenant)

### Compute Isolation

### Network Isolation

### Cell-Based Architecture

## Tenant Routing and Context

### Tenant Routing

### Tenant Identifier Propagation

## Resource Fairness

### Noisy Neighbor Problem

### Fair Scheduling Across Tenants

### Per-Tenant Rate Limiting

## Operational Concerns

### Tenant Onboarding and Offboarding

### Per-Tenant Observability

### Billing and Metering per Tenant

### Upgrade Strategies Across Tenants

### Tenant-Specific Customization

## Security and Compliance

### Tenant Security Boundaries

### Compliance and Data Residency per Tenant

## See Also
- [[multi-region-and-geo]] — data residency across regions
- [[sharding-and-partitioning]] — per-tenant data isolation
- [[06-software-design/system-design/rate-limiting|Rate Limiting (system design)]] — per-tenant rate limiting
- [[scalability]] — noisy-neighbor and pool sizing
- [[authorization]] — tenant security boundaries
