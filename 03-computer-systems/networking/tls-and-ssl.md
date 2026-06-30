# TLS and SSL

## Purpose (Confidentiality, Integrity, Authentication)

## SSL History and TLS Versions (1.0, 1.1, 1.2, 1.3)

## TLS Handshake (Full, Abbreviated, 1.3 Simplified)

## Cryptographic Building Blocks

### Cipher Suites

### Key Exchange (RSA, DHE, ECDHE)

### Symmetric Encryption in TLS (AES-GCM, ChaCha20-Poly1305)

## Certificates and Identity

### Certificate Chain and Trust Stores

### Server Name Indication (SNI)

### Encrypted Client Hello (ECH)

### Certificate Pinning

### OCSP and OCSP Stapling

### mTLS (Mutual TLS)

## Session Resumption (Session IDs, Session Tickets)

### 0-RTT and Replay Risks

## Perfect Forward Secrecy

## Downgrade and Protocol Attacks (POODLE, BEAST, Heartbleed)

## See Also
- [[08-quality-and-operations/security/certificates-and-pki|Certificates and PKI]] — trust chain behind TLS
- [[network-security]] — TLS is baseline defense
- [[http]] — HTTPS layers TLS under HTTP
- [[quic-and-http3]] — embeds TLS 1.3
- [[08-quality-and-operations/security/symmetric-encryption|Symmetric Encryption]] — bulk cipher in TLS
