# Server-Sent Events (SSE)

## Fundamentals

### One-Way Server-to-Client Streaming

### Event Stream Format (text/event-stream)

## Protocol Features

### Named Events

### Automatic Reconnection by Browser

### Last-Event-ID Resumption

### Retry Interval Hints

### Heartbeats and Keepalive Comments

## Comparisons

### Comparison with WebSockets

### Comparison with Long Polling

## Operational Concerns

### HTTP/2 and SSE

### Load Balancer and Proxy Considerations

### Scaling SSE (Backplane, Sharding)

### CORS and Credentials

### Error Handling in SSE

## Applications

### Use Cases (Notifications, Feeds, Progress)

## See Also
- [[06-software-design/api-design/websockets|WebSockets (API)]] — bidirectional streaming alternative
- [[03-computer-systems/networking/http|HTTP]] — runs over plain HTTP
- [[06-software-design/system-design/load-balancing|Load Balancing (system design)]] — proxy/balancer considerations
- [[03-computer-systems/networking/websockets|WebSockets (protocol)]] — protocol-level comparison
