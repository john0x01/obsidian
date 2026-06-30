# API Authentication and Authorization

## Fundamentals

### AuthN vs AuthZ Distinction

## Authentication Mechanisms

### API Keys

### Basic Auth

### Bearer Tokens

### JWT

### HMAC-Signed Requests (AWS SigV4-style)

### mTLS for Service-to-Service

## Delegated Authentication

### OAuth 2.0 Flows (Authorization Code, Client Credentials, Device, PKCE)

### OpenID Connect on Top of OAuth

## Sessions and Tokens

### Session Cookies vs Tokens

### Refresh Tokens

### Token Revocation and Blacklists

### Audience and Issuer Validation

## Authorization Models

### Scopes and Permissions

### Role-Based Access Control (RBAC)

### Attribute-Based Access Control (ABAC)

### Policy-Based Access Control (OPA, Cedar)

### Multi-Tenant Authorization

## See Also
- [[08-quality-and-operations/security/authentication|Authentication]] — identity verification deep-dive
- [[08-quality-and-operations/security/authorization|Authorization]] — access-control models deep-dive
- [[08-quality-and-operations/security/oauth|OAuth]] — delegated authorization flows
- [[08-quality-and-operations/security/jwt|JWT]] — bearer token format
- [[api-gateways]] — auth enforced at the edge
