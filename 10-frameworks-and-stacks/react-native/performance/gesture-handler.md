# Gesture Handler

## Fundamentals

### Native Gesture Recognition Model

### State Machine (UNDETERMINED, BEGAN, ACTIVE, END, CANCELLED, FAILED)

### Pointer Events vs Touch Events

### GestureHandlerRootView

## Gesture Types

### Built-In Gestures (Pan, Tap, LongPress, Pinch, Rotation, Fling)

### Composed Gestures (Race, Exclusive, Simultaneous)

## Gesture Relationships

### Simultaneous vs Exclusive Gestures

### waitFor and requireToFail Relationships

### Hit Slop and Hit Testing Customization

## Integration

### Coordination with Reanimated Worklets

### Conflicts with Native ScrollView / FlatList

## Platform Bridging

### iOS UIGestureRecognizer Bridging

### Android MotionEvent Bridging

### Platform-Specific Quirks

## Advanced Concerns

### Accessibility and Gesture Conflicts

### Testing Gestures

## See Also
- [[reanimated-worklets]] — gestures wired to worklets
- [[thread-delegation]] — gesture handling off JS thread
- [[animation-optimization]] — gesture-driven animation smoothness
