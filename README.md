# Node.js Internals: Architecture and Execution

This document explains the underlying mechanics that allow Node.js to handle asynchronous tasks and perform high-performance execution.

---

## 1. Node.js Architecture
Node.js is a runtime environment that allows JavaScript to run on the server side. Its architecture is structured in layers that bridge the gap between high-level JavaScript and low-level system operations.



### JavaScript Engine (V8)
Developed by Google for Chrome, **V8** is the core of Node.js. It compiles JavaScript code directly into **Machine Code** using Just-In-Time (JIT) compilation. This removes the need for line-by-line interpretation, making execution extremely fast.

### Node.js Core APIs
These are built-in modules (such as `fs`, `http`, `path`, and `crypto`) that provide essential functionality. While the interfaces are in JavaScript, the actual work is often handled by underlying C++ logic.

### Native Bindings
Since V8 is written in C++ and our application code is in JavaScript, they cannot communicate directly. **Native Bindings** act as the "glue" or translator, allowing JavaScript to trigger low-level C++ functions for system tasks.

---

## 2. libuv

### What is libuv?
**libuv** is a multi-platform C library that focuses on **asynchronous I/O**. If V8 is the brain of Node.js, libuv is the nervous system.

### Why Node.js needs libuv
JavaScript is single-threaded. Without libuv, tasks like reading a large file or waiting for a network request would freeze the entire application. libuv handles these "waiting" tasks in the background, keeping the main thread free.

### Responsibilities of libuv
* **Event Loop:** Managing the execution of asynchronous callbacks.
* **Thread Pool:** Handling tasks that are too heavy for the main thread.
* **System Tasks:** Managing File System I/O, DNS operations, and child processes.

---

## 3. Thread Pool

### What is a thread pool?
The **Thread Pool** is a collection of extra threads (default is 4) maintained by libuv. While the main thread handles the primary execution, the pool handles the "heavy manual labor" in the background.

### Why Node.js uses a thread pool
Node.js offloads operations that are inherently blocking or computationally expensive to the thread pool. This prevents the "Single Thread" from stuttering or stopping.

### Operations handled by the thread pool
* **File I/O:** Reading or writing files via the `fs` module.
* **Cryptography:** Heavy math operations like `crypto.pbkdf2`.
* **Compression:** Tasks using the `zlib` module.
* **DNS Lookups:** Resolving domain names to IP addresses.

---

## 4. Worker Threads

### What are worker threads?
**Worker Threads** allow developers to run multiple threads of JavaScript in parallel. Unlike the Thread Pool (managed internally by libuv), Worker Threads are managed manually by the developer.

### Why are worker threads needed?
They are used for **CPU-intensive tasks** (like video processing or data mining). Running these on a Worker Thread prevents the main Event Loop from being blocked.

### Difference: Thread Pool vs. Worker Threads

| Feature | Thread Pool (libuv) | Worker Threads |
| :--- | :--- | :--- |
| **Control** | Automatic / Internal | Manual / Developer-controlled |
| **Purpose** | Offloading I/O and specific APIs | Handling CPU-intensive JS logic |
| **Language** | Runs C++ code | Runs JavaScript code |

---

## 5. Event Loop Queues
The Event Loop sorts tasks into queues based on priority to determine the order of execution.



### Micro Task Queue
This is the **highest priority** queue. It must be completely emptied before the Event Loop moves to the next macro task.
* **Examples:** `process.nextTick` (highest priority), Promises (`.then`, `.catch`, `async/await`).

### Macro Task Queue (Task Queue)
This queue handles "larger" external events.
* **Examples:** `setTimeout`, `setInterval`, I/O callbacks, `setImmediate`.

### Execution Priority
1.  **The Call Stack:** Executes all synchronous code first.
2.  **Micro Task Queue:** Once the stack is empty, Node clears all microtasks.
3.  **Macro Task Queue:** The loop picks one task from the macro queue, executes it, and then checks the Micro Task Queue again.
