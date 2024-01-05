# Micro and mini waits in JS

Stage: 0

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

`PutThreadToSleepUntilLockReleased()` is impossible to optimally write on the main thread, as blocking is disallowed on the main thread as a policy choice to prevent deadlocks and hangs in the browser UI.

### Use case: Emscripten

Emscripten uses a [busy loop](https://github.com/emscripten-core/emscripten/blob/bc5998833dcd0f48e90a8cb13fdf40e36480e4cb/system/lib/pthread/emscripten_futex_wait.c#L20-L112) to emulate a blocking wait in its implementation of futexes on the main thread, which are in turn used to implement mutexes.

Allowing microwaits will improve the power and scheduling efficiency of multithreaded applications compiled using Emscripten.

## Proposal

I propose one new method on `Atomics` and a new overload of `Atomics.wait`.

### `Atomics.microwait`

For better spinning, add `Atomics.microwait(iterationNumber)`. It performs a finite-time wait for a very short time that runtimes can implement with the appropriate CPU hinting. It has no observable behavior other than timing.

Unlike `Atomics.wait`, since it does not block, it can be called from both the main thread and worker threads.

Implementations are expected to implement a short spin with CPU yielding, using best practices for the underlying architecture. The non-negative integer argument `iterationNumber` is a hint to implement a backoff algorithm if the microwait itself is in a loop. It defaults to `0`. `Atomics.microwait(n)` waits at most as long as `Atomics.microwait(n+1)`.

### `clampTimeoutIfCannotBlock`

For sleeping on the main thread, add the `Atomics.wait(ta, index, value, timeout, { clampTimeoutIfCannotBlock: true }` overload. This overload clamps the timeout to an implementation-defined value if the executing thread cannot block.

`Atomics.wait` with the `clampTimeoutIfCannotBlock` option set to `true` can be called on the main thread.

The option has no effect on worker threads because they can block.

### Open questions

- How should the maximum timeout value be determined on the main thread?
- Should the maximum timeout value be static or dynamic (e.g., time left until the next animation frame)?

## Prior discussions and acknowledgements

Microwaits have been discussed previously when SharedArrayBuffers were proposed. See https://github.com/tc39/proposal-ecmascript-sharedmem/issues/87. In my opinion, the arguments on that thread still hold today. `Atomics.microwait` is basically exactly the same as Lars's previous design.

Thread yields and efficient spin loops have also been discussed in the context of WebAssembly. See https://github.com/WebAssembly/threads/issues/15.

## FAQ

### Does `Atomics.microwait()` yield execution to another thread?

No. Microwaiting yields shared resources in a CPU without giving up occupancy of the core itself. Thread yielding is done at the OS level instead of the CPU level.

### Isn't blocking the main thread bad? Why is this a clamped timeout okay?

Yes. Blocking the main thread is bad because it has catastrophic effects on responsiveness and performance of web pages. However, blocking is still possible to emulate today by a naive spinlock, and a naive spinlock is much worse for power consumption and scheduling than a very small timed-out wait.

### Will this proposal let me write symmetric code for the main thread and workers?

No, except in very narrow cases. Indefinite blocks remain disallowed on the main thread. The main thread on the web platform remains special, and requires special handling, especially to [avoid deadlock](https://github.com/tc39/proposal-ecmascript-sharedmem/issues/100#issuecomment-220052785).

### Isn't this two separate proposals?

Sure. This can be split into two independent proposals with no dependencies on each other.
