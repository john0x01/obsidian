# Rollback Strategies

## Fundamentals

### Fast Rollback as a Safety Net

### Versioned Artifacts for Rollback

### Rollback vs Roll-Forward Decisions

## Rollback Mechanics

### Blue-Green Rollback (Swap Traffic)

### Canary Rollback (Revert Weights)

### Feature Flag Rollback

### Automated Rollback on Health Check Failure

### Manual Rollback Procedures

### Rollback Runbooks

## Data and Schema Concerns

### Schema Migration Rollback Challenges

### Expand-Contract Migrations (Backward-Compatible)

### Data Backfill Considerations

### State Compatibility Across Versions

## Operational Follow-Up

### Rollback Observability

### Communication During Rollback

### Post-Rollback Root Cause Analysis

### Mean Time to Rollback (MTTR) Metric

## See Also
- [[deployment-strategies]] — strategies rollback reverses
- [[feature-flags]] — instant flag-based rollback
- [[immutable-infrastructure]] — rollback via image swap
- [[incident-response]] — rollback as mitigation step
- [[artifact-management]] — versioned artifacts to revert to
