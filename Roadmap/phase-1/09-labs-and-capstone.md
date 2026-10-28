# 9. Labs, Algorithms and Capstone TicketHub v0.1

> Phase 1 · All weeks · [← Phase index](README.md)

This guide is the **hands-on half** of Phase 1. The theory guides tell you *what*; these labs make you *see it happen*. Keep every lab in a Git repository with short notes. Interviewers love "I reproduced a deadlock and fixed it" stories.

**Suggested layout:**

```
dotnet-levelup/
├─ labs/                    ← throwaway experiments (one console project each)
│  ├─ Benchmarks/
│  ├─ LeakHunt/
│  ├─ DeadlockDemo/
│  ├─ StarvationDemo/
│  └─ LoadGen/
└─ notes/                   ← your summaries and benchmark write-ups
TicketHub/                  ← the capstone, in its own Git repository
```

---

## Lab 1 — Benchmark experiments (10 total)

### Setup

```powershell
dotnet new console -n Benchmarks -o labs/Benchmarks -f net10.0
cd labs/Benchmarks
dotnet add package BenchmarkDotNet
```

**Experiment 1 (complete example): boxing.**

```csharp
using System.Collections;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

BenchmarkSwitcher.FromAssembly(typeof(Program).Assembly).Run(args);

[MemoryDiagnoser]
public class BoxingBenchmarks
{
    private const int N = 1_000;

    [Benchmark(Baseline = true)]
    public int GenericList()
    {
        var list = new List<int>(N);
        for (int i = 0; i < N; i++) list.Add(i);
        int sum = 0;
        foreach (int x in list) sum += x;
        return sum;
    }

    [Benchmark]
    public int ArrayList()
    {
        var list = new ArrayList(N);
        for (int i = 0; i < N; i++) list.Add(i);       // boxes every int
        int sum = 0;
        foreach (object x in list) sum += (int)x;      // unboxes every int
        return sum;
    }
}
```

```powershell
dotnet run -c Release -- --filter *Boxing*      # always Release; never benchmark under the debugger
```

**Read the results:** look at the `Mean` (time), `Ratio` (relative to the baseline) and `Allocated` (bytes per operation) columns. Expect `ArrayList` to allocate about 24 KB per call (1,000 boxes × 24 bytes) while `List<int>` allocates only its backing array.

### The 10 experiments

| # | Experiment | Compare | What to look for | Guide |
|---|---|---|---|---|
| 1 | Boxing | `List<int>` vs `ArrayList` | allocations per op | [01](01-type-system.md) |
| 2 | String building (N = 1,000) | `+=` vs `StringBuilder` vs `string.Join` | O(n²) growth of `+=` | [01](01-type-system.md) |
| 3 | Struct dictionary keys | plain `struct` vs `IEquatable<T>` struct vs `readonly record struct` | the cost of reflection-based `Equals`, boxing | [01](01-type-system.md) |
| 4 | LINQ vs loop | `Where().Select().Sum()` vs a `for` loop over `int[]` | the delegate and enumerator overhead | [04](04-collections-and-linq.md) |
| 5 | List pre-sizing | `new List<T>()` vs `new List<T>(n)` for n = 100,000 | reallocations, LOH allocations | [04](04-collections-and-linq.md) |
| 6 | Membership | `List.Contains` vs `HashSet.Contains` vs `FrozenSet.Contains` (1,000 items) | O(n) vs O(1) | [04](04-collections-and-linq.md) |
| 7 | Read-heavy lookups | `Dictionary` vs `FrozenDictionary` (50 string keys); **also benchmark creation** | the read gain vs the build cost | [04](04-collections-and-linq.md) |
| 8 | Parsing | `line.Split(';')` vs `ReadOnlySpan<char>` slicing | allocations | [02](02-memory-and-gc.md) |
| 9 | Async overhead | a sync method vs `async Task` completing synchronously vs `ValueTask` vs `await Task.Yield()` | when the state machine allocates | [03](03-async-and-concurrency.md) |
| 10 | Exceptions vs results | validate 10,000 items (10% invalid) by throwing vs returning `Result` | the cost of throwing | [07](07-error-handling.md) |
| bonus | Large buffers | `new byte[100_000]` vs `ArrayPool<byte>.Shared.Rent(100_000)` | Gen2 collections (the `Gen2` column) | [02](02-memory-and-gc.md) |

### Write-up template (`notes/benchmarks.md`)

```markdown
## Experiment N — <title>
- **Hypothesis:** what I expected and why
- **Setup:** .NET 10.0.x, Release, CPU <model>, N = …
- **Results:** (paste the BenchmarkDotNet summary table)
- **Conclusion:** what surprised me; the rule I'll apply in real code
```

---

## Lab 2 — Memory-leak hunt

**Goal:** find a leak with the real diagnostics tools, prove its cause with `gcroot`, then fix it.

```powershell
dotnet new console -n LeakHunt -o labs/LeakHunt -f net10.0
dotnet tool install -g dotnet-counters
dotnet tool install -g dotnet-gcdump
dotnet tool install -g dotnet-dump
```

```csharp
// labs/LeakHunt/Program.cs
var feed = new PriceFeed();
Console.WriteLine($"PID {Environment.ProcessId}. Press Ctrl+C to stop.");

for (int i = 1; ; i++)
{
    var widget = new PriceWidget(feed);   // subscribes to feed.PriceChanged and never unsubscribes
    widget.Render();                      // the widget is "done" here, but the feed still references it

    if (i % 20_000 == 0)
    {
        long mb = GC.GetTotalMemory(forceFullCollection: true) / 1024 / 1024;
        Console.WriteLine($"{i:N0} widgets | heap after full GC: {mb} MB");
        Thread.Sleep(1_000);
    }
}

sealed class PriceFeed
{
    public event EventHandler<decimal>? PriceChanged;
    public void Publish(decimal price) => PriceChanged?.Invoke(this, price);
}

sealed class PriceWidget
{
    private readonly byte[] _chartBuffer = new byte[1_024];

    public PriceWidget(PriceFeed feed) => feed.PriceChanged += OnPriceChanged;

    public void Render() => _chartBuffer[0] = 1;

    private void OnPriceChanged(object? sender, decimal price) => _chartBuffer[1] = (byte)price;
}
```

**Steps:**

1. Run `dotnet run -c Release`. The heap keeps growing *even after a forced full GC*, so this is a true leak and not just garbage waiting to be collected.
2. In a second terminal, run `dotnet-counters monitor -n LeakHunt --counters System.Runtime`. Watch the GC heap size and the Gen2 GC count.
3. Run `dotnet-gcdump collect -n LeakHunt` twice, about 10 seconds apart. Open both `.gcdump` files in Visual Studio (*File → Open*) or PerfView and compare them. `PriceWidget` and `Byte[]` grow, and so does one huge `Object[]`: the event's **invocation list**.
4. Run `dotnet-dump collect -n LeakHunt`, then `dotnet-dump analyze <dump-file>`:
   ```
   > dumpheap -stat                       # the biggest types by size and count
   > dumpheap -type PriceWidget -short    # addresses of the instances
   > gcroot <one-address>                 # root chain: local 'feed' → PriceFeed → EventHandler → Object[] → PriceWidget
   ```
5. **Explain it:** each `+=` creates a *new* invocation-list array containing every previous handler. So this leak also costs **O(n²) copying** and LOH allocations, and the loop gets slower over time.
6. **Fix it** and verify that memory stays flat:

```csharp
sealed class PriceWidget : IDisposable
{
    private readonly PriceFeed _feed;
    private readonly byte[] _chartBuffer = new byte[1_024];

    public PriceWidget(PriceFeed feed)
    {
        _feed = feed;
        _feed.PriceChanged += OnPriceChanged;
    }

    public void Render() => _chartBuffer[0] = 1;
    public void Dispose() => _feed.PriceChanged -= OnPriceChanged;

    private void OnPriceChanged(object? sender, decimal price) => _chartBuffer[1] = (byte)price;
}

// in the loop:
using var widget = new PriceWidget(feed);
```

**Write-up:** a short note in your own words: the symptom, the tools, the root chain, the fix and the verification.

---

## Lab 3 — Reproduce and fix a sync-over-async deadlock

A console app has no `SynchronizationContext`, so we install a tiny single-threaded one that behaves like a UI thread.

```powershell
dotnet new console -n DeadlockDemo -o labs/DeadlockDemo -f net10.0
```

```csharp
// labs/DeadlockDemo/Program.cs
using System.Collections.Concurrent;

string mode = args.Length > 0 ? args[0] : "deadlock";   // deadlock | configureawait | async

var uiContext = new SingleThreadSyncContext();
SynchronizationContext.SetSynchronizationContext(uiContext);

var watchdog = new Thread(() =>
{
    Thread.Sleep(TimeSpan.FromSeconds(3));
    Console.WriteLine("No progress after 3 s: DEADLOCK. The UI thread is blocked waiting for a continuation that needs the UI thread.");
    Environment.Exit(1);
}) { IsBackground = true };
watchdog.Start();

// Simulates a button click that the UI thread handles
uiContext.Post(_ =>
{
    Console.WriteLine($"[thread {Environment.CurrentManagedThreadId}] click handler started, mode = {mode}");

    if (mode == "async")
    {
        HandleClickAsync(uiContext);                     // async all the way: the UI thread isn't blocked
        return;
    }

    string result = mode == "configureawait"
        ? GetDataWithoutContextAsync().Result            // works, but still blocks the UI thread for 100 ms
        : GetDataAsync().Result;                         // deadlock
    Console.WriteLine($"[thread {Environment.CurrentManagedThreadId}] got: {result}");
    uiContext.Complete();
}, null);

uiContext.RunOnCurrentThread();                           // the "UI message loop"
Console.WriteLine("Finished without deadlock.");

static async void HandleClickAsync(SingleThreadSyncContext uiContext)   // async void: OK for event handlers only
{
    string result = await GetDataAsync();
    Console.WriteLine($"[thread {Environment.CurrentManagedThreadId}] got: {result}");
    uiContext.Complete();
}

static async Task<string> GetDataAsync()
{
    await Task.Delay(100);                    // captures the UI context, so the continuation is POSTED to the UI thread
    return "data";
}

static async Task<string> GetDataWithoutContextAsync()
{
    await Task.Delay(100).ConfigureAwait(false);   // the continuation runs on the thread pool
    return "data";
}

sealed class SingleThreadSyncContext : SynchronizationContext
{
    private readonly BlockingCollection<(SendOrPostCallback Callback, object? State)> _queue = new();

    public override void Post(SendOrPostCallback d, object? state) => _queue.Add((d, state));

    public void RunOnCurrentThread()
    {
        foreach (var (callback, state) in _queue.GetConsumingEnumerable())
            callback(state);
    }

    public void Complete() => _queue.CompleteAdding();
}
```

```powershell
dotnet run -- deadlock         # hangs; the watchdog reports a DEADLOCK after 3 s
dotnet run -- configureawait   # completes (it hides the problem)
dotnet run -- async            # completes (the real fix)
```

**Explain in your notes:** draw the cycle "UI thread blocked in `.Result` → task waits for its continuation → the continuation is queued to the UI thread". Then explain why `ConfigureAwait(false)` *masks* the problem while *async all the way* actually fixes it.

---

## Lab 4 — Thread-pool starvation

**The server:**

```powershell
dotnet new web -n StarvationDemo -o labs/StarvationDemo -f net10.0
```

```csharp
// labs/StarvationDemo/Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/async", async () =>
{
    await Task.Delay(500);                 // releases the thread while "waiting for I/O"
    return "ok";
});

app.MapGet("/sync-over-async", () =>
{
    Task.Delay(500).Wait();                // blocks a thread-pool thread for 500 ms
    return "ok";
});

app.Run("http://localhost:5000");
```

**The load generator** (200 concurrent requests):

```powershell
dotnet new console -n LoadGen -o labs/LoadGen -f net10.0
```

```csharp
// labs/LoadGen/Program.cs
using System.Diagnostics;

string url = args.Length > 0 ? args[0] : "http://localhost:5000/async";
using var client = new HttpClient { Timeout = TimeSpan.FromMinutes(2) };

var total = Stopwatch.StartNew();
var requests = Enumerable.Range(0, 200).Select(async _ =>
{
    long start = Stopwatch.GetTimestamp();
    using var response = await client.GetAsync(url);
    return Stopwatch.GetElapsedTime(start).TotalMilliseconds;
});

double[] latencies = await Task.WhenAll(requests);
Array.Sort(latencies);
Console.WriteLine(
    $"{url}\n total {total.ElapsedMilliseconds} ms | p50 {latencies[latencies.Length / 2]:F0} ms | " +
    $"p95 {latencies[(int)(latencies.Length * 0.95)]:F0} ms | max {latencies[^1]:F0} ms");
```

**Steps:**

1. Start the server with `dotnet run -c Release --project labs/StarvationDemo`.
2. Start monitoring with `dotnet-counters monitor -n StarvationDemo --counters System.Runtime`. Watch the thread-pool **thread count** and **queue length**.
3. Run `dotnet run -c Release --project labs/LoadGen -- http://localhost:5000/async`. Expect a total of about 0.5–0.7 s and a flat thread count.
4. Run `dotnet run -c Release --project labs/LoadGen -- http://localhost:5000/sync-over-async`. Expect **several seconds**, a growing queue and a climbing thread count. The exact numbers depend on your core count. Since .NET 6 the pool injects threads faster for blocking waits, which reduces the effect without removing it.
5. **Write down:** the numbers for both endpoints, the counter behavior, and a one-paragraph explanation for a teammate.

---

## Algorithms: 25 problems for Phase 1

**Method for each problem:**

1. Spend 25 minutes on it **without AI**.
2. If you're stuck, read a *hint*, not the solution.
3. After solving it, write the time and space complexity in a comment.
4. **Re-solve it 7 days later** (spaced repetition).

| Pattern | Problems (LeetCode names) |
|---|---|
| Arrays and hashing | Two Sum · Contains Duplicate · Valid Anagram · Group Anagrams · Top K Frequent Elements · Product of Array Except Self · Longest Consecutive Sequence |
| Two pointers | Valid Palindrome · Two Sum II · 3Sum · Container With Most Water |
| Sliding window | Best Time to Buy and Sell Stock · Longest Substring Without Repeating Characters · Longest Repeating Character Replacement |
| Stack | Valid Parentheses · Min Stack · Evaluate Reverse Polish Notation · Daily Temperatures |
| Binary search | Binary Search · Search a 2D Matrix · Koko Eating Bananas · Find Minimum in Rotated Sorted Array |
| Linked list | Reverse Linked List · Merge Two Sorted Lists · Linked List Cycle |

**C# toolkit for interviews:**

```csharp
var seen = new HashSet<int>();
var counts = new Dictionary<char, int>();
counts[c] = counts.GetValueOrDefault(c) + 1;                 // frequency count

var stack = new Stack<int>();       stack.Push(1); stack.TryPop(out var top);
var queue = new Queue<(int Row, int Col)>(); queue.Enqueue((0, 0));
var heap = new PriorityQueue<string, int>();                  // a min-heap by priority
heap.Enqueue("job", priority: 3); heap.TryDequeue(out var job, out var prio);

Array.Sort(nums);                                             // in place, unstable
int[] sorted = [.. nums.Order()];                             // a sorted copy (.NET 7+)
int idx = Array.BinarySearch(sorted, target);                 // a negative result means "not found" (~idx = insertion point)
Span<int> window = nums.AsSpan(start, length);                // a slice without copying
string s = new string(chars);  var sb = new StringBuilder();  // build strings efficiently
```

---

## Capstone: TicketHub v0.1 (domain model)

### Scope

Phase 1 builds **only the domain**: pure C#, no database, no HTTP. Everything is unit-testable in milliseconds.

| Concept | Rules (invariants) |
|---|---|
| `Money` (value object) | amount ≥ 0, rounded to 2 decimals; a 3-letter currency; adding different currencies throws |
| `SeatId` (value object) | section, row and number are required; normalized to upper case |
| `Event` (aggregate) | `Draft → Published → Cancelled`; publishing only from Draft; holding seats only when Published |
| `SeatHold` | 1–10 **distinct** seats; every seat must exist and be free; expires 10 minutes after creation; expired holds free their seats |
| **Your tasks** | `ConfirmHold` (the seats become *sold*), `ReleaseHold`, pricing per seat category with `Money`, and an `Order` aggregate with lines and a total |

### 1. Create the solution

```powershell
mkdir TicketHub; cd TicketHub
git init
dotnet new gitignore
dotnet new editorconfig
dotnet new sln -n TicketHub                      # the .NET 10 SDK creates the new XML format: TicketHub.slnx
dotnet new classlib -n TicketHub.Domain -o src/TicketHub.Domain -f net10.0
dotnet new install xunit.v3.templates
dotnet new xunit3 -n TicketHub.Domain.Tests -o tests/TicketHub.Domain.Tests -f net10.0   # defaults to net8.0 without -f
dotnet sln add src/TicketHub.Domain tests/TicketHub.Domain.Tests
dotnet add tests/TicketHub.Domain.Tests reference src/TicketHub.Domain
dotnet add tests/TicketHub.Domain.Tests package Shouldly
dotnet add tests/TicketHub.Domain.Tests package Microsoft.Extensions.TimeProvider.Testing
```

Then **delete** the `<TargetFramework>`, `<Nullable>` and `<ImplicitUsings>` lines from both generated `.csproj` files, so that `Directory.Build.props` controls them. Also delete the template's `Class1.cs` and `UnitTest1.cs`.

**`global.json`:** the xUnit v3 template already created this file, with a `test` section that switches `dotnet test` to **Microsoft.Testing.Platform**. Add an `sdk` section next to it. That pins a minimum SDK, so everyone builds with a known SDK, while still allowing the .NET 11 SDK:

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestMajor"
  },
  "test": {
    "runner": "Microsoft.Testing.Platform"
  }
}
```

**`Directory.Build.props`** holds the shared settings for every project:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>
</Project>
```

**Append to `.editorconfig`:**

```ini
[*.cs]
# Application code doesn't need ConfigureAwait(false); enable this in reusable libraries
dotnet_diagnostic.CA2007.severity = none
# C#-only codebase: names like Event/Error are fine (this rule protects VB.NET consumers, where they're keywords)
dotnet_diagnostic.CA1716.severity = none
# Possible multiple enumeration of IEnumerable
dotnet_diagnostic.CA1851.severity = warning

[src/TicketHub.Domain/Common/Result.cs]
# The static factories Result<T>.Success/Failure are intentional
dotnet_diagnostic.CA1000.severity = none

[tests/**.cs]
# Test names use underscores for readability
dotnet_diagnostic.CA1707.severity = none
```

> I verified these settings against the starter code below with the .NET 10 SDK: the build has zero warnings and all 8 starter tests pass. Without the CA1716 and CA1000 lines, `latest-recommended` reports 4 errors, which is a good first lesson in reading analyzer output. If other rules fire as your code grows, **fix the code** or adjust *that specific rule* with a comment explaining why. Never switch analyzers off wholesale.

### 2. Domain code (starter)

Copy `Result`, `Result<T>` and `Error` from [guide 7](07-error-handling.md#73-the-result-pattern) into `src/TicketHub.Domain/Common/Result.cs`.

```csharp
// src/TicketHub.Domain/Common/Money.cs
namespace TicketHub.Domain.Common;

public readonly record struct Money
{
    public Money(decimal amount, string currency)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(amount);
        ArgumentException.ThrowIfNullOrWhiteSpace(currency);
        if (currency.Length != 3)
            throw new ArgumentException("Currency must be a 3-letter ISO code.", nameof(currency));

        Amount = decimal.Round(amount, 2, MidpointRounding.ToEven);
        Currency = currency.ToUpperInvariant();
    }

    public decimal Amount { get; }
    public string Currency { get; }   // default(Money) has Currency == null: decide how to handle it (see guide 1)

    public static Money operator +(Money left, Money right)
    {
        if (left.Currency != right.Currency)
            throw new InvalidOperationException($"Cannot add {left.Currency} to {right.Currency}.");
        return new Money(left.Amount + right.Amount, left.Currency);
    }

    public override string ToString() => $"{Amount:0.00} {Currency}";
}
```

```csharp
// src/TicketHub.Domain/Events/SeatId.cs
namespace TicketHub.Domain.Events;

public readonly record struct SeatId
{
    public SeatId(string section, string row, int number)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(section);
        ArgumentException.ThrowIfNullOrWhiteSpace(row);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(number);

        Section = section.Trim().ToUpperInvariant();
        Row = row.Trim().ToUpperInvariant();
        Number = number;
    }

    public string Section { get; }
    public string Row { get; }
    public int Number { get; }

    public override string ToString() => $"{Section}-{Row}-{Number}";
}
```

```csharp
// src/TicketHub.Domain/Events/SeatHold.cs
namespace TicketHub.Domain.Events;

public sealed record SeatHold(Guid Id, Guid CustomerId, IReadOnlyList<SeatId> Seats, DateTimeOffset ExpiresAt)
{
    public bool IsExpired(DateTimeOffset now) => now >= ExpiresAt;
}
```

```csharp
// src/TicketHub.Domain/Events/EventStatus.cs
namespace TicketHub.Domain.Events;

public enum EventStatus
{
    Draft = 0,
    Published = 1,
    Cancelled = 2,
}
```

```csharp
// src/TicketHub.Domain/Events/EventErrors.cs
using TicketHub.Domain.Common;

namespace TicketHub.Domain.Events;

public static class EventErrors
{
    public static Error NotDraft(Guid id) =>
        new("Event.NotDraft", $"Event {id} can only be published from Draft status.");

    public static Error AlreadyCancelled(Guid id) =>
        new("Event.AlreadyCancelled", $"Event {id} is already cancelled.");

    public static Error NotOnSale(Guid id) =>
        new("Event.NotOnSale", $"Event {id} is not on sale.");

    public static Error InvalidSeatCount(int max) =>
        new("Event.InvalidSeatCount", $"You can hold between 1 and {max} distinct seats.");

    public static Error SeatsUnavailable(IEnumerable<SeatId> seats) =>
        new("Event.SeatsUnavailable", $"Seats unavailable: {string.Join(", ", seats)}.");
}
```

```csharp
// src/TicketHub.Domain/Events/Event.cs
using TicketHub.Domain.Common;

namespace TicketHub.Domain.Events;

public sealed class Event
{
    public const int MaxSeatsPerHold = 10;
    public static readonly TimeSpan HoldDuration = TimeSpan.FromMinutes(10);

    private readonly HashSet<SeatId> _seats;
    private readonly Dictionary<SeatId, SeatHold> _holdsBySeat = new();

    public Event(Guid id, string name, IEnumerable<SeatId> seats)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        ArgumentNullException.ThrowIfNull(seats);

        Id = id;
        Name = name.Trim();
        _seats = seats.ToHashSet();
        if (_seats.Count == 0)
            throw new ArgumentException("An event needs at least one seat.", nameof(seats));
    }

    public Guid Id { get; }
    public string Name { get; }
    public EventStatus Status { get; private set; } = EventStatus.Draft;
    public int Capacity => _seats.Count;

    public Result Publish()
    {
        if (Status != EventStatus.Draft) return EventErrors.NotDraft(Id);
        Status = EventStatus.Published;
        return Result.Success();
    }

    public Result Cancel()
    {
        if (Status == EventStatus.Cancelled) return EventErrors.AlreadyCancelled(Id);
        Status = EventStatus.Cancelled;
        _holdsBySeat.Clear();
        return Result.Success();
    }

    public Result<SeatHold> HoldSeats(IReadOnlyCollection<SeatId> seatIds, Guid customerId, TimeProvider clock)
    {
        ArgumentNullException.ThrowIfNull(seatIds);
        ArgumentNullException.ThrowIfNull(clock);

        if (Status != EventStatus.Published) return EventErrors.NotOnSale(Id);

        var requested = seatIds.Distinct().ToList();
        if (requested.Count is 0 or > MaxSeatsPerHold || requested.Count != seatIds.Count)
            return EventErrors.InvalidSeatCount(MaxSeatsPerHold);

        var now = clock.GetUtcNow();
        ReleaseExpiredHolds(now);

        var unavailable = requested.Where(s => !_seats.Contains(s) || _holdsBySeat.ContainsKey(s)).ToList();
        if (unavailable.Count > 0) return EventErrors.SeatsUnavailable(unavailable);

        var hold = new SeatHold(Guid.NewGuid(), customerId, requested, now + HoldDuration);
        foreach (var seat in requested)
            _holdsBySeat[seat] = hold;

        return hold;
    }

    public int AvailableSeatCount(TimeProvider clock)
    {
        ArgumentNullException.ThrowIfNull(clock);
        ReleaseExpiredHolds(clock.GetUtcNow());
        return _seats.Count - _holdsBySeat.Count;
    }

    private void ReleaseExpiredHolds(DateTimeOffset now)
    {
        var expired = _holdsBySeat.Where(kv => kv.Value.IsExpired(now)).Select(kv => kv.Key).ToList();
        foreach (var seat in expired)
            _holdsBySeat.Remove(seat);
    }
}
```

> **Discussion point for your notes:** `Event` isn't thread-safe, and that's fine. In Phase 2, the *database* (optimistic concurrency) protects against two requests holding the same seat at the same time. The domain enforces rules *within* one operation.

### 3. Tests (starter)

```csharp
// tests/TicketHub.Domain.Tests/Events/EventHoldSeatsTests.cs
using Microsoft.Extensions.Time.Testing;
using Shouldly;
using TicketHub.Domain.Events;

namespace TicketHub.Domain.Tests.Events;

public class EventHoldSeatsTests
{
    private static readonly SeatId A1 = new("Stalls", "A", 1);
    private static readonly SeatId A2 = new("Stalls", "A", 2);

    private readonly FakeTimeProvider _clock = new(new DateTimeOffset(2027, 1, 10, 18, 0, 0, TimeSpan.Zero));

    private static Event PublishedEvent()
    {
        var evt = new Event(Guid.NewGuid(), "Jazz Night", [A1, A2]);
        evt.Publish().IsSuccess.ShouldBeTrue();
        return evt;
    }

    [Fact]
    public void Holding_free_seats_succeeds_and_expires_after_the_hold_duration()
    {
        var evt = PublishedEvent();

        var result = evt.HoldSeats([A1], Guid.NewGuid(), _clock);

        result.IsSuccess.ShouldBeTrue();
        result.Value.Seats.ShouldHaveSingleItem().ShouldBe(A1);
        result.Value.ExpiresAt.ShouldBe(_clock.GetUtcNow() + Event.HoldDuration);
        evt.AvailableSeatCount(_clock).ShouldBe(1);
    }

    [Fact]
    public void Holding_a_seat_that_is_already_held_fails()
    {
        var evt = PublishedEvent();
        evt.HoldSeats([A1], Guid.NewGuid(), _clock);

        var result = evt.HoldSeats([A1, A2], Guid.NewGuid(), _clock);

        result.IsFailure.ShouldBeTrue();
        result.Error!.Code.ShouldBe("Event.SeatsUnavailable");
    }

    [Fact]
    public void A_seat_is_still_held_one_tick_before_expiry()
    {
        var evt = PublishedEvent();
        evt.HoldSeats([A1], Guid.NewGuid(), _clock);

        _clock.Advance(Event.HoldDuration - TimeSpan.FromTicks(1));

        evt.HoldSeats([A1], Guid.NewGuid(), _clock).IsFailure.ShouldBeTrue();
    }

    [Fact]
    public void A_seat_becomes_available_again_when_the_hold_expires()
    {
        var evt = PublishedEvent();
        evt.HoldSeats([A1], Guid.NewGuid(), _clock);

        _clock.Advance(Event.HoldDuration);

        evt.HoldSeats([A1], Guid.NewGuid(), _clock).IsSuccess.ShouldBeTrue();
    }

    [Theory]
    [InlineData(0)]
    [InlineData(11)]
    public void Holding_an_invalid_number_of_seats_fails(int count)
    {
        var evt = PublishedEvent();
        var seats = Enumerable.Range(1, count).Select(n => new SeatId("Stalls", "B", n)).ToList();

        var result = evt.HoldSeats(seats, Guid.NewGuid(), _clock);

        result.Error!.Code.ShouldBe("Event.InvalidSeatCount");
    }

    [Fact]
    public void Holding_the_same_seat_twice_in_one_request_fails()
    {
        var result = PublishedEvent().HoldSeats([A1, A1], Guid.NewGuid(), _clock);

        result.Error!.Code.ShouldBe("Event.InvalidSeatCount");
    }

    [Fact]
    public void Holding_seats_on_a_draft_event_fails()
    {
        var draft = new Event(Guid.NewGuid(), "Jazz Night", [A1]);

        var result = draft.HoldSeats([A1], Guid.NewGuid(), _clock);

        result.Error!.Code.ShouldBe("Event.NotOnSale");
    }
}
```

**Add the rest yourself** (following [guide 8, section 8.7](08-testing.md#87-what-to-test-in-a-domain-model-phase-1-focus)):

- `Money` tests: rounding, a negative amount, a currency mismatch, and the `default(Money)` behavior you chose.
- `Event` lifecycle tests: publishing twice, cancelling twice, holds being cleared on cancel.
- The tasks you implement yourself: `ConfirmHold`, `ReleaseHold`, pricing and `Order`.

### 4. Continuous integration

```yaml
# .github/workflows/ci.yml
name: ci

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'      # the latest .NET 10 patch; satisfies global.json
      - run: dotnet restore
      - run: dotnet build --no-restore -c Release
      - run: dotnet test --no-build -c Release
```

> Update the action versions to their latest major versions when you create the file.

### 5. Definition of Done for v0.1

- [ ] `dotnet build` passes with **zero warnings** (warnings are errors), with NRT on.
- [ ] Every invariant in the scope table has at least one test, and every error code has a test.
- [ ] Domain coverage is above 80%, and you've run Stryker at least once and killed the obvious surviving mutants.
- [ ] CI is green on `main`.
- [ ] The `README.md` covers what TicketHub is, how to build and test it, and the domain rules.
- [ ] `docs/adr/0001-result-pattern-for-domain-errors.md` explains why you chose `Result` over exceptions for business rules (use a short context → decision → consequences format).
- [ ] The release is tagged: `git tag v0.1.0` and `git push --tags`, with release notes on GitHub.
- [ ] Your brag document is updated with what you built and learned.
