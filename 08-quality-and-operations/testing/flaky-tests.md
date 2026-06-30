# Flaky Tests

## What Makes a Test Flaky

## Common Causes (Timing, Ordering, Resources, Network, Randomness)

## Flaky Test Impact (Lost Confidence, CI Noise)

## Detection Strategies

### Automatic Flake Detection in CI

### Root Cause Analysis Approach

## Mitigation Workflows

### Quarantine Workflows

### Retrying Flaky Tests Selectively

## Remediation Techniques

### Test Isolation Improvements

### Determinism Injection (Fixed Clocks, Seeds)

### Replacing Sleeps with Event Waits

## Sources of Flakiness

### Parallel Test Flakiness

### Environmental Flakiness

### Third-Party Service Flakiness

## Metrics for Flake Rate

## Culture Around Flaky Test Ownership

## See Also
- [[end-to-end-testing]] — where flakiness concentrates
- [[regression-testing]] — undermines regression suites
- [[integration-testing]] — timing/dependency flakiness
