# Layered Architecture

## Fundamentals

### Classic Three-Tier (Presentation, Logic, Data)

### N-Tier Variations

### Layer Responsibilities and Boundaries

## Layering Rules

### Strict vs Relaxed Layering

### Dependency Direction (Top-Down Only)

### Persistence Ignorance

## Cross-Layer Mechanics

### Data Transfer Objects Between Layers

### Cross-Cutting Concerns Across Layers

### Anti-Corruption Layer

## Quality Attributes

### Testability by Layer

### Layer Replacement and Swap-Out

### Performance Impact of Layer Boundaries

## Pitfalls

### Common Violations (Skip-Layer Calls)

### When Layered Becomes Anemic

## Comparisons

### Comparison with Hexagonal / Clean Architecture

## See Also
- [[clean-architecture]] — refines layering with dependency rule
- [[hexagonal-architecture]] — ports-and-adapters alternative
- [[onion-architecture]] — inverts the layer dependencies
- [[coupling-and-cohesion]] — layer boundaries manage coupling
- [[mvc-mvp-mvvm]] — presentation-layer patterns
