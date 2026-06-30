# Event Loops

## Single-Threaded Concurrency Model

## Run-to-Completion Semantics

## Task Queues (Macro Tasks, Micro Tasks)

## I/O Polling Inside the Loop

## Timers and Scheduled Callbacks

## Fairness Between Queues

## Reactor vs Proactor Patterns

## Worker Threads for CPU Work

## Integration with Native Blocking APIs

## Multiple Event Loops per Process

## Libuv, Tokio, Asyncio Internals

## Starvation from Long Tasks

## Event Loop Lag Measurement

## Common Pitfalls (Blocking the Loop, Unhandled Rejections)

## See Also
- [[async-and-await]] — async tasks run on the loop
- [[futures-and-promises]] — microtask queue resolves promises
- [[03-computer-systems/operating-systems/io-models|I/O Models]] — readiness drives the loop
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime]] — concrete event loop implementation
- [[01-programming-foundations/paradigms/event-driven-programming|Event-Driven Programming]] — the paradigm it realizes
