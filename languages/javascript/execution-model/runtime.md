# Event Loop

# Task Queue

# Micro-task Queue

# JavaScript Engine

## Call Stack

Executes synchronous code. Breaks tasks into functions calls. For every function, its created a stack frame. Functions can also create Promises, pushing it into the Promise API.

### Execution Context

A.k.a Stack Frame, Call Stack creates an Execution Context in order to execute the received function. Alongside with the function code, the Execution Context will also have:
- The value of "this"
- The function code
- Arguments Value
- Variable Mapping

In order to achieve the variable mapping, the Execution Context needs to access the heap memory to find the actual values of those variables.

Both the global context and the local context get injected to the execution context when it's loaded.

## Heap

## Event Listeners

## Timer API

Pushes the timer callback to Macro-task Queue

## Promise API

Pushes the promise callback to Micro-task Queue