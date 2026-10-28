# 4. Collections and LINQ

> Phase 1 · Week 5 · [← Phase index](README.md)

**Why this matters:** Picking the wrong collection or writing an accidentally quadratic LINQ query is the most common cause of "it worked in dev and is slow in prod". `IEnumerable` vs `IQueryable` confusion also causes the classic "we loaded the whole table into memory" incident.

---

## 4.1 Complexity cheat sheet

| Collection | Lookup by key/index | Add | Insert/remove in the middle | Contains | Ordered? | Notes |
|---|---|---|---|---|---|---|
| `T[]` | O(1) by index | n/a (fixed size) | O(n) copy | O(n) | insertion order | the fastest iteration |
| `List<T>` | O(1) by index | **amortized O(1)** | O(n) | O(n) | insertion order | the default sequence type |
| `Dictionary<K,V>` | **O(1) average** | O(1) average | n/a | O(1) by key | no guaranteed order | the default map |
| `HashSet<T>` | n/a | O(1) average | n/a | **O(1)** | none | uniqueness and membership |
| `SortedDictionary<K,V>` | O(log n) | O(log n) | n/a | O(log n) | sorted by key | a red-black tree |
| `SortedList<K,V>` | O(log n) | **O(n)** | O(n) | O(log n) | sorted | array-backed; compact; fast if you add in sorted order |
| `SortedSet<T>` | n/a | O(log n) | n/a | O(log n) | sorted | range queries (`GetViewBetween`) |
| `Queue<T>` / `Stack<T>` | peek O(1) | O(1) | n/a | O(n) | FIFO / LIFO | |
| `PriorityQueue<TElement,TPriority>` | peek O(1) | O(log n) | n/a | n/a | by priority (min-heap) | .NET 6+; **not** stable |
| `LinkedList<T>` | O(n) | O(1) at the ends | **O(1) given a node** | O(n) | insertion order | rarely the right choice (poor cache locality) |

**Rule of thumb:** use `List<T>` for sequences, `Dictionary` for lookups and `HashSet` for "have I seen this?". Anything else needs a reason.

```csharp
// ❌ O(n·m): Contains on a List inside a loop
var vipIds = vipCustomers.Select(c => c.Id).ToList();
var vipOrders = orders.Where(o => vipIds.Contains(o.CustomerId)).ToList();

// ✅ O(n + m): a HashSet makes each lookup O(1)
var vipIdSet = vipCustomers.Select(c => c.Id).ToHashSet();
var vipOrders2 = orders.Where(o => vipIdSet.Contains(o.CustomerId)).ToList();
```

With 10,000 orders and 10,000 VIPs, the first version does about 100 million comparisons and the second does about 20,000 hash lookups.

---

## 4.2 How `List<T>` works

- It's backed by an **array**. When the array is full, it allocates a new one of **double the capacity** (4 → 8 → 16 → …) and copies the elements across.
- **Pre-size it** when you know the count: `new List<Order>(rows.Count)`. That avoids several reallocations and the garbage they leave behind.
- `RemoveAt(0)` and `Insert(0, x)` shift every element, so they're O(n). If you need cheap removal at the front, use a `Queue<T>`.
- `CollectionsMarshal.AsSpan(list)` gives you a `Span<T>` over the internal array for hot loops. Don't add or remove items while you hold that span.

---

## 4.3 How `Dictionary<K,V>` works and the hashing contract

1. The dictionary calls `key.GetHashCode()`, which picks a **bucket**.
2. Inside the bucket, it compares entries with `key.Equals(other)`.

**The contract:**

- If `a.Equals(b)`, then `a.GetHashCode() == b.GetHashCode()`. Unequal objects *may* share a hash code.
- A key's hash code must **not change while the key is in the dictionary**.

```csharp
// ❌ A mutable key: after the mutation the entry is "lost"
var key = new MutableKey { Id = 1 };
var map = new Dictionary<MutableKey, string> { [key] = "one" };
key.Id = 2;
Console.WriteLine(map.ContainsKey(key)); // False: it's looking in the wrong bucket now

class MutableKey
{
    public int Id { get; set; }
    public override bool Equals(object? obj) => obj is MutableKey k && k.Id == Id;
    public override int GetHashCode() => Id;
}
```

**Use immutable keys:** `string`, numbers, `Guid` or `readonly record struct`.

```csharp
// Custom comparison without changing the key type
var byEmail = new Dictionary<string, Customer>(StringComparer.OrdinalIgnoreCase);

// One lookup instead of two
if (map.TryGetValue(id, out var customer)) { /* ... */ }     // ✅
if (map.ContainsKey(id)) { var c = map[id]; }                 // ❌ double lookup

// Pre-size
var index = new Dictionary<Guid, Order>(capacity: orders.Count);
```

---

## 4.4 Read-only, immutable and frozen collections

| Type | Guarantee | Cost | Use for |
|---|---|---|---|
| `IReadOnlyList<T>` / `IReadOnlyCollection<T>` / `IReadOnlyDictionary<K,V>` | **an interface view only**; the underlying object can still change | free | API return types: "you may not modify this" |
| `ReadOnlyCollection<T>` (`list.AsReadOnly()`) | a wrapper; reflects changes to the source | a small wrapper | exposing an internal list safely |
| `ImmutableArray<T>`, `ImmutableList<T>`, `ImmutableDictionary<K,V>` | **never changes**; "modifications" return new instances | `ImmutableList`/`Dictionary` are trees (slower reads); `ImmutableArray` is a fast array | shared state across threads, snapshots |
| `FrozenDictionary<K,V>` / `FrozenSet<T>` (.NET 8+) | never changes; **optimized for reads** | slow to *create*, the **fastest** lookups | lookup tables built once at startup (config, routing, mappings) |

```csharp
using System.Collections.Frozen;

private static readonly FrozenDictionary<string, decimal> VatRates =
    new Dictionary<string, decimal> { ["DE"] = 0.19m, ["FR"] = 0.20m, ["NL"] = 0.21m }
        .ToFrozenDictionary(StringComparer.OrdinalIgnoreCase);
```

### What to accept and what to return

- **Accept** the *most general* type you need: `IEnumerable<T>` if you only iterate once, `IReadOnlyCollection<T>` if you need `Count`, and `IReadOnlyList<T>` if you need an indexer.
- **Return** a *specific, materialized, read-only* type such as `IReadOnlyList<T>`. Don't return a lazy `IEnumerable<T>` that depends on a resource that may be disposed (a `DbContext` or a stream).

---

## 4.5 Iterators and `yield`

```csharp
public static IEnumerable<int> Evens(int max)
{
    for (int i = 0; i <= max; i += 2)
        yield return i;          // lazy: produced one at a time as the caller iterates
}
```

- The compiler generates a state machine, just as it does for `async`.
- **Nothing runs until the first `MoveNext()` call**, and that includes argument validation:

```csharp
// ❌ The exception is thrown when someone iterates, not when the method is called
public static IEnumerable<T> TakeEvery<T>(IEnumerable<T> source, int step)
{
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(step);
    int i = 0;
    foreach (var item in source)
        if (i++ % step == 0) yield return item;
}

// ✅ Validate eagerly, then iterate lazily in a local function
public static IEnumerable<T> TakeEveryFixed<T>(IEnumerable<T> source, int step)
{
    ArgumentNullException.ThrowIfNull(source);
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(step);
    return Iterate();

    IEnumerable<T> Iterate()
    {
        int i = 0;
        foreach (var item in source)
            if (i++ % step == 0) yield return item;
    }
}
```

---

## 4.6 LINQ to Objects

### Deferred vs immediate execution

| Deferred (it builds a pipeline) | Immediate (it runs now) |
|---|---|
| `Where`, `Select`, `SelectMany`, `OrderBy`, `GroupBy`, `Skip`, `Take`, `Distinct`, `Join`, `Chunk`… | `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`, `Count`, `Sum`, `Max`, `First`, `Single`, `Any`, `All`, `Aggregate`, `foreach` |

```csharp
var threshold = 100;
var expensive = products.Where(p => p.Price > threshold);  // nothing has run yet
threshold = 1_000;                                         // the lambda captured the *variable*
Console.WriteLine(expensive.Count());                      // uses 1_000!
```

### The multiple-enumeration bug

```csharp
IEnumerable<Order> pending = LoadOrdersLazily().Where(o => o.Status == OrderStatus.Pending);

if (pending.Any())                        // enumeration #1: runs the source query
{
    Console.WriteLine(pending.Count());   // enumeration #2: runs it again
    foreach (var o in pending) { }        // enumeration #3: and again
}
```

If the source is a database query, an HTTP call or a file, you pay for it three times, and you may even get **different results** each time. **Fix:** materialize once with `var pending = ….ToList();`. Analyzer **CA1851** flags this pattern, so turn it on.

### Performance notes

| Prefer | Over | Why |
|---|---|---|
| `Any()` | `Count() > 0` | `Any` stops at the first element |
| `FirstOrDefault(predicate)` | `Where(predicate).FirstOrDefault()` | same result, more direct (and the analyzer suggests it) |
| `Single` only when "exactly one" is a real invariant | `First` | `Single` must scan everything to prove uniqueness |
| `TryGetNonEnumeratedCount` | `Count()` when you only want a cheap count | avoids enumerating |
| a plain loop in proven hot paths | LINQ | avoids enumerator and delegate allocations (modern .NET LINQ is heavily optimized, so measure first) |

> `FirstOrDefault()` on value types returns `default`, such as `0` or a zero `Guid`, not `null`. Use the overload `FirstOrDefault(defaultValue)` or check with `Any` first.

### Newer operators worth knowing

```csharp
// .NET 6
var batches = ids.Chunk(100);                                     // IEnumerable<int[]>
var cheapest = products.MinBy(p => p.Price);
var oneEach = products.DistinctBy(p => p.Category);

// .NET 9
var perCategory = products.CountBy(p => p.Category);              // IEnumerable<KeyValuePair<string,int>>
var totals = orders.AggregateBy(o => o.CustomerId, 0m, (sum, o) => sum + o.Total);
foreach (var (index, item) in products.Index()) { /* index + item */ }

// .NET 10: LeftJoin / RightJoin; .NET 11: FullJoin and tuple-returning Join/GroupJoin overloads
var withOrders = customers.LeftJoin(orders,
    c => c.Id, o => o.CustomerId,
    (c, o) => new { c.Name, OrderId = o?.Id });
```

---

## 4.7 `IEnumerable<T>` vs `IQueryable<T>`: the most important LINQ concept

| | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| Lambdas compile to | **delegates** (`Func<T,bool>`): executable code | **expression trees** (`Expression<Func<T,bool>>`): *data* that describes the code |
| Executed by | your process, in memory | a **provider** (EF Core) that *translates* the expression to SQL |
| Where filtering happens | in memory, after loading | **in the database** |

```csharp
// ✅ IQueryable: the WHERE and TOP run in SQL, so only 20 rows come back
var recent = await db.Orders
    .Where(o => o.CustomerId == customerId)
    .OrderByDescending(o => o.CreatedAt)
    .Take(20)
    .ToListAsync(ct);

// ❌ AsEnumerable()/ToList() too early: the WHOLE table is loaded, then filtered in memory
var recentBad = db.Orders.AsEnumerable()
    .Where(o => o.CustomerId == customerId)
    .Take(20)
    .ToList();

// ❌ An IEnumerable-typed parameter silently switches to in-memory filtering
IEnumerable<Order> source = db.Orders;               // static type is now IEnumerable
var bad = source.Where(o => o.Total > 100).ToList(); // Enumerable.Where → full table scan into memory
```

- **A custom C# method inside an `IQueryable` query** (`.Where(o => IsVip(o))`) can't be translated to SQL. EF Core throws a "could not be translated" exception, or evaluates it on the client only in the final projection. Rewrite it as a translatable expression.
- **Know where the boundary is.** Everything *before* `ToList()`, `ToListAsync()` or `AsEnumerable()` runs in SQL; everything after runs in memory. Make that point deliberate.
- You'll go deep on this with EF Core in Phase 2 (N+1 queries, projections, split queries).

---

## 4.8 Equality and comparers

```csharp
// Sorting by multiple keys
var sorted = orders.OrderBy(o => o.CustomerName).ThenByDescending(o => o.Total);

// A custom equality comparer (e.g., case-insensitive SKUs)
var skuComparer = EqualityComparer<Product>.Create(
    (a, b) => string.Equals(a?.Sku, b?.Sku, StringComparison.OrdinalIgnoreCase),
    p => StringComparer.OrdinalIgnoreCase.GetHashCode(p.Sku));
var unique = products.Distinct(skuComparer);
```

- `OrderBy` is **stable**: equal keys keep their original order. `List<T>.Sort` and `Array.Sort` are **not** stable.
- `IComparer<T>` defines *ordering*; `IEqualityComparer<T>` defines *equality plus hashing*.

---

## Interview questions

<details><summary><b>1. What's the time complexity of <code>List.Contains</code>, <code>HashSet.Contains</code> and <code>Dictionary</code> lookup?</b></summary>

`List.Contains` is O(n). The other two are O(1) on average, and degrade toward O(n) when many keys share a hash code. `SortedDictionary` lookups are O(log n).
</details>

<details><summary><b>2. How does a Dictionary find a value? What happens if you mutate a key?</b></summary>

`GetHashCode` selects a bucket, then `Equals` resolves collisions within it. If you mutate a key so that its hash changes, the entry sits in the "wrong" bucket and the dictionary can't find it. Keys must be immutable.
</details>

<details><summary><b>3. What is deferred execution? Give a bug it causes.</b></summary>

LINQ operators like `Where` and `Select` build a pipeline that runs only when the sequence is enumerated. Bugs it causes: multiple enumeration that re-runs an expensive source, captured variables that change before execution, and enumerating after the underlying resource (such as a `DbContext`) has been disposed.
</details>

<details><summary><b>4. IEnumerable vs IQueryable?</b></summary>

`IEnumerable` executes delegates in memory. `IQueryable` carries expression trees that a provider translates, for example into SQL. Calling `AsEnumerable`, `ToList` or typing a variable as `IEnumerable` too early moves filtering into memory and can load entire tables.
</details>

<details><summary><b>5. IReadOnlyList vs ImmutableList vs FrozenDictionary?</b></summary>

- **`IReadOnlyList`** is a read-only *view*; the source can still change.
- **Immutable collections** never change, and "modifying" one returns a new instance (good for thread-safe snapshots).
- **Frozen collections** are immutable and optimized for the fastest reads, at a high creation cost. Use them for startup lookup tables.
</details>

<details><summary><b>6. Why does iterator argument validation run late, and how do you fix it?</b></summary>

An iterator's body (including its argument checks) runs only on the first `MoveNext`. Do the validation in a non-iterator wrapper method that returns the result of a local iterator function.
</details>

---

## Exercises

1. **Quadratic hunt:** find a `List.Contains` (or `.Any(x => x.Id == …)`) inside a loop in your work code. Replace it with a `HashSet` or `Dictionary` and benchmark both versions with 10,000 items.
2. **Multiple enumeration:** enable `CA1851` (`dotnet_diagnostic.CA1851.severity = warning` in `.editorconfig`) and fix every hit in one project.
3. **IQueryable:** with EF Core and SQLite, write a query correctly and then "accidentally" with `AsEnumerable()`. Log the generated SQL (`.LogTo(Console.WriteLine)`) and compare.
4. **Frozen vs Dictionary:** benchmark lookups of 50 string keys in a `Dictionary` vs a `FrozenDictionary`, and measure the creation cost of each.
5. **Write LINQ yourself:** implement your own `MyWhere`, `MySelect` and `MyTake` with `yield`. You'll understand LINQ forever after.

## Self-check

- [ ] I pick collections by their operations and complexity, not by habit.
- [ ] I know the `GetHashCode`/`Equals` contract and why keys must be immutable.
- [ ] I can explain deferred execution and spot multiple enumeration.
- [ ] I can explain `IEnumerable` vs `IQueryable` and where the in-memory boundary is.
- [ ] I know when to use read-only, immutable and frozen collections.

## Further reading

- Microsoft Learn: *Collections and data structures*, *LINQ overview*, *Expression trees*, *System.Collections.Frozen*.
- Jon Skeet, the *Edulinq* blog series (reimplementing LINQ to Objects). Old, but brilliant for understanding.
