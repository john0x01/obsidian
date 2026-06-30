# JavaScript Runtime

## JavaScript Engine

### Call Stack

The **call stack** executes synchronous code. It breaks work into function calls: for every function invocation, a stack frame is created. Functions can also create Promises, which are pushed into the Promise API.

#### Execution Context

The **execution context** (also called a stack frame) is created by the call stack in order to execute a received function. Alongside the function code, the execution context holds:

- The value of `this`
- The function code
- The arguments value
- The variable mapping

To build the variable mapping, the execution context accesses the heap memory to find the actual values of those variables. Both the global context and the local context are injected into the execution context when it is loaded.

### Heap

## Runtime APIs

### Event Listeners

### Timer API

Pushes the timer callback to the macro-task queue.

### Promise API

Pushes the promise callback to the micro-task queue.

## Event Loop

### Task Queue

### Micro-task Queue
