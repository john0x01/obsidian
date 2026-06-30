# WebSockets (API Design Perspective)

## Fundamentals

### When WebSockets Fit Your API

### Connection Lifecycle Design

### Subprotocol Negotiation

## Message Design

### Message Framing and Schema

### Request-Response over Persistent Connection

### Correlation IDs for RPC Semantics

## Security and Access Control

### Authentication on Upgrade

### Authorization per Message

### Rate Limiting on Persistent Connections

### Security Considerations (Origin Checks, CSRF)

## Reliability

### Reconnection and Resumption

### Sequence Numbers and Missed Messages

### Graceful Close and Backoff

### Fallback to SSE or Long Polling

## Scaling and Operations

### Scaling Horizontally (Pub/Sub Backplanes)

### Observability for WebSocket APIs

## See Also
- [[server-sent-events]] — one-way streaming alternative
- [[03-computer-systems/networking/websockets|WebSockets (protocol)]] — underlying protocol mechanics
- [[06-software-design/api-design/rate-limiting|Rate Limiting (API)]] — limits on persistent connections
- [[authentication-and-authorization]] — auth on upgrade
- [[06-software-design/system-design/pub-sub|Pub/Sub]] — horizontal scaling backplane
