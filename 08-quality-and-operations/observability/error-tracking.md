# Error Tracking

## Fundamentals

### Purpose of Error Tracking

### Error Rate vs Error Count

## Capture Mechanics

### Stack Trace Capture

### Source Maps for Frontend Errors

### Breadcrumbs / Context Capture

### User Context Attached to Errors

## Grouping and Triage

### Error Grouping / Fingerprinting

### Regression Detection (New vs Seen Errors)

### Notification and Triage Workflows

## Noise and Volume Control

### Ignore Rules and Noise Reduction

### Sampling and Rate Limiting Errors

## Release and Correlation

### Release Tracking

### Correlation with Traces and Logs

## Privacy and Compliance

### PII Scrubbing in Error Reports

## Tooling and Deployment Trade-offs

### Sentry, Rollbar, Bugsnag, Raygun

### Self-Hosted vs SaaS Trade-offs

## See Also
- [[debugging-production]] — investigating captured errors
- [[distributed-tracing]] — correlating errors with traces
- [[logging]] — log context behind error reports
- [[real-user-monitoring]] — browser-side error capture
- [[deployment-strategies]] — release tracking and regressions
