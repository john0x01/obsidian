# API Error Handling

## HTTP Status Codes

### HTTP Status Code Semantics

### 4xx vs 5xx Boundary

## Error Response Format

### Problem Details for HTTP APIs (RFC 7807/9457)

### Error Envelopes vs Plain Bodies

### Consistent Error Shape Across Endpoints

## Error Content

### Error Codes vs Error Messages

### Machine-Readable vs Human-Readable Errors

### Localization of Error Messages

### Field-Level Validation Errors

### Including Trace IDs / Request IDs

### Retry Guidance in Error Responses

## Error Categories

### Error Taxonomy Design

### Authentication vs Authorization Errors

### Rate Limit Errors

### Idempotency Conflict Errors

## Security

### Exposing vs Hiding Internal Details

## See Also
- [[rest]] — HTTP status code semantics
- [[06-software-design/api-design/rate-limiting|Rate Limiting (API)]] — 429 rate-limit errors
- [[idempotency]] — idempotency conflict errors
- [[authentication-and-authorization]] — authn vs authz errors
- [[08-quality-and-operations/observability/correlation-ids|Correlation IDs]] — trace IDs in error responses
