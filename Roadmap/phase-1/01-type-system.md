# 1. The C# Type System

> Phase 1 · Week 1 · [← Phase index](README.md)

**Why this matters:** Many subtle bugs come from how types behave when you assign, compare, pass or store them. Typical examples are "why didn't my change stick?", "why is this slow?" and "why is this null?". Interviews start here because this topic shows quickly whether you *understand* C# or have only memorized it.

---

## 1.1 Value types vs reference types

| | Value types | Reference types |
|---|---|---|
| Examples | `int`, `double`, `bool`, `char`, `decimal`, `DateTime`, `Guid`, enums, `struct`, `record struct`, tuples (`ValueTuple`) | `class`, `record` (class), `string`, arrays, delegates, `object` |
| The variable holds | the data itself | a *reference* (a managed pointer) to an object on the heap |
| `b = a` | copies the data | copies the reference, so both variables point to the **same** object |
| Default value | all bits zero (`0`, `false`, null fields inside) | `null` |
| Nullable? | only as `Nullable<T>` (`int?`) | yes |
| Inheritance | implicitly sealed; derives from `System.ValueType` | single inheritance from a class |

```csharp
var s1 = new PointStruct(1, 2);
var s2 = s1;            // full copy of the data
s2.X = 100;
Console.WriteLine(s1.X); // 1

var c1 = new PointClass(1, 2);
var c2 = c1;            // copy of the reference: same object
c2.X = 100;
Console.WriteLine(c1.X); // 100

struct PointStruct(int x, int y) { public int X = x; public int Y = y; }
class PointClass(int x, int y) { public int X = x; public int Y = y; }
```

### Myth: "value types live on the stack"

Where data lives depends on **where it is declared**, not on whether it's a value type:

- A value-type **local variable** usually lives on the stack (or in a CPU register).
- A value-type **field of a class** lives *inside that object, on the heap*.
- The elements of an `int[]` are stored inline *inside the array object, on the heap*.
- A local that a lambda captures, or that is used across an `await` or `yield`, is *hoisted* into a heap-allocated closure or state-machine object.

The correct statement is: *"Value types are stored inline wherever they are declared. Reference-type objects always live on the heap, and variables hold references to them."* (See Eric Lippert's blog post "The Stack Is an Implementation Detail".)

### How arguments are passed

All arguments are passed **by value** by default. For a reference type, the *reference* is what gets copied:

```csharp
var list = new List<int> { 1 };
Mutate(list);    // list is now [1, 2]: the shared object changed
Reassign(list);  // list is still [1, 2]: only the method's local copy of the reference changed

static void Mutate(List<int> items) => items.Add(2);
static void Reassign(List<int> items) => items = new List<int>();
```

| Modifier | Meaning | Typical use |
|---|---|---|
| `ref` | The callee can read **and reassign** the caller's variable | Rare; swapping values or high-performance code |
| `out` | The callee **must** assign it before returning | The Try-pattern: `int.TryParse(s, out var n)` |
| `in` / `ref readonly` | Pass by reference, **read-only** | Large structs, to avoid copying them |

### Equality

| Type | `==` | `Equals` |
|---|---|---|
| `class` | reference equality (same object?) unless the operator is overloaded | reference equality unless overridden |
| `string` | **value** equality (ordinal) | value equality |
| `record` / `record struct` | value equality (generated) | value equality (generated) |
| `struct` | **not defined** unless you write it | `ValueType.Equals` compares fields. It's fast for "bit-comparable" structs but uses **reflection** when the struct has reference fields or padding |

**Rule:** a struct that you compare or use as a dictionary key should implement `IEquatable<T>`. Making it a `record struct` does this for you.

---

## 1.2 Boxing and unboxing

**Boxing** converts a value type to `object` or to an interface. The runtime **allocates a heap object** and **copies** the value into it. **Unboxing** checks the type and copies the value back out.

```csharp
int n = 42;
object boxed = n;          // heap allocation + copy
int back = (int)boxed;     // type check + copy
// long wrong = (long)boxed; // InvalidCastException: you must unbox to the exact type
```

Boxing costs an allocation, a copy and GC pressure. In a hot loop it can dominate the run time.

### Hidden boxing: where it hides

| Code | Why it boxes | Fix |
|---|---|---|
| `ArrayList`, `Hashtable` | they store `object` | `List<T>`, `Dictionary<TKey,TValue>` |
| Passing a struct to an `object` parameter | conversion to `object` | a generic method `void Log<T>(T value)` |
| `IComparable c = 5;` (casting a struct to an interface) | interfaces are reference types | a generic constraint `where T : IComparable<T>` |
| A struct dictionary key without `IEquatable<T>` | `Equals(object)` boxes the argument | implement `IEquatable<T>` or use `record struct` |
| `string.Format("{0}", 42)`, `Console.WriteLine("{0}", x)` | `params object[]` | interpolated strings `$"{x}"`. Interpolated string handlers (C# 10+) avoid boxing |
| `foreach` over a `List<T>` declared as `IEnumerable<T>` | the struct enumerator gets boxed | iterate the concrete type in hot code |

### Mutating a boxed copy

```csharp
var counter = new Counter();
object boxed = counter;
((Counter)boxed).Increment();               // unboxes into a temporary copy and increments the copy
Console.WriteLine(((Counter)boxed).Value);  // 0

struct Counter { public int Value; public void Increment() => Value++; }
```

**See it yourself:** paste code into SharpLab, switch the output to *IL*, and search for `box`. In a benchmark, `[MemoryDiagnoser]` shows the allocations.

---

## 1.3 Structs: when and how

Consider a struct only when **all** of these are true (from the Framework Design Guidelines):

- It logically represents a **single value**, like `int`, `DateTime` or `Money`.
- It is **small**. A rule of thumb is about 16 bytes or less.
- It is **immutable**.
- It **won't be boxed** often.

Otherwise, use a class.

### Struct pitfalls

**1. Mutable structs in collections.** The indexer returns a *copy*:

```csharp
var points = new List<PointStruct> { new(1, 2) };
// points[0].X = 10;     // CS1612: you can't modify the return value; it's a copy
var p = points[0];
p.X = 10;
points[0] = p;           // copy out, modify, write back
```

**2. Defensive copies.** If you call a method on a non-`readonly` struct that is stored in a `readonly` field or passed as an `in` parameter, the compiler **silently copies** the struct first, because it can't prove that the method doesn't mutate it. Fix this with `readonly struct`, or mark individual members `readonly`.

**3. `default` bypasses constructors.** `default(Money)`, `new Money[10]` and uninitialized fields all produce all-zero instances. If your constructor validates `Currency != null`, a `default` value still has `Currency == null`. Either design so that all-zero is valid, or check for it.

**4. Copy cost.** Passing a 64-byte struct by value copies 64 bytes on every call. Use `in` or `ref readonly`, or make it a class.

### `readonly struct`

The compiler enforces immutability, so no defensive copies are ever needed:

```csharp
public readonly struct Temperature(double celsius)
{
    public double Celsius { get; } = celsius;
    public double Fahrenheit => Celsius * 9 / 5 + 32;
}
```

### `ref struct`: stack-only types

`Span<T>` and `ReadOnlySpan<T>` are `ref struct`s. Their rules guarantee that they never escape to the heap:

- They can't be boxed, stored in a class field or array, or captured by a lambda.
- Since C# 13, a `ref struct` local *can* appear in `async` methods and iterators, as long as it isn't used **across** an `await` or `yield return`.
- Since C# 13, `ref struct`s can implement interfaces, and generic code can accept them with the `allows ref struct` anti-constraint.

---

## 1.4 Records

```csharp
public record Customer(string Name, string Email);                    // reference type with value equality
public readonly record struct Money(decimal Amount, string Currency); // value type, immutable, value equality
```

The compiler generates:

| Feature | Example |
|---|---|
| Properties from positional parameters | `init`-only for `record` and `readonly record struct`; `get; set;` for a plain `record struct` |
| Value equality | `Equals`, `GetHashCode`, `==`, `!=`, `IEquatable<T>` |
| `ToString()` | `Customer { Name = Ann, Email = ann@x.com }` |
| `Deconstruct` | `var (name, email) = customer;` |
| `with` expressions (non-destructive mutation) | `var moved = customer with { Email = "new@x.com" };` |

### Record pitfalls

**Equality is shallow.** Collection members are compared **by reference**:

```csharp
public record Order(string Id, List<string> Items);

var a = new Order("1", ["x"]);
var b = new Order("1", ["x"]);
Console.WriteLine(a == b); // False: two different List instances
```

Fix this by overriding `Equals(Order? other)` and `GetHashCode()`, or keep collections out of records whose equality matters.

**`with` is a shallow copy.** The new record shares the *same* `List` instance as the original.

**Records are a poor fit for entities.** An entity has *identity*: two `Order` objects with the same `Id` are the same order even if a field differs. Value equality contradicts that. Use classes for entities, and use records for DTOs, messages, events and value objects.

---

## 1.5 Choosing: class, struct or record?

| You need… | Use |
|---|---|
| An entity with identity and a lifecycle (`Order`, `Customer`) | `class` |
| A DTO, an API request or response, a message or event, or other immutable data | `record` |
| A small immutable value (`Money`, `SeatId`, `Coordinates`) | `readonly record struct` |
| A high-performance buffer view that stays on the stack | `ref struct` |
| A fixed set of variants (`Success`/`NotFound`/`Conflict`) | C# 15 `union` or a `closed` record hierarchy (see [guide 5](05-modern-csharp.md)) |

---

## 1.6 Strings

- A `string` is a **reference type** that is **immutable** and has **value equality** (`==` compares the content).
- Every "modification" creates a new string, so `s += x` inside a loop is O(n²). Use `StringBuilder`, `string.Join` or `string.Concat`, or a single interpolated string.
- Identical literals are *interned* (they share one instance). Never rely on reference equality for strings.
- **Always be explicit about how you compare strings:**

```csharp
// Identifiers, keys, protocol values, file names: ordinal
bool same = string.Equals(a, b, StringComparison.OrdinalIgnoreCase);
var lookup = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);

// Text shown to users (sorting names): culture-aware
names.Sort(StringComparer.CurrentCulture);
```

> A classic bug: under Turkish culture (`tr-TR`), `"FILE".ToLower()` returns `"fıle"` with a dotless *ı*. Use `ToLowerInvariant()` or an ordinal comparison for anything that isn't shown to users.

**Allocation-free parsing with spans:**

```csharp
using System.Globalization;

ReadOnlySpan<char> line = "2027-01-10;Jazz Night;49.90";
int first = line.IndexOf(';');
ReadOnlySpan<char> date = line[..first];                       // a slice, not a new string
decimal price = decimal.Parse(line[(line.LastIndexOf(';') + 1)..], CultureInfo.InvariantCulture);
```

**Raw string literals** (`"""`) are handy for JSON or SQL. **UTF-8 literals** (`"OK"u8`) give you a `ReadOnlySpan<byte>` with no allocation.

---

## 1.7 Nullable reference types (NRT)

**The problem:** `NullReferenceException` is the most common runtime error in .NET.
**The solution:** with NRT, the compiler's flow analysis warns you wherever a `null` could reach a dereference.

```xml
<!-- Directory.Build.props: apply to every project -->
<PropertyGroup>
  <Nullable>enable</Nullable>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```

```csharp
public sealed class CustomerService(ICustomerRepository repository)
{
    public async Task<string> GetDisplayNameAsync(Guid id, CancellationToken ct)
    {
        Customer? customer = await repository.FindAsync(id, ct); // may be null
        if (customer is null)
            return "Unknown";

        return customer.Name; // flow analysis knows customer isn't null here
    }
}
```

### Annotations

| Syntax / attribute | Meaning |
|---|---|
| `string?` | may be null |
| `string` | should never be null |
| `x!` | null-forgiving: "trust me" (it suppresses the warning but **doesn't check anything**) |
| `[NotNullWhen(true)] out T? value` | not null when the method returns `true` (the Try-pattern) |
| `[MaybeNull]`, `[AllowNull]`, `[DisallowNull]`, `[NotNull]` | fine-grained contracts for generics and properties |
| `[MemberNotNull(nameof(_field))]` | after this method runs, `_field` isn't null |
| `[DoesNotReturn]` | the method always throws, so the code after it is unreachable |

```csharp
public bool TryFindSeat(string code, [NotNullWhen(true)] out Seat? seat)
{
    seat = _seats.FirstOrDefault(s => s.Code == code);
    return seat is not null;
}

if (TryFindSeat("A1", out var seat))
    Console.WriteLine(seat.Row); // no warning
```

### Important truths about NRT

1. **NRT is compile-time only.** Nothing changes at runtime. JSON deserialization, EF Core, reflection and external callers can still hand you `null`, so **validate at system boundaries** with `ArgumentNullException.ThrowIfNull(x)` and model validation.
2. **Treat every `!` as a code-review question.** Legitimate uses are rare: test code, or after a check that the compiler can't see.
3. **Migrating legacy code:** add `#nullable enable` file by file, starting with domain and leaf projects, annotate the public APIs first, and then switch the project-level setting on.

---

## 1.8 Enums

- An enum is backed by an integer type (`int` by default), and **any** integer can be cast to it: `(OrderStatus)999` compiles and runs. **Validate external input** with `Enum.IsDefined<TEnum>(value)` or with a `switch` whose default case throws.
- Use `[Flags]` for combinations. Give the members power-of-two values (`1, 2, 4, 8`). In modern .NET, `HasFlag` is a JIT intrinsic and fast.
- **Persisted enums:** always assign explicit values, because reordering the members silently corrupts numeric data that's already stored. Consider storing enums as strings in JSON (`JsonStringEnumConverter`) and in the database.

---

## 1.9 Generics essentials

- **Generics are real at runtime.** `List<int>` and `List<string>` are distinct types. For *value-type* arguments the JIT generates specialized code, so there is no boxing. *Reference-type* arguments share one implementation.
- **Static fields are per closed type:** `Cache<int>.Count` and `Cache<string>.Count` are separate fields.

| Constraint | Meaning |
|---|---|
| `where T : class` / `class?` | a reference type (non-nullable / nullable) |
| `where T : struct` | a non-nullable value type |
| `where T : notnull` | any non-nullable type |
| `where T : unmanaged` | a struct with no references (can be used with pointers) |
| `where T : new()` | has a public parameterless constructor |
| `where T : BaseClass, IInterface` | inherits from the class and implements the interface |
| `where T : allows ref struct` | `T` may be a `ref struct` (C# 13) |

### Variance

```csharp
IEnumerable<string> strings = ["a", "b"];
IEnumerable<object> objects = strings;      // OK: IEnumerable<out T> is covariant

Action<object> printAny = o => Console.WriteLine(o);
Action<string> printString = printAny;      // OK: Action<in T> is contravariant

// List<object> list = new List<string>();  // CS0029: List<T> is invariant (it both accepts and returns T)

object[] array = new string[1];
array[0] = 42; // compiles, but throws ArrayTypeMismatchException: array covariance is unsafe
```

---

## Interview questions

<details><summary><b>1. Where do value types live in memory?</b></summary>

They're stored inline wherever they're declared: a local usually goes on the stack, a field inside an object goes on the heap, and array elements live inside the array on the heap. Locals that are captured by a closure or used across an `await` are hoisted to the heap. Reference-type objects always live on the heap.
</details>

<details><summary><b>2. What is boxing? Why does it matter? Give hidden examples.</b></summary>

Boxing converts a value type to `object` or to an interface: the runtime allocates on the heap and copies the value. It adds allocation and GC cost. Hidden examples: `ArrayList`, passing a struct to an `object` parameter, casting a struct to an interface, a struct dictionary key without `IEquatable<T>`, and `params object[]` APIs.
</details>

<details><summary><b>3. When would you choose class, struct, record and record struct?</b></summary>

- **class:** an entity with identity, or a type with behavior and a lifecycle.
- **record:** immutable data with value equality (DTOs, messages).
- **readonly record struct:** a small immutable value (Money, IDs).
- **struct:** small, single-value, rarely boxed types where performance matters.
</details>

<details><summary><b>4. Why are mutable structs dangerous?</b></summary>

Copies are made implicitly on assignment, on method calls, by indexers and when boxing. A modification often changes a copy instead of the original. That produces bugs like CS1612 compile errors, silent defensive copies, and lost updates through a boxed copy.
</details>

<details><summary><b>5. What does <code>==</code> do for a class, a struct, a record and a string?</b></summary>

- **class:** reference equality, unless the operator is overloaded.
- **struct:** not defined unless you write it.
- **record:** generated value equality.
- **string:** ordinal value equality.
</details>

<details><summary><b>6. You pass a List to a method that does <code>items = new List&lt;int&gt;()</code>. What does the caller see?</b></summary>

Nothing changes for the caller, because the method only reassigned its own copy of the reference. With `ref List<int> items`, the caller's variable would be reassigned too.
</details>

<details><summary><b>7. Do nullable reference types change runtime behavior?</b></summary>

No. They are compile-time annotations plus flow analysis. You still have to validate at the boundaries, because deserializers, reflection and external callers can pass `null`.
</details>

<details><summary><b>8. What's wrong with <code>object[] a = new string[1]; a[0] = 1;</code>?</b></summary>

Array covariance lets the assignment compile, but storing an `int` into what is really a `string[]` throws `ArrayTypeMismatchException` at runtime. Generic interfaces use declaration-site variance (`out`/`in`), which is type-safe.
</details>

<details><summary><b>9. How do you compare strings for identifier lookups, and why?</b></summary>

Use `StringComparison.Ordinal` or `OrdinalIgnoreCase`, or a matching `StringComparer` for dictionaries. It's fast and culture-independent, so you avoid bugs like the Turkish-I problem.
</details>

<details><summary><b>10. Two records have equal properties, but one of them is a <code>List</code>. Are they equal?</b></summary>

Not unless they share the same list instance. Generated equality compares each member with `EqualityComparer<T>.Default`, which for a `List<T>` means reference equality.
</details>

---

## Exercises

1. **Boxing hunt:** in SharpLab, write code with five *hidden* boxing cases from the table above, find each `box` IL instruction, and fix it.
2. **Money:** implement `Money` as a `readonly record struct` with validation and `operator +` that throws when the currencies differ. Show the `default(Money)` problem in a test and decide how to handle it.
3. **The record equality trap:** write a failing test that proves the `List` equality pitfall, then fix it by overriding `Equals` and `GetHashCode`.
4. **NRT migration:** enable `<Nullable>` and warnings-as-errors in a real project (at work or a pet project). Fix 20 warnings and write down the recurring patterns.
5. **A benchmark preview:** compare a `Dictionary<PointStruct, int>` keyed by a plain struct with one keyed by a `readonly record struct`. Record the time and allocations with BenchmarkDotNet (set it up as described in [guide 9](09-labs-and-capstone.md)).

## Self-check

- [ ] I can explain value vs reference semantics without mentioning "stack vs heap".
- [ ] I can list five sources of hidden boxing and how to fix each.
- [ ] I know three struct pitfalls and when to use `readonly struct`.
- [ ] I know what records generate and two record pitfalls.
- [ ] I always compare strings with an explicit `StringComparison`.
- [ ] NRT is on in my capstone, with zero warnings.

## Further reading

- Microsoft Learn: *Value types*, *Records*, *Nullable reference types*, *Choosing between class and struct*.
- *C# in Depth* (Jon Skeet), the chapters on types and generics.
- Eric Lippert, *"The Stack Is an Implementation Detail"* (blog, parts 1–2).
