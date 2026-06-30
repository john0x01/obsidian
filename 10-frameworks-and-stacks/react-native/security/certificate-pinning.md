# Certificate Pinning

## Fundamentals

### Why Pin on Mobile

### SPKI Pinning vs Certificate Pinning

### Pinning vs mTLS

## Implementation

### Pinning Libraries (TrustKit iOS, OkHttp CertificatePinner Android)

### RN Integrations (react-native-ssl-pinning, react-native-cert-pinner)

## Rotation

### Backup Pins for Rotation

### Pin Rotation Strategies

### Reporting-Only Mode Before Enforcement

### Rotation Tooling in OTA Channels

## Failure Handling and Observability

### Handling Pinning Failures Gracefully

### Observability for Pinning Errors

## Variants and Contexts

### Pinning in WebViews

### Pinning with HTTP/3 and QUIC

### Pinning for Third-Party SDKs

## Threats and Defenses

### Bypassing Pinning (Frida, Objection) and Defenses

## See Also
- [[03-computer-systems/networking/tls-and-ssl|TLS and SSL]] — underlying transport security
- [[08-quality-and-operations/security/certificates-and-pki|Certificates and PKI]] — trust chain pinning relies on
- [[secure-storage]] — protects data at rest
- [[device-attestation]] — verify client before trust
