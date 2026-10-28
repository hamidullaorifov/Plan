# 2. Memory, the Garbage Collector and Resource Management

> Phase 1 · Week 2 · [← Phase index](README.md)

**Why this matters:** In production, many .NET performance and stability problems come down to memory: latency spikes from GC pauses, OOM kills in containers, memory leaks, and connections or file handles that were never released. A Middle developer must be able to *reason* about allocations and *diagnose* a leak.

---

## 2.1 The mental model

```
┌─────────────── Process ────────────────────────────────────────────┐
│  Thread 1 stack   Thread 2 stack   …   (≈1 MB each by default on    │
│  ┌──────────┐     ┌──────────┐          Windows; frames of locals,  │
│  │ frames   │     │ frames   │          args, return addresses)     │
│  └──────────┘     └──────────┘                                      │
│                                                                     │
│  Managed heap (GC)                                                  │
│  ┌── SOH: Gen0 │ Gen1 │ Gen2 ──┐  ┌── LOH ──┐  ┌── POH ──┐          │
│  └─────────────────────────────┘  └─────────┘  └─────────┘          │
│                                                                     │
│  Native memory (runtime, JIT code, unmanaged libs, buffers)         │
└─────────────────────────────────────────────────────────────────────┘
```

- **Stack:** one per thread. Allocating on it is essentially free, and it's cleaned up automatically when a method returns. Too much recursion causes a `StackOverflowException`, which **can't be caught** and kills the process.
- **Managed heap:** allocating is **cheap**. It's usually just a pointer bump in the current allocation context. **The expensive part is collection.** Every allocation you avoid also saves future GC work.

---

## 2.2 How the GC works

### Reachability

The GC doesn't count references. It starts from the **roots** and marks every object it can reach:

- local variables and arguments in active stack frames (and CPU registers),
- static fields,
- GC handles (pinned handles, handles held by native code),
- the finalization queue.

**Everything that isn't marked is garbage.** The surviving objects are then (usually) **compacted** to remove fragmentation.

### Generations

This rests on the **generational hypothesis**: *most objects die young.*

| Generation | Contains | Collected | Cost |
|---|---|---|---|
| **Gen0** | newly allocated objects | very often | cheap (small) |
| **Gen1** | objects that survived one GC (a buffer between Gen0 and Gen2) | often | cheap |
| **Gen2** | long-lived objects (caches, singletons, statics) | rarely | **expensive**: scans the whole heap |
| **LOH** (Large Object Heap) | objects of **85,000 bytes or more** (large arrays and strings) | only during Gen2 GCs; **not compacted** by default | expensive and fragments |
| **POH** (Pinned Object Heap, .NET 5+) | arrays allocated with `GC.AllocateArray<T>(n, pinned: true)` | with Gen2 | avoids fragmenting the SOH (small object heap) |

**Objects that survive a GC are promoted to the next generation.** The worst case is an object that lives *just long enough* to reach Gen2 and then dies (the "mid-life crisis"). A per-request object cached for a few minutes is a typical example, and it forces expensive Gen2 collections.

### Why LOH allocations hurt

1. Each one is a large allocation that has to be zeroed.
2. The LOH is only collected during a Gen2 GC, so allocating there frequently **triggers Gen2 GCs**.
3. The LOH isn't compacted, so it fragments and memory grows.

**Fixes:** reuse buffers (`ArrayPool<T>`), stream instead of materializing (`Stream`, `IAsyncEnumerable`, `System.IO.Pipelines`), and avoid building huge strings and `byte[]`s.

### GC flavors

| Setting | Default | Notes |
|---|---|---|
| **Workstation GC** | console and desktop apps | one heap; lower memory use |
| **Server GC** | ASP.NET Core apps (`Microsoft.NET.Sdk.Web`) | a heap and GC thread per core, so higher throughput and more memory |
| **Background (concurrent) GC** | on | Gen2 is collected mostly while your threads keep running |
| **DATAS** (Dynamic Adaptation To Application Sizes) | on with Server GC since .NET 9 | adjusts the number of heaps to the load. Good for containers |
| **Container awareness** | automatic | the heap hard limit defaults to about 75% of the container's memory limit |

```xml
<!-- csproj: choose explicitly only when you have a measured reason -->
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

> ⚠️ **Don't call `GC.Collect()`** in application code. It disrupts the GC's tuning and promotes objects. The rare legitimate uses are in benchmarks and tests.

---

## 2.3 Memory leaks in managed code

The GC frees *unreachable* objects. A "leak" in .NET means **something still references objects you no longer need**.

| Leak | What happens | Fix |
|---|---|---|
| **Event handlers** | `publisher.Event += subscriber.Handler`: a long-lived publisher keeps every subscriber alive | unsubscribe (`-=`) in `Dispose`; weak events; avoid subscribing to static events |
| **Static collections and caches** | `static List<T>` or a `Dictionary` that only grows | bounded caches (`IMemoryCache` with a size limit and expiration) |
| **Captured closures** | a lambda captures a big object and is stored somewhere long-lived | capture only what you need; use `static` lambdas |
| **Timers** | `System.Threading.Timer` callbacks keep their target alive | dispose the timer |
| **Captive dependencies** | a singleton holds a scoped or transient service (see [guide 6](06-generic-host-di-config-logging.md)) | fix the lifetimes |
| **Undisposed unmanaged resources** | handles, sockets and native memory pile up | `using` / `Dispose` |
| **`HttpClient` per request** | socket exhaustion (sockets stuck in TIME_WAIT) | `IHttpClientFactory` or a shared `HttpClient` |

---

## 2.4 `IDisposable`, `using` and finalizers

The GC manages **memory**. It doesn't know about **other resources**: file handles, sockets, database connections, locks or native memory. `IDisposable` is the contract for **deterministic cleanup**.

```csharp
// using declaration: disposed at the end of the enclosing scope
using var stream = File.OpenRead(path);

// using statement: disposed at the closing brace, even if an exception is thrown
using (var connection = new SqlConnection(cs))
{
    await connection.OpenAsync(ct);
}

// async disposal (IAsyncDisposable): DbContext, streams, Utf8JsonWriter, ...
await using var writer = new StreamWriter(outputPath);
```

Under the hood, `using` compiles to `try { … } finally { x?.Dispose(); }`.

### Implementing `IDisposable`: the 95% case

When your class **owns other disposables**, but no raw native handles, you don't need a finalizer:

```csharp
public sealed class ReportExporter : IDisposable
{
    private readonly FileStream _file;
    private bool _disposed;

    public ReportExporter(string path) => _file = File.Create(path);

    public void Write(ReadOnlySpan<byte> data)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        _file.Write(data);
    }

    public void Dispose()
    {
        if (_disposed) return;
        _file.Dispose();
        _disposed = true;
    }
}
```

### The full dispose pattern (rare)

You need the full pattern only if you hold a **raw native handle**. Even then, prefer wrapping the handle in a `SafeHandle`, which already has a correct finalizer.

```csharp
public class NativeBuffer : IDisposable
{
    private IntPtr _memory = Marshal.AllocHGlobal(1024);
    private bool _disposed;

    ~NativeBuffer() => Dispose(disposing: false);   // finalizer: a safety net only

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);                  // already cleaned up, so skip the finalizer
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing)
        {
            // dispose managed disposables here
        }
        Marshal.FreeHGlobal(_memory);               // always free unmanaged memory
        _memory = IntPtr.Zero;
        _disposed = true;
    }
}
```

### Why finalizers are expensive

Objects with a finalizer are registered on the **finalization queue** when they're allocated. When such an object becomes unreachable, it moves to the *f-reachable* queue, the **finalizer thread** runs its finalizer, and only the *next* GC can free it. So the object **survives at least one extra GC and gets promoted**. That's why `GC.SuppressFinalize` matters.

### Who disposes?

**The owner does.** Whoever *created* the object disposes it.

- The DI container disposes the services *it* created. **Never dispose an injected service yourself.**
- If a method returns an `IDisposable` to you, *you* own it.

---

## 2.5 `Span<T>`, `ReadOnlySpan<T>` and `Memory<T>`

`Span<T>` is a **view over contiguous memory**: a managed array, a slice of an array, `stackalloc` memory, native memory or a string (`ReadOnlySpan<char>`). It **doesn't copy** anything, and it is a `ref struct`, so it stays on the stack.

```csharp
int[] numbers = [1, 2, 3, 4, 5, 6];
Span<int> middle = numbers.AsSpan(2, 3);   // [3, 4, 5]: a view, no copy
middle[0] = 99;                            // numbers[2] is now 99

ReadOnlySpan<char> text = "key=value";
int eq = text.IndexOf('=');
ReadOnlySpan<char> key = text[..eq];       // no substring allocation
```

**`stackalloc` with a safe fallback for large sizes:**

```csharp
using System.Buffers;

static int CountVowels(ReadOnlySpan<char> input)
{
    const int MaxStack = 256;
    char[]? rented = null;
    Span<char> buffer = input.Length <= MaxStack
        ? stackalloc char[MaxStack]
        : (rented = ArrayPool<char>.Shared.Rent(input.Length));
    try
    {
        int written = input.ToLowerInvariant(buffer);
        int count = 0;
        foreach (char c in buffer[..written])
            if ("aeiou".Contains(c)) count++;
        return count;
    }
    finally
    {
        if (rented is not null) ArrayPool<char>.Shared.Return(rented);
    }
}
```

| Type | Lives | Use when |
|---|---|---|
| `Span<T>` / `ReadOnlySpan<T>` | stack only | synchronous code: parsing, formatting, slicing |
| `Memory<T>` / `ReadOnlyMemory<T>` | can be stored on the heap | you need to keep a buffer in a field or pass it **across an `await`** |

---

## 2.6 `ArrayPool<T>`: reuse instead of allocating

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(minimumLength: 64 * 1024);
try
{
    int read = await stream.ReadAsync(buffer.AsMemory(0, 64 * 1024), ct);
    Process(buffer.AsSpan(0, read));    // the rented array may be LARGER than requested
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer, clearArray: false); // clearArray: true for sensitive data
}
```

**Rules:**

- Never use an array after returning it.
- Never return the same array twice.
- Always slice to the length you actually requested.

---

## 2.7 Other everyday allocation reducers

| Instead of | Prefer | Why |
|---|---|---|
| `s1 + s2 + s3` in a loop | `StringBuilder`, or `string.Create` / interpolation | avoids intermediate strings |
| LINQ in a hot loop | a plain `for`/`foreach` | avoids enumerator and delegate allocations |
| A lambda capturing locals in a hot path | a `static` lambda plus a state argument | avoids allocating a closure |
| A `List<T>` that grows from empty | `new List<T>(expectedCount)` | avoids repeated resize-and-copy |
| `params object[]` APIs | generic overloads, `params ReadOnlySpan<T>` (C# 13) | avoids an array allocation plus boxing |
| `.ToList()` "just in case" | stream with `IEnumerable<T>` / `IAsyncEnumerable<T>` | avoids materializing everything |
| `new HttpClient()` per call | `IHttpClientFactory` | sockets and handlers are pooled |

> **The golden rule:** *measure first.* Optimize the hot paths that a profiler or benchmark points to. Clean, readable code matters more everywhere else.

---

## 2.8 Measuring and diagnosing

| Question | Tool |
|---|---|
| How much does this method allocate? | BenchmarkDotNet `[MemoryDiagnoser]`; `GC.GetAllocatedBytesForCurrentThread()` |
| Is the app GC-heavy right now? | `dotnet-counters monitor -n MyApp --counters System.Runtime`. Watch the GC heap size, Gen0/1/2 counts, % time in GC and allocation rate |
| What is filling the heap? | `dotnet-gcdump collect -p <pid>`, then open the `.gcdump` in Visual Studio or PerfView and compare two snapshots |
| Who keeps this object alive? | `dotnet-dump collect -p <pid>`, then `dotnet-dump analyze <file>` with `dumpheap -stat`, `dumpheap -mt <MT>` and `gcroot <address>` |
| Where is the CPU time spent? | `dotnet-trace collect -p <pid>`, opened in PerfView, Visual Studio or speedscope |

**Leak investigation workflow:**

1. Watch the memory trend with `dotnet-counters`. Does it grow without ever coming back down?
2. Take heap snapshot A, run the suspicious scenario N times, and take snapshot B.
3. Diff the snapshots and find the types whose count grows with N.
4. Run `gcroot` on one instance to find the chain of references that keeps it alive (an event, a static, or a cache).
5. Fix it and verify that the count stays flat.

You'll practice this exact workflow in [Lab 2 of guide 9](09-labs-and-capstone.md#lab-2--memory-leak-hunt).

---

## Interview questions

<details><summary><b>1. How does the .NET GC decide what to collect?</b></summary>

It marks every object reachable from the roots (stack locals, statics, GC handles and the finalization queue). Everything unmarked is garbage. The survivors are compacted and promoted to the next generation.
</details>

<details><summary><b>2. What are generations, and why do they exist?</b></summary>

Gen0, Gen1 and Gen2 exploit the fact that most objects die young. Collecting only the young generations is cheap, because it's small and mostly garbage. Gen2 holds long-lived objects and is collected rarely, because collecting it is expensive.
</details>

<details><summary><b>3. What is the LOH, and why should you care?</b></summary>

The LOH holds objects of 85,000 bytes or more. It's collected only during Gen2 GCs and isn't compacted by default, so frequent large allocations cause Gen2 GCs and fragmentation. Pool or stream large buffers instead.
</details>

<details><summary><b>4. What's the difference between <code>Dispose</code> and a finalizer?</b></summary>

`Dispose` is deterministic: the caller invokes it through `using`. A finalizer is a non-deterministic safety net that the finalizer thread runs. Finalizable objects survive an extra GC and get promoted, which is why `Dispose` calls `GC.SuppressFinalize`. Prefer `SafeHandle` over writing finalizers.
</details>

<details><summary><b>5. Can you have a memory leak in .NET? Give examples.</b></summary>

Yes, whenever unneeded objects stay reachable. Examples: events on long-lived publishers, unbounded static caches, captured closures, undisposed timers, and singletons holding scoped services.
</details>

<details><summary><b>6. What are <code>Span&lt;T&gt;</code> and <code>Memory&lt;T&gt;</code>, and how do they differ?</b></summary>

Both are views over contiguous memory that avoid copying. `Span<T>` is a `ref struct` that stays on the stack, so it can't be used across an `await` or stored on the heap. `Memory<T>` can be stored on the heap and used in async code, and you get a `Span` from it with `.Span`.
</details>

<details><summary><b>7. What's the difference between Server GC and Workstation GC?</b></summary>

Workstation GC uses one heap and a smaller footprint, and it's the default for client apps. Server GC uses a heap and GC thread per core for higher throughput, and it's the default for ASP.NET Core. Since .NET 9, DATAS adapts the number of heaps dynamically.
</details>

<details><summary><b>8. How would you investigate memory that keeps growing in production?</b></summary>

1. Confirm the trend with `dotnet-counters`.
2. Take two `dotnet-gcdump` snapshots and diff them to find the types that grow.
3. Use `dotnet-dump` with `gcroot` on an instance to find what keeps it alive.
4. Fix it and verify the result in a load test.
</details>

---

## Exercises

1. **Allocation intuition:** for each line in a 20-line method of yours, predict whether it allocates. Verify with `GC.GetAllocatedBytesForCurrentThread()` before and after.
2. **A Dispose review:** find three classes at work that own disposables. Check that they implement `IDisposable` correctly and that their callers use `using`.
3. **Spans:** rewrite a `Split`-based CSV line parser to use `ReadOnlySpan<char>`. Benchmark the old and new versions for time and bytes allocated.
4. **LOH:** benchmark allocating `new byte[100_000]` versus `ArrayPool<byte>.Shared.Rent(100_000)` in a loop. Compare the Gen2 counts.
5. **Leak hunt:** complete [Lab 2](09-labs-and-capstone.md#lab-2--memory-leak-hunt).

## Self-check

- [ ] I can draw the heap and stack and explain roots, marking and compaction.
- [ ] I can explain the generations, promotion and the LOH, and why mid-life objects hurt.
- [ ] I can write a correct `IDisposable` and know when a finalizer is justified (almost never).
- [ ] I know four causes of managed leaks.
- [ ] I can use `dotnet-counters`, `dotnet-gcdump` and `dotnet-dump` with `gcroot`.

## Further reading

- Microsoft Learn: *Fundamentals of garbage collection*, *Implement a Dispose method*, *Memory&lt;T&gt; and Span&lt;T&gt; usage guidelines*, *Debug a memory leak tutorial*.
- *Pro .NET Memory Management* (Konrad Kokosa), chapters on generations, the LOH and finalization.
- Maoni Stephens's GC blog posts (the .NET GC architect).
