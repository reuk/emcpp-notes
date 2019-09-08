# The Concurrency API

## Prefer task-based programming to thread-based

If you want to run some work asynchronously, you have two basic choices, `thread` and `async`:

```
auto doWork = [] { /* ... */ }; // some async task
std::thread t { doWork };
auto fut = std::async (doWork);
```

The `async` approach is often better, for a few reasons:
- It provides a way to `get` the result computed by the task (if we care about that)
- It captures exceptions thrown by the task and re-emits them on the call to `get`, whereas
  exceptions thrown when using `thread` will `terminate` the program.
- It offloads some correctness/performance details to the stdlib implementation.

The OS only has a limited number of threads it can provide. If more `thread`s are requested than the
system can provide, a `system_error` exception will be thrown, even if the task itself is
`noexcept`.

If you create more software threads than there are available hardware threads, the OS will be
forced to time-slice the threads, introducing potentially costly context switches. Avoiding this
is difficult because different systems will have different hardware and workloads.

Using `std::async` allows the system to schedule work using a system-wide thread pool, improving
load balancing across hardware threads (although this is not required). Using `thread` directly will
require you to deal with thread exhaustion and load balancing yourself, and your solution may not
interact well with similar systems in other software.

There are a few situations in which using `thread` directly makes sense:
- Interacting directly with the system threading API to set thread priorities/affinities via
  `native_handle`.
- You know that your program will be the only significant process running on a machine, and can
  optimise thread usage accordingly.
- You need to implement an advanced construct such as a thread pool on a system that doesn't already
  provide it.

### Guidelines

- `async` allows accessing the result of an asynchronous task, and won't terminate your program if
  an exception is thrown, so it should generally be preferred over `thread`.
- Using `thread` directly requires consideration of thread exhaustion, oversubscription, load
  balancing, and portability, so make sure you've considered all these aspects (and alternative
  solutions) before using it.

## Specify `std::launch::async` if asynchronicity is essential

There are two launch policies:
- `async` means the task will run on a different thread
- `deferred` means the task will run synchronously when `get` is called on the future.

If no launch policy is provided, the implementation is free to choose whether to run the task
`async` or `deferred`. This has some interesting implications: We don't know when and where the task
will run, or even if it will run at all (if `get` isn't called on some paths).

Using the default launch policy is only acceptable under the following circumstances:
- The task doesn't need to run concurrently with the thread calling `get/wait`.
- It doesn't matter which thread's `thread_local` storage is accessed.
- Failure to run the task is acceptable *or* `get/wait` are guaranteed to be called on the future.
- Code using `wait_for/wait_until` takes deferred futures into account:
  - Using `future.wait_for(foo) == std::future_status::ready` as a loop predicate might never
    terminate if the task is `deferred`, because `wait_for` will always return
    `future_status::deferred`.

    To check whether a future has a deferred task, you can check the result of `wait_for(0s)`, and
    then take different actions depending on whether the task is running asynchronously or not.

If any of those conditions cannot be met, you should explicitly request the `async` launch policy.

You can fairly trivially write a `reallyAsync` wrapper for `std::async`:
```
template <typename... Ts>
auto reallyAsync (Ts&&... ts) {
  return std::async (std::launch::async, std::forward<Ts> (ts)...);
}
```

### Guidelines

- Exercise caution when using the default launch policy with `std::async` and understand the
  potential failure cases (`thread_local` storage access, timeout-based looping, failure to run the
  task altogether).
- Explicitly request the `async` launch policy when necessary (or use a wrapper function).

## Make `std::thread`s unjoinable on all paths

`thread` objects can be in either 'joinable' or 'unjoinable' state.

A 'joinable' thread is a thread that is (or is waiting to be) running.

An 'unjoinable' thread might be:
- A default-constructed `std::thread` (these don't represent a thread of execution)
- A moved-from `std::thread` (ownership of the thread of execution is transferred away)
- A `std::thread` that has been `join`ed (the thread of execution has finished running)
- A `std::thread` that has been `detach`ed (the thread of execution is no longer owned by any
  `thread` object)

If the destructor of a `thread` in a joinable state is invoked, the program will be terminated. This
sounds bad, but the alternatives are worse:
- An implicit join would wait in its destructor, which might lead to awkward performance anomalies.
- An implicit detach would cause the thread to live on indefinitely. If the thread captured and
  modified references to stack variables, this could cause very subtle and difficult-to-find bugs.

A sudden program termination should be easier to debug than either of the other two cases.

To guarantee that a `thread` is made unjoinable on every path, you can write a straightforward RAII
wrapper for `thread` that joins/detaches in its destructor. Be careful using this though, for the
reasons mentioned earlier.

There's no direct support in the C++ stdlib for 'interruptible threads', but they can be implemented
by hand using the provided primitives.

### Guidelines

- Make `std::thread` objects unjoinable on all paths, ideally using RAII, but be aware that:
  - join-on-destruct may have performance implications
  - detach-on-destruct may have UB/lifetime implications

## Be aware of varying thread handle destructor behaviour

The result of a `std::async` call cannot be stored in an object associated with the callee, as the
callee might finish before `get` is called on the future. In this scenario, objects local to the
callee would be destroyed before they could be read.

Similarly, the result cannot be stored directly in the associated future, because this object might
be `move`d or converted to a `shared_future`, so there wouldn't be a constant target address to
which to write the result.

Instead, there's typically a shared object stored on the heap known as the 'Shared State' which
is referenced by both the `promise` that sends the result and the `future` which receives it.

The behaviour of a `future`'s destructor is determined by this shared state:
- The destructor for the last future referring to a shared state for a non-`deferred` task launched
  by `std::async` blocks until the task completes. This is akin to `join`ing the associated thread.
- The destructor for all other futures simply destroys the future object but leaves the shared state
  intact. For async tasks (not created by `std::async`) this is akin to `detach`ing the associated
  thread. For deferred tasks, the task will not run.

That is, `async` futures launched by `std::async` are a special case with different behaviour. This
is important if creating `future`s using other mechanisms, like `promise` or `packaged_task`.

### Guidelines

- When manipulating/storing `std::future` objects, try to ensure that they are created in a
  consistent way, so that their destructors don't block and cause unexpected performance issues.
  - Future destructors normally just destroy the future's data members.
  - The final future referring to a shared state for a non-deferred task launched via `std::async`
    blocks until the task completes.

## Consider `void` futures for one-shot event communication

There are a few ways to signal events between threads. One option might be to use a
`std::condition_variable`:

```
// shared state
std::condition_variable cv;
std::mutex m;

// event source
cv.notify_one();

// event receiver
{
  std::unique_lock lk { m }; // unique_lock supports CTAD
  cv.wait (lk);
}
```

This approach has some issues:
- A mutex's job is to synchronise accesses to shared state, but there's no shared state to sync in
  this scenario.
- If the detecting task notifies the condvar before the other task calls `cv.wait`, that task could
  hang indefinitely.
- The waiting code has no way to determine whether or not a wakeup that occurs is spurious (without
  sharing additional state).

Another option is to use a boolean flag, set it from the signalling thread, and poll it from
the waiting thread, but the polling is very wasteful.

The two approaches can be combined (i.e. set a bool flag and then notify the condvar, check the bool
flag after waking) but that's a lot of boilerplatey code that's awkward to reuse cleanly.

A much cleaner approach is to to use a `std::promise<void>`:

```
// shared state
std::promise<void> p;

// event source
p.set_value();

// event receiver
p.get_future().wait();
```

This approach is more concise, will work even if the `set_value` is called before `wait`, and is
immune to spurious wakeups. However, it does require a heap allocation for the shared state, and the
system can only be used to signal a single event.

When combined with shared futures, this pattern can be used to send notifications to multiple
threads at once:

```
std::promise<void> p;
auto sf = p.get_future().share();

std::vector<std::thread> vt;

for (auto i = 0; i != 2; ++i)
  vt.emplace_back ([sf] { sf.wait(); });

p.set_value(); // all threads waiting on the shared future will be notified

for (auto& t : vt)
  t.join();
```

### Guidelines

- Prefer promises and futures over condvar/flag-based designs when the overhead of a heap allocation
  is acceptable and only single-shot communication is required.

## Use `std::atomic` for concurrency, `volatile` for special memory

`atomic` and `volatile` have different meanings. Operations on `atomic<T>` objects are guaranteed
to seen as atomic from other threads. `volatile` does *not* give this guarantee. Multiple
simultaneous readers/writers of `volatile` storage is still a data race.

Using `atomic`s (with default settings) also ensures sequential consistency, whereas `volatile`
does not, so the optimiser might do surprising things with code that just uses `volatile`.

So what is `volatile` actually good for? In normal code, the optimiser is allowed to remove
redundant loads and dead stores (series of non-interleaved reads or writes to an address) because
such code wouldn't have any useful side-effects. However, for some types of special memory (e.g.
memory-mapped IO), addresses might be read/written by peripherals, so these chains of reads/writes
would no longer be redundant: reading the same address twice in a row might yield different
results, and setting the same address twice in a row might adjust settings on some peripheral.

`volatile` just tells the compiler not to remove 'redundant' loads and stores.

### Guidelines

- You're unlikely to need `volatile`:
  - `std::atomic` is for data accessed by multiple threads.
  - `volatile` is for ensuring that reads/writes are not optimised away.
