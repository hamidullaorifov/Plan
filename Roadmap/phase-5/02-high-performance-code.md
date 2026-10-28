# 2. Writing High-Performance .NET Code

> Phase 5 · Weeks 3–4 · [← Phase index](README.md)

**Why this matters:** Once profiling points at your code, you need a toolbox of proven techniques. Modern .NET gives you allocation-free primitives (`Span<T>`, pools, `SearchValues`), source generators that remove reflection, and a JIT that rewards simple code. **Apply them to measured hot paths only.** Everywhere else, readability wins.

---

## 2.1 The priorities, in order

1. **Do less work:** a better algorithm, caching, fewer database round trips, not computing what isn't used.
2. **Do less I/O:** batching, streaming, projections.
3. **Allocate less:** fewer GCs, especially fewer Gen2 and LOH collections.
4. **Help the JIT:** sealed classes, avoiding interface dispatch in tight loops, spans that let it remove bounds checks.
5. **Go wide:** parallelism and SIMD, only for CPU-bound work, after 1–4.

---

## 2.2 Source generators: remove reflection

### System.Text.Json

```csharp
[JsonSerializable(typeof(EventDto))]
[JsonSerializable(typeof(List<EventDto>))]
[JsonSourceGenerationOptions(JsonSerializerDefaults.Web)]
internal partial class ShopJsonContext : JsonSerializerContext;

// ASP.NET Core: put the generated resolver first
builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, ShopJsonContext.Default));

// Direct use
string json = JsonSerializer.Serialize(dto, ShopJsonContext.Default.EventDto);
```

The result is faster startup, less memory and Native AOT compatibility ([guide 3](03-native-aot-and-startup.md)).

### Regex

```csharp
public static partial class Validators
{
    [GeneratedRegex(@"^[A-Z]{3}$", RegexOptions.CultureInvariant)]
    private static partial Regex CurrencyCode();

    public static bool IsCurrency(string value) => CurrencyCode().IsMatch(value);
}
```

The regex is compiled at build time, with no startup cost, and the generated code is readable.

### Logging

`[LoggerMessage]` (Phase 1, guide 6) avoids boxing and formatting when the level is disabled. In hot paths, logging is often the top allocation source.

---

## 2.3 Strings and text

```csharp
// SearchValues (.NET 8+): vectorized search for a set of characters
private static readonly SearchValues<char> Separators = SearchValues.Create(",;| \t");

static int CountFields(ReadOnlySpan<char> line)
{
    int count = 1, index;
    while ((index = line.IndexOfAny(Separators)) >= 0)
    {
        count++;
        line = line[(index + 1)..];
    }
    return count;
}

// string.Create: write the result directly into the new string's memory (no StringBuilder or char[] copy)
static string ToSlug(string title) =>
    string.Create(title.Length, title, static (span, source) =>
    {
        for (int i = 0; i < source.Length; i++)
            span[i] = char.IsLetterOrDigit(source[i]) ? char.ToLowerInvariant(source[i]) : '-';
    });

// Formatting into spans (ISpanFormattable / IUtf8SpanFormattable)
Span<char> buffer = stackalloc char[64];
if (DateTime.UtcNow.TryFormat(buffer, out int written, "O", CultureInfo.InvariantCulture))
    Process(buffer[..written]);
```

| Prefer | Over |
|---|---|
| `ReadOnlySpan<char>` slicing | `Substring`, `Split` |
| `SearchValues<T>` + `IndexOfAny` | loops checking each character against a list |
| `StringBuilder` (pooled for reuse) | `+=` in loops |
| UTF-8 end to end (`Utf8JsonWriter`, `"…"u8`) | UTF-16 → UTF-8 conversions |
| `string.Equals(a, b, StringComparison.Ordinal)` | `ToLower()` + `==` (allocates, and is culture-sensitive) |

---

## 2.4 Pooling

```csharp
// ArrayPool for temporary buffers (Phase 1, guide 2)
byte[] buffer = ArrayPool<byte>.Shared.Rent(64 * 1024);
try { /* … */ }
finally { ArrayPool<byte>.Shared.Return(buffer); }

// ObjectPool for expensive reusable objects. Package: Microsoft.Extensions.ObjectPool
builder.Services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
builder.Services.AddSingleton(sp => sp.GetRequiredService<ObjectPoolProvider>().CreateStringBuilderPool());

public sealed class ReportFormatter(ObjectPool<StringBuilder> pool)
{
    public string Format(IReadOnlyList<Line> lines)
    {
        var sb = pool.Get();
        try
        {
            foreach (var line in lines) sb.Append(line.Name).Append(';').Append(line.Total).AppendLine();
            return sb.ToString();
        }
        finally { pool.Return(sb); }        // Return clears the builder
    }
}

// RecyclableMemoryStream (Microsoft.IO.RecyclableMemoryStream) instead of MemoryStream for large or frequent buffers
```

**Pooling rules:** return in `finally`, never use an object after returning it, and reset its state. Measure first: pooling small, short-lived objects can be *slower* than allocating, because Gen0 allocation is very cheap.

---

## 2.5 Collections and LINQ in hot paths

| Hot-path technique | Why |
|---|---|
| Pre-size `List<T>` and `Dictionary` | no growth or copying |
| `FrozenDictionary` / `FrozenSet` for read-only lookups | the fastest reads |
| `CollectionsMarshal.AsSpan(list)` for tight loops | no enumerator; bounds checks can be removed |
| `CollectionsMarshal.GetValueRefOrAddDefault` | one dictionary lookup for an "add or update" |
| `foreach` over the concrete type (`List<T>`, an array) | struct enumerators, no boxing |
| Plain loops instead of LINQ where profiled hot | no delegates or enumerators |

```csharp
// Count occurrences with a single lookup per key
var counts = new Dictionary<string, int>(capacity: 1024, StringComparer.Ordinal);
foreach (var word in words)
{
    ref int count = ref CollectionsMarshal.GetValueRefOrAddDefault(counts, word, out _);
    count++;
}
```

> Modern LINQ is heavily optimized (and improves with each release), so don't rewrite LINQ by reflex. Do it where a profile shows it.

---

## 2.6 Async overhead

- An `async` method that usually completes synchronously (a cache hit) can return `ValueTask<T>` to avoid allocating a `Task` (Phase 1, guide 3).
- Avoid `async`/`await` in a trivial pass-through when there's no `try`/`using` around it. Return the task directly. Be careful, though: this changes exception and stack-trace behavior, so it's best only in genuinely hot code.
- **Don't** wrap synchronous CPU work in `Task.Run` in ASP.NET Core.
- Avoid capturing closures in hot callbacks. Use `static` lambdas with state arguments.

---

## 2.7 Streaming large data

```csharp
// ❌ Loads everything into memory: a large LOH allocation and a long time to first byte
app.MapGet("/api/export", async (ShopDbContext db, CancellationToken ct) =>
    Results.Ok(await db.Orders.AsNoTracking().ToListAsync(ct)));

// ✅ Streams rows as JSON as they're read (IAsyncEnumerable is serialized incrementally)
app.MapGet("/api/export", (ShopDbContext db) =>
    db.Orders.AsNoTracking().Select(o => new OrderRow(o.Id, o.Total)).AsAsyncEnumerable());
```

For high-throughput byte processing (parsers, protocols), use **`System.IO.Pipelines`**: it manages buffers for you and handles messages split across reads.

```csharp
static async Task ProcessLinesAsync(PipeReader reader, Func<ReadOnlySequence<byte>, ValueTask> handleLine, CancellationToken ct)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync(ct);
        ReadOnlySequence<byte> buffer = result.Buffer;

        while (TryReadLine(ref buffer, out var line))
            await handleLine(line);

        reader.AdvanceTo(buffer.Start, buffer.End);   // consumed up to Start; examined everything
        if (result.IsCompleted) break;
    }
    await reader.CompleteAsync();
}

static bool TryReadLine(ref ReadOnlySequence<byte> buffer, out ReadOnlySequence<byte> line)
{
    SequencePosition? position = buffer.PositionOf((byte)'\n');
    if (position is null) { line = default; return false; }
    line = buffer.Slice(0, position.Value);
    buffer = buffer.Slice(buffer.GetPosition(1, position.Value));
    return true;
}
```

---

## 2.8 Helping the JIT

- **`sealed` classes:** enable devirtualization and faster type checks. Seal by default (analyzer CA1852 helps).
- **Spans and arrays with `for (int i = 0; i < span.Length; i++)`:** the JIT removes bounds checks it can prove are safe.
- **Avoid `params object[]`, boxing and `dynamic`** in hot paths.
- **Dynamic PGO** (on by default) optimizes hot paths based on runtime behavior, which is another reason to benchmark after warm-up.
- **SIMD:** `Vector128<T>`/`Vector256<T>`/`Vector<T>` and `TensorPrimitives` (System.Numerics.Tensors) for numeric bulk operations. Many BCL methods (`IndexOf`, `SequenceEqual`, `Sum` over spans) are already vectorized, so use them before writing intrinsics.

---

## Interview questions

<details><summary><b>1. How do you reduce allocations in a hot path?</b></summary>

Profile allocations to find the top stacks. Then use spans instead of substrings and arrays, pools for buffers, pre-sized collections, source-generated serialization, regex and logging, struct enumerators instead of LINQ in the hot loop, and avoid closures and boxing. Verify with BenchmarkDotNet's `MemoryDiagnoser` and the production GC metrics.
</details>

<details><summary><b>2. What do source generators give you over reflection?</b></summary>

Code is generated at compile time, so there's no runtime reflection or emit. That gives faster startup, lower memory, compile-time errors and compatibility with trimming and Native AOT. Examples: System.Text.Json, `[GeneratedRegex]`, `[LoggerMessage]`, options validation, Mapperly.
</details>

<details><summary><b>3. When is object pooling a bad idea?</b></summary>

For small, short-lived objects, where Gen0 allocation is cheaper than the pool's synchronization. Also when objects are hard to reset correctly, or when returning them is easy to forget, which causes leaks or shared-state bugs. Measure before and after.
</details>

---

## Exercises

1. **Allocation diet:** take one hot endpoint, reduce its allocations per request by at least 50%, and prove it with a profile plus a load test ([Lab 2](11-labs.md#lab-2-allocation-diet)).
2. Switch an API to source-generated JSON and measure startup time and the allocation rate.
3. Rewrite a CSV import with `System.IO.Pipelines`, and compare its memory against `ReadAllLines`.

## Self-check

- [ ] I use source generators for JSON, regex and logging in performance-sensitive services.
- [ ] I can apply spans, `SearchValues`, pooling and `CollectionsMarshal` where profiles justify it.
- [ ] I stream large responses instead of materializing them.
- [ ] I know the JIT-friendly habits (sealed, simple loops, no boxing).

## Further reading

- Stephen Toub, *"Performance Improvements in .NET 10"* (and the .NET 11/12 editions).
- Microsoft Learn: *System.IO.Pipelines*, *Memory&lt;T&gt; and Span&lt;T&gt; usage guidelines*, *JSON source generation*.
- *Writing High-Performance .NET Code*, 2nd ed. (Ben Watson). Older, but the fundamentals hold.
