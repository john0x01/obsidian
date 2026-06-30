# Race Conditions

## Fundamentals

### What Constitutes a Race

### Data Races vs Race Conditions

## Common Race Patterns

### Read-Modify-Write Hazards

### Check-Then-Act Anti-Pattern

### TOCTOU (Time-of-Check to Time-of-Use)

### Initialization Races

### Publication Races and Safe Publication

### Double-Checked Locking

## Special Cases

### Benign Races

## Mitigation Strategies

### Atomic Operations as a Remedy

### Immutability as a Race Mitigation

### Confinement to Single Thread

## Detection and Examples

### Tools for Race Detection (ThreadSanitizer, Helgrind)

### Examples in Real Systems (Counters, Caches, Singletons)
