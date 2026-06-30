# Hermes Bytecode Reverse Engineering

## Fundamentals

### Hermes Bytecode Format

### Comparison to Plain JS Reverse Engineering

### JSC vs Hermes Threat Model Differences

## Tooling

### hbcdump and Disassemblers

### Function and String Table Recovery

### Native Module Inspection (Frida, Objection)

## Attack Surface

### Source Map Leakage Risks

### Symbolication Risk in Crash Reports

### OTA Bundle Interception

### API Key Recovery from Bundles

## Defenses

### Obfuscation Options (String Encryption, Renaming)

### Tampering Detection

### Anti-Debugging Techniques

### Mitigations Layered with Server-Side Checks

### Proof-Carrying Releases and Attestation

## Legal

### Legal and Responsible Disclosure Boundaries

## See Also
- [[secure-storage]] — keeps secrets out of bytecode
- [[jailbreak-root-detection]] — rooting eases dumps
- [[certificate-pinning]] — reversing reveals pins
