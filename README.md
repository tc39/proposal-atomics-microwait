# Micro waits in JS

Stage: 2.7

Author: Shu-yu Guo

Champion: Shu-yu Guo

## Motivation
Efficient implementation of the locks use a loop body of the following shape for acquiring the lock:

```javascript
// Fast path
let spins = 0;
do {
  if (TryLock()) {
    // Lock acquired.
    return;
  }

  SpinForALittleBit();
  spins++;
} while (spins < kSpinCount);

// Slow path
PutThreadToSleepUntilLockReleased();
```

This algorithm is both fast when contention is low and efficient when contention is high. When contention is low (i.e. the likelihood that the lock is released after a very short amount of time is high), spinning for a little bit improves the performance because the code does not re-enter the kernel to yield or to sleep. When contention is high (i.e. the likelihood that the lock is released after a very short amount of time is low), going back to the kernel to put the executing thread to sleep improves efficiency.

`SpinForALittleBit()` is impossible to optimally write in JS, as the optimal version often requires hinting the CPU to allow sibling cores to access shared resources. On x86, for example, the [Intel optimization manual](https://www.intel.com/content/www/us/en/content-details/671488/intel-64-and-ia-32-architectures-optimization-reference-manual-volume-1.html) recommends a loop with the `pause` instruction and exponential backoff. Versions without this hinting suffer performance and scheduling issues.

`PutThreadToSleepUntilLockReleased()` is impossible to optimally write on the main thread, as blocking is disallowed on the main thread as a policy choice to prevent deadlocks and hangs in the browser UI. **This proposal does not seek to solve this use case**.

### Use case: Emscripten

Emscripten uses a [busy loop](https://github.com/emscripten-core/emscripten/blob/bc5998833dcd0f48e90a8cb13fdf40e36480e4cb/system/lib/pthread/emscripten_futex_wait.c#L20-L112) to emulate a blocking wait in its implementation of futexes on the main thread, which are in turn used to implement mutexes.

Allowing microwaits will improve the power and scheduling efficiency of multithreaded applications compiled using Emscripten.

## Proposal

I propose one new method on `Atomics`.

### `Atomics.pause`

For better spinning, add `Atomics.pause(iterationNumber)`. It performs a finite-time wait for a very short time that runtimes can implement with the appropriate CPU hinting. It has no observable behavior other than timing.

Unlike `Atomics.wait`, since it does not block, it can be called from both the main thread and worker threads.

Implementations are expected to implement a short spin with CPU yielding, using best practices for the underlying architecture. The non-negative integer argument `iterationNumber` is a hint to implement a backoff algorithm if the microwait itself is in a loop. It defaults to `0`. `Atomics.pause(n)` waits at most as long as `Atomics.pause(n+1)`.

## Prior discussions and acknowledgements

Microwaits have been discussed previously when SharedArrayBuffers were proposed. See https://github.com/tc39/proposal-ecmascript-sharedmem/issues/87. In my opinion, the arguments on that thread still hold today. `Atomics.pause` is basically exactly the same as Lars's previous design.

Thread yields and efficient spin loops have also been discussed in the context of WebAssembly. See https://github.com/WebAssembly/threads/issues/15.

## FAQ

### Does `Atomics.pause()` yield execution to another thread?

No. Microwaiting yields shared resources in a CPU without giving up occupancy of the core itself. Thread yielding is done at the OS level instead of the CPU level.

### Why can't I block the main thread?

Blocking the main thread is bad because it has catastrophic effects on responsiveness and performance of web pages.

### I still want to block the main thread, can we clamp timeouts on `Atomics.wait`?

Initially, this proposal included an overload of `Atomics.wait` that allowed the timeout value to be clamped to an implementation-defined limit on the main thread. The idea was that if the blocking periods were short enough and somehow lined up with what implementations considered "idle periods", then the policy choice of disallowing the main thread from being blocked would not be violated.

This overload was removed from the proposal for the following reasons:

- Not enough bang for the buck. There is no consensus to allow indefinite blocking on the main thread, and bounded-time blocking is of limited value for the use case.
- It is difficult to specify _how_ to clamp the timeout in a web embedding. The previous idea was to tie it to the current "idle period", which is sensitive to scheduled tasks and the presence of event handlers responding to input, etc. However, this is mechanically complicated. Moreover, there is likely to be _no_ free idle periods in an application, which suggests we may need both a floor and a ceiling for the clamping. This further raises the specification challenge for the small gain.
