# Server-Side Request Forgery (SSRF)

## SSRF Attack Surface

## SSRF Types

### Basic SSRF (Direct Responses)

### Blind SSRF

## Attack Targets and Techniques

### Cloud Metadata Endpoint Attacks (169.254.169.254)

### Protocol Smuggling (gopher://, file://, dict://)

## Bypass Techniques

### URL Parser Discrepancies

### Bypasses via Redirects

### IPv6 and Octal / Hex Encoding Bypasses

### DNS Rebinding

## Vulnerable Sinks

### SSRF in Webhooks and Server-Side Image Fetchers

### SSRF in PDF / Image Generation Libraries

## Defenses

### Allow-List vs Block-List Approaches

### Outbound Network Segmentation

### Egress Firewalls

### IMDSv2 as Cloud Mitigation

## Detection via Logging and Monitoring
