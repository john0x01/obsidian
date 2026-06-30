# Secure Storage

## Fundamentals

### Why AsyncStorage Is Not Secure

### Data at Rest vs In Memory

## Platform-Backed Stores

### iOS Keychain (kSec Access Classes, Access Control)

### Android Keystore (Hardware-Backed Keys)

### Encrypted SharedPreferences (Android)

## Libraries

### react-native-keychain, expo-secure-store

### MMKV with Encryption Key

## Key Management

### Key Derivation for Storage Keys

### Key Rotation Strategies

### Biometric-Protected Storage

## Secrets Handling

### Secrets in JS Bundle (Don't)

### Secrets in Native Source (Don't)

### Secrets Delivered Post-Authentication

### Token Storage Best Practices (Access, Refresh)

## Edge Cases

### Secure Clipboard Handling

### Backup Exclusion (Android allowBackup, iOS NSURLIsExcludedFromBackupKey)

## See Also
- [[certificate-pinning]] — protects data in transit
- [[jailbreak-root-detection]] — compromised OS breaks storage
- [[device-attestation]] — trust the device first
- [[08-quality-and-operations/security/secret-management|Secret Management]] — where app secrets live
