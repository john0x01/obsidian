# Debugging Production

## Fundamentals

### Read-Only Investigation First

### Correlation IDs for Incident Root Causing

## Inspection Techniques

### Heap Dumps and Thread Dumps

### Live Debuggers (rr, gdb, dlv)

### Profilers in Production (Continuous Profiling)

### eBPF for Dynamic Tracing

### Logpoints / Dynamic Logging

### Safe Shelling into Containers

## Traffic-Based Techniques

### Canary Traffic Analysis

### Replay of Traffic for Bug Reproduction

### Mirror / Shadow Traffic

### A/B Compare Tool

## Mitigation

### Feature Flag Rollbacks

## Operational Concerns

### Production Debugging Runbooks

### Postmortem Data Capture During Incidents

### Privacy and Compliance in Debugging

## See Also
- [[error-tracking]] — surfaces errors to investigate
- [[correlation-ids]] — root-causing by request ID
- [[distributed-tracing]] — latency and root-cause analysis
- [[profiling]] — continuous profiling in production
- [[incident-response]] — debugging during incidents
- [[feature-flags]] — flag-based mitigation
