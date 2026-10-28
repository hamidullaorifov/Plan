# 3. Async, Await and Concurrency

> Phase 1 · Weeks 3–4 · [← Phase index](README.md)
> Week 3 covers sections 3.1–3.8 and week 4 covers 3.9–3.16.

**Why this matters:** Async bugs are among the most expensive production incidents: deadlocks, thread-pool starvation, lost exceptions, race conditions and requests that never get cancelled. They're also the most popular topic in Middle-level interviews. Expect to be asked: *"What happens when `await` reaches an incomplete `Task`?"*

---

## 3.1 Vocabulary: concurrency, parallelism and asynchrony

| Term | Meaning | Example |
|---|---|---|
| **Concurrency** | Several tasks are *in progress* at the same time (possibly interleaved on one core) | A web server handling 1,000 requests |
| **Parallelism** | Several tasks *execute* at the same instant on several cores | `Parallel.For` over an image's pixels |
| **Asynchrony** | Starting an operation and **not blocking a thread** while it's in flight | `await httpClient.GetAsync(...)` |

| Work type | Bottleneck | Correct tool |
|---|---|---|
| **I/O-bound** (HTTP, database, files, queues) | waiting on a device or the network | `async`/`await`, which releases the thread while waiting |
| **CPU-bound** (hashing, image processing, big calculations) | CPU cycles | `Parallel.*`, PLINQ, or `Task.Run` to offload from a UI thread |

> **Key insight:** async doesn't make a single operation faster. It makes the *system* more **scalable**, because threads aren't parked doing nothing during I/O. A server can handle 10,000 concurrent requests with a few dozen threads.

### The thread pool

.NET keeps a pool of worker threads that run `Task` work items and async continuations. It starts with a minimum number of threads (roughly the core count) and **adds threads slowly** when work queues up. The hill-climbing algorithm injects roughly one or two threads per second. That slow injection is why blocking pool threads is so harmful (see section 3.7).

---

## 3.2 `Task` and `Task<T>`

A `Task` is a **promise** of a future result (or failure, or cancellation). It isn't a thread.

| State | Meaning |
|---|---|
| `WaitingForActivation` / `Running` | in progress |
| `RanToCompletion` | success; `Result` is available |
| `Faulted` | failed; the exception is stored in `task.Exception` (an `AggregateException`) |
| `Canceled` | canceled through a `CancellationToken` |

Tasks returned by `async` methods are **hot**: they're already running when you get them.

---

## 3.3 How `async`/`await` really works

```csharp
public async Task<int> GetOrderCountAsync(Guid customerId, CancellationToken ct)
{
    Console.WriteLine("A: runs synchronously on the caller's thread");
    var orders = await _repository.GetOrdersAsync(customerId, ct);   // ← suspension point
    Console.WriteLine("B: runs as a continuation, later, possibly on another thread");
    return orders.Count;
}
```

Step by step:

1. The compiler rewrites the method into a **state machine**: a struct in Release builds that implements `IAsyncStateMachine`. Each `await` becomes a numbered state.
2. Calling the method runs it **synchronously** up to the first `await`.
3. At `await x`, the method calls `x.GetAwaiter()` and checks **`IsCompleted`**:
   - **Already complete** (for example, the data was cached): it continues synchronously with no thread switch and no allocation. This is the "fast path".
   - **Not complete:** the state machine stores its locals, **registers a continuation** with `awaiter.OnCompleted(...)`, and **returns** an incomplete `Task` to the caller. The thread is now free. On this slow path, the state machine is boxed onto the heap once.
4. When the I/O finishes, the continuation is **scheduled**:
   - on the captured `SynchronizationContext` if there is one (WPF/WinForms UI thread, classic ASP.NET),
   - otherwise on the **thread pool**. ASP.NET Core has **no** `SynchronizationContext`.
5. The continuation calls `awaiter.GetResult()`. That returns the value or **rethrows the exception** (the first inner exception, not an `AggregateException`), and execution resumes after the `await`.
6. When the method returns, the `Task` is completed with the result, and *its* awaiters' continuations run in turn.

Simplified, the generated code looks like this:

```csharp
// Pseudo-code (much simplified) of the compiler-generated MoveNext()
void MoveNext()
{
    try
    {
        if (_state == 0) goto Resume0;
        Console.WriteLine("A");
        _awaiter = _repository.GetOrdersAsync(_customerId, _ct).GetAwaiter();
        if (!_awaiter.IsCompleted)
        {
            _state = 0;
            _builder.AwaitUnsafeOnCompleted(ref _awaiter, ref this); // register the continuation, then return
            return;
        }
    Resume0:
        var orders = _awaiter.GetResult();   // returns the value or rethrows
        Console.WriteLine("B");
        _builder.SetResult(orders.Count);
    }
    catch (Exception ex) { _builder.SetException(ex); }
}
```

> **Try it:** paste an async method into SharpLab and choose the *C#* output to see the real generated state machine.

### Runtime Async (.NET 11, preview)

.NET 11 adds **Runtime Async**. Instead of the compiler emitting a state machine, the **runtime itself** suspends and resumes methods. To opt in, add `<Features>runtime-async=on</Features>` to the project file. The .NET 11 libraries are already compiled this way.

What it changes:

- **Cleaner live stack traces.** You see the real call chain (Outer → Middle → Inner) without `AsyncMethodBuilderCore.Start` frames.
- Breakpoints and stepping work better in the debugger.
- There's less overhead and fewer allocations.

**Your code's semantics don't change.** `await`, exceptions, `SynchronizationContext` and `ConfigureAwait` all behave the same, so everything in this guide still applies.

### `ExecutionContext` and `AsyncLocal<T>`

Ambient data (`AsyncLocal<T>`, the `Activity` used for tracing, culture) **flows down** into awaited calls automatically. Changes made inside a callee do **not** flow back up to the caller.

---

## 3.4 `Task` vs `ValueTask`

| | `Task<T>` | `ValueTask<T>` |
|---|---|---|
| Allocation | a heap object (results like `true`/`false`, small ints and completed tasks are cached) | **none** when the result is available synchronously |
| Can be awaited many times | ✅ | ❌ **only once** |
| Can be awaited concurrently | ✅ | ❌ |
| `.Result` before completion | blocks (bad, but defined) | ❌ undefined behavior |
| Use for | **the default, everywhere** | hot paths that *usually* complete synchronously (cache hits, buffered reads) |

```csharp
public ValueTask<Product?> GetProductAsync(int id, CancellationToken ct)
{
    if (_cache.TryGetValue(id, out var product))
        return ValueTask.FromResult<Product?>(product);    // sync path: no allocation
    return new ValueTask<Product?>(LoadAndCacheAsync(id, ct)); // async path wraps a Task
}
```

If you need to await a `ValueTask` twice or store it, call `.AsTask()` once.

---

## 3.5 `ConfigureAwait(false)`

`await task.ConfigureAwait(false)` means *"I don't need to resume on the captured context."*

| Where | Guidance |
|---|---|
| **ASP.NET Core application code** | Not needed, because there's no `SynchronizationContext`. Leaving it out is fine. |
| **Reusable libraries (NuGet packages, shared libraries)** | **Use it on every await.** The library might be called from a UI app, where resuming on the UI thread costs time and can deadlock. |
| **UI code that touches UI controls after the await** | **Don't** use it. You need to be back on the UI thread. |

.NET 8 added `ConfigureAwaitOptions`. For example, `await task.ConfigureAwait(ConfigureAwaitOptions.SuppressThrowing)` waits for completion without throwing, which is useful for "fire, then observe completion" scenarios.

---

## 3.6 The classic deadlock: sync-over-async

```csharp
// WinForms / WPF button handler (or classic ASP.NET):
private void LoadButton_Click(object sender, EventArgs e)
{
    var data = LoadAsync().Result;   // ❌ blocks the UI thread
    label.Text = data;
}

private async Task<string> LoadAsync()
{
    await Task.Delay(100);           // captures the UI SynchronizationContext
    return "done";                   // this continuation must run on the UI thread, which is blocked → DEADLOCK
}
```

Here's the cycle: the UI thread waits for the task → the task waits for its continuation → the continuation waits for the UI thread.

**Fixes, best first:**

1. **Async all the way:** `private async void LoadButton_Click(...) { label.Text = await LoadAsync(); }`. `async void` is acceptable *only* for event handlers.
2. Use `ConfigureAwait(false)` inside `LoadAsync`. This hides the problem but doesn't fix the design.
3. Never call `.Result`, `.Wait()` or `.GetAwaiter().GetResult()` on tasks that aren't complete.

You'll reproduce this deadlock in a console app in [Lab 3](09-labs-and-capstone.md#lab-3--reproduce-and-fix-a-sync-over-async-deadlock).

---

## 3.7 Thread-pool starvation, the ASP.NET Core version

ASP.NET Core has no `SynchronizationContext`, so the classic deadlock *doesn't* happen. Sync-over-async still hurts, though:

1. A request calls `.Result`, which **blocks a thread-pool thread** while it waits for I/O.
2. Under load, every pool thread ends up blocked.
3. The I/O completes, but its continuation needs a pool thread to run, and none is free.
4. The pool injects new threads only **slowly**, so latency explodes and health checks time out. The app looks frozen even though the CPU is idle.

**Symptoms:** high latency with *low* CPU, a growing `ThreadPool Queue Length`, and a thread count that climbs slowly in `dotnet-counters`.

**Causes:** `.Result`, `.Wait()`, `Thread.Sleep`, synchronous I/O (`File.ReadAllText`, `stream.Read`) or a synchronous database driver on request paths.

**Fix:** go async all the way. Since .NET 6 the pool injects threads faster when it detects blocking on tasks. That mitigates starvation but doesn't fix it.

You'll practice this in [Lab 4](09-labs-and-capstone.md#lab-4--thread-pool-starvation).

---

## 3.8 `async void`: only for event handlers

| | `async Task` | `async void` |
|---|---|---|
| Can be awaited | ✅ | ❌ |
| Exceptions | stored in the Task | **raised directly on the thread pool or context, which crashes the process** |
| Testable | ✅ | ❌ |

**A hidden `async void`:** an async lambda passed to a parameter of type `Action`:

```csharp
items.ForEach(async item => await SaveAsync(item));        // ❌ List<T>.ForEach takes Action<T> → async void
Parallel.ForEach(items, async item => await SaveAsync(item)); // ❌ same: fire-and-forget, exceptions crash

foreach (var item in items) await SaveAsync(item);         // ✅ sequential
await Parallel.ForEachAsync(items, ct, async (item, token) => await SaveAsync(item, token)); // ✅ parallel
```

---

## 3.9 Cancellation

**Rule:** every async method that does I/O should accept a `CancellationToken` and pass it on to every call it makes.

```csharp
public async Task<Report> BuildReportAsync(Guid id, CancellationToken ct)
{
    var data = await _repository.LoadAsync(id, ct);          // pass it down
    foreach (var chunk in data.Chunk(1_000))
    {
        ct.ThrowIfCancellationRequested();                   // check in CPU-bound loops
        Process(chunk);
    }
    return await _renderer.RenderAsync(data, ct);
}
```

```csharp
// Timeout plus a caller's token, combined
using var timeoutCts = CancellationTokenSource.CreateLinkedTokenSource(callerToken);
timeoutCts.CancelAfter(TimeSpan.FromSeconds(5));

try
{
    await client.GetStringAsync(url, timeoutCts.Token);
}
catch (OperationCanceledException) when (!callerToken.IsCancellationRequested)
{
    throw new TimeoutException("Upstream call took longer than 5 s.");
}
```

**Practical points:**

- In ASP.NET Core, add a `CancellationToken` parameter to an endpoint or action and it's bound to `HttpContext.RequestAborted` automatically. It fires when the client disconnects.
- Cancellation surfaces as an `OperationCanceledException` (`TaskCanceledException` derives from it). Treat it as expected, not as an error, and don't log it as one.
- `CancellationTokenSource` is `IDisposable`, especially when you use `CancelAfter`.
- `task.WaitAsync(timeout, ct)` (.NET 6+) stops **waiting**, but it **doesn't stop** the underlying operation.

---

## 3.10 Composing tasks

```csharp
// Run independent I/O in parallel
Task<Customer> customerTask = _customers.GetAsync(id, ct);
Task<IReadOnlyList<Order>> ordersTask = _orders.GetForCustomerAsync(id, ct);
await Task.WhenAll(customerTask, ordersTask);
var dto = new CustomerSummary(customerTask.Result, ordersTask.Result); // safe: both are complete
```

| API | Behavior | Watch out for |
|---|---|---|
| `Task.WhenAll` | completes when **all** tasks complete | `await` rethrows only the **first** exception. Inspect `whenAllTask.Exception.InnerExceptions` to see them all |
| `Task.WhenAny` | completes when **any** task completes | the other tasks keep running, so cancel them yourself |
| `Task.WhenEach` (.NET 9) | an `IAsyncEnumerable` of tasks in completion order | `await foreach (var t in Task.WhenEach(tasks))` |
| `Parallel.ForEachAsync` (.NET 6) | bounded parallel async loop | set `MaxDegreeOfParallelism` |

**Throttling (don't send 10,000 requests at once):**

```csharp
await Parallel.ForEachAsync(
    productIds,
    new ParallelOptions { MaxDegreeOfParallelism = 8, CancellationToken = ct },
    async (id, token) => await _priceClient.RefreshAsync(id, token));
```

---

## 3.11 Fire-and-forget: avoid it

```csharp
_ = SendWelcomeEmailAsync(user); // ❌ exceptions are lost, and it may outlive the request
```

Why it's a problem:

- Exceptions go unobserved and vanish silently.
- In ASP.NET Core, scoped services (such as `DbContext`) are disposed when the request ends, and your background code then crashes with `ObjectDisposedException`.
- The app can shut down in the middle of the operation.

**Do this instead:** put the work on a queue (a `Channel<T>`) that a `BackgroundService` consumes, as shown in section 3.13. For durable work, use a message broker or an outbox (Phase 3).

---

## 3.12 CPU-bound work

```csharp
// Desktop/UI: keep the UI responsive
var hash = await Task.Run(() => ComputeHash(bigFile));

// Data-parallel loops
Parallel.For(0, pixels.Length, i => pixels[i] = Transform(pixels[i]));

// PLINQ
var primes = numbers.AsParallel().Where(IsPrime).ToArray();
```

> In **ASP.NET Core**, wrapping work in `Task.Run` gains nothing. You just move it from one pool thread to another and add overhead. Run CPU-bound request work directly. If it's heavy, move it to a background worker or queue.

---

## 3.13 Async streams and `Channel<T>`

### `IAsyncEnumerable<T>`: stream results as they arrive

```csharp
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    int page = 0;
    while (true)
    {
        var batch = await _api.GetPageAsync(page++, ct);
        if (batch.Count == 0) yield break;
        foreach (var order in batch) yield return order;
    }
}

await foreach (var order in StreamOrdersAsync(ct))
    Console.WriteLine(order.Id);
```

### `Channel<T>`: an async producer/consumer queue

```csharp
using System.Threading.Channels;

public sealed class EmailQueue
{
    private readonly Channel<EmailMessage> _channel = Channel.CreateBounded<EmailMessage>(
        new BoundedChannelOptions(capacity: 1_000) { FullMode = BoundedChannelFullMode.Wait });

    public ValueTask EnqueueAsync(EmailMessage message, CancellationToken ct) =>
        _channel.Writer.WriteAsync(message, ct);              // waits (back-pressure) when the queue is full

    public IAsyncEnumerable<EmailMessage> ReadAllAsync(CancellationToken ct) =>
        _channel.Reader.ReadAllAsync(ct);
}
```

The consuming `BackgroundService` is shown in [guide 6, section 6.6](06-generic-host-di-config-logging.md#66-background-services).

- **Bounded** channels give you back-pressure and protect memory. Prefer them.
- **Unbounded** channels can grow until the process runs out of memory.

---

## 3.14 Shared state and synchronization

### A race condition

```csharp
int counter = 0;
Parallel.For(0, 100_000, _ => counter++);  // read-modify-write is not atomic
Console.WriteLine(counter);                // < 100000, and different on every run
```

### Choosing a primitive

| Need | Use | Notes |
|---|---|---|
| Atomic counter or flag | `Interlocked.Increment/Add/Exchange/CompareExchange` | lock-free and fastest |
| A short critical section (sync code) | `lock` on a `System.Threading.Lock` (.NET 9+) | **you can't `await` inside a `lock`** (CS1996) |
| A critical section containing `await` | `SemaphoreSlim(1, 1)` with `WaitAsync` | always `Release` in `finally` |
| Limit concurrency to N | `SemaphoreSlim(N, N)` or `Parallel.ForEachAsync` | throttles outgoing calls |
| Many readers, few writers (sync) | `ReaderWriterLockSlim` | not async-friendly |
| A thread-safe dictionary or queue | `ConcurrentDictionary`, `ConcurrentQueue` | individual operations are atomic; *compound* operations aren't |
| Snapshot-style shared data | immutable collections plus `Interlocked.Exchange` | readers never lock |

```csharp
public sealed class Inventory
{
    private readonly Lock _lock = new();
    private readonly Dictionary<string, int> _stock = new();

    public bool TryReserve(string sku, int quantity)
    {
        lock (_lock)                                // a compound check-then-act must be atomic
        {
            if (_stock.GetValueOrDefault(sku) < quantity) return false;
            _stock[sku] -= quantity;
            return true;
        }
    }
}

public sealed class TokenProvider(IAuthClient client)
{
    private readonly SemaphoreSlim _gate = new(1, 1);
    private string? _token;
    private DateTimeOffset _expiresAt;

    public async Task<string> GetTokenAsync(CancellationToken ct)
    {
        if (_token is not null && DateTimeOffset.UtcNow < _expiresAt) return _token;

        await _gate.WaitAsync(ct);
        try
        {
            if (_token is null || DateTimeOffset.UtcNow >= _expiresAt)   // double-check inside the gate
            {
                var result = await client.RequestTokenAsync(ct);
                (_token, _expiresAt) = (result.AccessToken, result.ExpiresAt.AddMinutes(-1));
            }
            return _token;
        }
        finally
        {
            _gate.Release();
        }
    }
}
```

> **`TokenProvider` is simplified for teaching.** Reading `_token` and `_expiresAt` outside the gate can tear on some platforms. Production code would store them in one immutable record and swap it atomically.

### `ConcurrentDictionary` gotcha

```csharp
// The factory can run MORE THAN ONCE under contention; only one result is kept
var client = _clients.GetOrAdd(tenantId, id => CreateExpensiveClient(id));

// Guarantee a single execution with Lazy<T>
private readonly ConcurrentDictionary<string, Lazy<ApiClient>> _lazyClients = new();
var client2 = _lazyClients.GetOrAdd(tenantId, id => new Lazy<ApiClient>(() => CreateExpensiveClient(id))).Value;
```

### Thread safety of common types

| Type | Thread-safe? |
|---|---|
| `List<T>`, `Dictionary<K,V>`, `HashSet<T>` | ❌ (concurrent *reads* are safe only if nobody writes) |
| `HttpClient` | ✅ share it, or use the factory |
| **`DbContext`** | ❌ **never run two operations on it concurrently** (no `Task.WhenAll` over one context) |
| `ILogger<T>`, `IOptionsMonitor<T>` | ✅ |
| `Random` | ❌ use `Random.Shared` (thread-safe) |

### Deadlocks with locks

Two threads take locks A and B in *opposite* order, and each waits forever for the other. **Fix:** always acquire locks in the same global order, keep critical sections tiny, and never call unknown code (callbacks or events) while holding a lock.

---

## 3.15 Periodic async work

```csharp
using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));
while (await timer.WaitForNextTickAsync(stoppingToken))
{
    await RefreshCacheAsync(stoppingToken);   // the next tick never overlaps a running iteration
}
```

---

## 3.16 Async code-review checklist

- [ ] No `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` or `Thread.Sleep` in async flows.
- [ ] No `async void` except UI event handlers. No async lambdas passed to `Action` parameters.
- [ ] A `CancellationToken` is accepted **and passed on** to every I/O call.
- [ ] No fire-and-forget. Background work goes through a queue or worker.
- [ ] `Task.WhenAll` is used for independent I/O, with bounded concurrency for large fan-outs.
- [ ] No `await` inside `lock`. A `SemaphoreSlim` is released in `finally`.
- [ ] No concurrent operations on one `DbContext`.
- [ ] `ConfigureAwait(false)` is used in library code.
- [ ] `ValueTask` is awaited exactly once.
- [ ] `OperationCanceledException` is treated as expected, not logged as an error.

---

## Interview questions

<details><summary><b>1. What happens when <code>await</code> reaches an incomplete Task?</b></summary>

The compiler-generated state machine saves its state, registers a continuation on the awaiter and returns an incomplete Task to the caller, which frees the thread. When the operation completes, the continuation is scheduled on the captured `SynchronizationContext` if there is one, otherwise on the thread pool. It calls `GetResult()`, which returns the value or rethrows, and resumes after the `await`. If the awaited task was already complete, execution simply continues synchronously.
</details>

<details><summary><b>2. Why can <code>.Result</code> deadlock in WPF but not in ASP.NET Core? Is it safe in ASP.NET Core?</b></summary>

WPF has a single-threaded `SynchronizationContext`: the blocked UI thread is the same thread the continuation needs. ASP.NET Core has no context, so continuations run on the pool and there's no classic deadlock. It's still **not safe**: it blocks pool threads and causes thread-pool starvation under load.
</details>

<details><summary><b>3. Task vs ValueTask?</b></summary>

`ValueTask` avoids an allocation when the result is usually available synchronously, but it has restrictions: you can await it only once, you can't await it concurrently, and you can't block on it. Use `Task` by default and `ValueTask` for measured hot paths.
</details>

<details><summary><b>4. When should you use <code>ConfigureAwait(false)</code>?</b></summary>

In general-purpose library code, to avoid capturing the caller's context (for performance and to prevent deadlocks). It isn't needed in ASP.NET Core app code. Don't use it in UI code that must return to the UI thread.
</details>

<details><summary><b>5. Why is <code>async void</code> dangerous?</b></summary>

You can't await it, so callers don't know when it finishes. Its exceptions are raised on the context or thread pool and crash the process. Use it only for event handlers. Watch for async lambdas passed to `Action` parameters, such as `List.ForEach` or `Parallel.ForEach`.
</details>

<details><summary><b>6. How do you run 3 independent HTTP calls in parallel and handle their errors?</b></summary>

Start all three tasks, then `await Task.WhenAll(...)`. The `await` rethrows the first exception. For full details, inspect `whenAll.Exception.InnerExceptions` or the status of each task. Pass the same `CancellationToken` to all three.
</details>

<details><summary><b>7. <code>lock</code> vs <code>SemaphoreSlim</code>?</b></summary>

`lock` is for synchronous critical sections; you can't `await` inside it. `SemaphoreSlim.WaitAsync` supports async critical sections and limiting concurrency to N. Release it in `finally`.
</details>

<details><summary><b>8. How does cancellation work in .NET?</b></summary>

It's cooperative. A `CancellationTokenSource` signals its token, and code checks `IsCancellationRequested` or calls `ThrowIfCancellationRequested`, or passes the token to APIs that honor it. Cancellation surfaces as an `OperationCanceledException`. In ASP.NET Core, `RequestAborted` is bound automatically to an action's `CancellationToken` parameter.
</details>

<details><summary><b>9. What is thread-pool starvation and how do you detect it?</b></summary>

All pool threads are blocked (usually by sync-over-async or sync I/O), so queued work and continuations can't run, and the pool grows only slowly. Symptoms: high latency with low CPU, a growing ThreadPool queue length and a slowly increasing thread count in `dotnet-counters`. Fix it by going async all the way.
</details>

<details><summary><b>10. Is <code>DbContext</code> thread-safe? What happens with <code>Task.WhenAll</code> over the same context?</b></summary>

No. Concurrent operations throw `InvalidOperationException` ("A second operation was started…"). Run the queries sequentially, or use separate contexts through `IDbContextFactory`.
</details>

---

## Exercises

1. **SharpLab:** decompile a 2-await async method and label each part of the generated `MoveNext()`.
2. **Labs:** do [Lab 3 (the deadlock)](09-labs-and-capstone.md#lab-3--reproduce-and-fix-a-sync-over-async-deadlock) and [Lab 4 (thread-pool starvation)](09-labs-and-capstone.md#lab-4--thread-pool-starvation).
3. **Fan-out:** write `GetPricesAsync(IEnumerable<string> skus)` that calls a fake API (`Task.Delay(Random.Shared.Next(50, 300))`) with at most 5 calls in flight, a 2-second overall timeout and correct cancellation. Write tests for the timeout path.
4. **Race fix:** fix the `counter++` race three ways (`Interlocked`, `lock`, a local per-thread sum) and benchmark each.
5. **Code review:** search your work codebase for `.Result`, `.Wait()`, `async void` and `Task.Run` in controllers. Write down every hit and propose a fix.

## Self-check

- [ ] I can explain the state machine, the fast path vs the slow path, and where continuations run.
- [ ] I can explain the UI deadlock *and* ASP.NET Core thread-pool starvation.
- [ ] I pass `CancellationToken`s everywhere and know how to add a timeout.
- [ ] I know when to use `WhenAll`, `WhenAny` and `Parallel.ForEachAsync`.
- [ ] I can pick between `Interlocked`, `lock`, `SemaphoreSlim` and the concurrent collections.
- [ ] I know which common types aren't thread-safe (`DbContext`!).

## Further reading

- *Concurrency in C# Cookbook* (Stephen Cleary). The whole book is relevant to this guide.
- David Fowler, *ASP.NET Core Diagnostic Scenarios: AsyncGuidance.md* (on GitHub).
- Stephen Toub, *"How Async/Await Really Works in C#"* (.NET blog) and *"ConfigureAwait FAQ"*.
- Microsoft Learn: *What's new in .NET 11 runtime*, the Runtime Async section.
