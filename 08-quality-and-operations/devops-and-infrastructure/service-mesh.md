# Service Mesh

## Architecture

### Data Plane vs Control Plane

### Sidecar Proxy Model

### Ambient / Sidecar-less Models

### eBPF-Based Meshes

## Implementations

### Envoy as Data Plane

### Istio, Linkerd, Consul Connect, Cilium

## Traffic Management

### Traffic Routing and Splitting

### Retries, Timeouts, Circuit Breakers

### Rate Limiting

### Canary and A/B with Mesh

## Security and Policy

### mTLS Between Services

### Policy Enforcement

## Observability

### Observability (Metrics, Traces, Logs)

## Scale and Trade-offs

### Multi-Cluster Mesh

### Performance Overhead

### Operational Complexity

### Service Mesh vs API Gateway Boundaries

## See Also
- [[kubernetes-concepts]] — common mesh deployment substrate
- [[orchestration]] — layer the mesh sits above
- [[microservices]] — the architecture meshes serve
- [[api-gateways]] — north-south vs east-west boundary
- [[06-software-design/system-design/rate-limiting|Rate Limiting (system design)]] — mesh-enforced rate limits
- [[06-software-design/system-design/load-balancing|Load Balancing (system design)]] — mesh traffic distribution
