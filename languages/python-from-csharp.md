# Python from a C# developer's perspective

Quick translation guide for the Python idioms that show up most often in your code and recur enough to be worth a permanent mental map. Extracted from the inline-comment scaffolding in `local-rag/` (kb_index.py + kb_search.py) when those files were prepared for public release — the public versions dropped these comments since they're noise for native Python readers, but they're useful for you when reading or writing Python.

## Modules and tooling

| Python | Equivalent in C# / .NET |
|---|---|
| `argparse` | `System.CommandLine` — CLI flag parser, idiomatic |
| `hashlib` | `System.Security.Cryptography` (returns hex digest strings by default, not byte arrays) |
| `json` | `System.Text.Json` — `json.dumps(obj) -> str`, `json.loads(str) -> obj` |
| `logging` | `Microsoft.Extensions.Logging` — hierarchical by dotted name (`module.sub.thing`) |
| `os` | `Environment` for env vars (`os.environ`), `System.IO` mixed in |
| `re` | `System.Text.RegularExpressions` — `re.compile()` returns a reusable `Pattern` object |
| `sqlite3` | `Microsoft.Data.Sqlite` (Python ships it; .NET needs the NuGet) |
| `struct` | `BitConverter` but explicit about format and endianness |
| `urllib.request` | `HttpClient` (lighter than `requests`, slightly clunkier — no fluent builder) |
| `pathlib.Path` | `Path` + `FileInfo` combined into one object |
| `typing` | Type annotations — runtime no-op, used by static checkers like `mypy` |

## Language idioms

**Failing fast on import errors:**

```python
try:
    import sqlite_vec
except ImportError:
    sys.exit("sqlite-vec not installed. Run: pip install sqlite-vec")
```

C# equivalent: failing in `Main` with a clear message instead of throwing deep in the call stack. Python doesn't have a compile-time check for missing packages — you find out at import.

**`with` statement = `using`:**

```python
with urllib.request.urlopen(req) as resp:
    data = resp.read()
```

C# `using` block. Calls `resp.__exit__` on scope exit (same as `Dispose()`).

**Tuple unpacking and `*args`:**

```python
struct.pack("%df" % len(values), *values)
```

The `*values` is the equivalent of C# `params` — splats a list into positional args.

**`enumerate(seq, start=1)`:**

```python
for idx, line in enumerate(lines, start=1):
    ...
```

C# equivalent: `seq.Select((item, i) => (i + 1, item))`. Built-in to Python; in C# you have to compose it yourself.

**Inner functions (closures):**

```python
def chunk_markdown(text):
    current_lines = []
    def flush(end_line):
        # closes over current_lines — captures by REFERENCE, mutations show through
        ...
```

Same as C# local functions. **Critical difference:** Python closures capture by reference, not value. Mutating an outer list from inside the closure is visible to the outer scope. (C# does this too for local variables, but the binding rules are more explicit.)

**`zip()`:**

```python
for c, vec in zip(chunks, vectors):
    ...
```

C# `chunks.Zip(vectors, (c, v) => (c, v))`. Pairs two sequences elementwise; stops at the shorter one.

## Collections

**List comprehensions ≈ LINQ Select then ToList:**

```python
[x / norm for x in vec]
```

≈ `vec.Select(x => x / norm).ToList()`. Returns a new list eagerly.

**Generator expressions ≈ deferred LINQ:**

```python
sum(x * x for x in vec)
```

The `(x * x for x in vec)` is a generator — lazy, no list allocation. Like an `IEnumerable<>` you only enumerate once. Wrapping in `sum()` consumes it immediately. C# equivalent: `vec.Sum(x => x * x)`.

**Set comprehensions ≈ `new HashSet<>(query)`:**

```python
existing_hashes = {row[0] for row in con.execute("SELECT hash FROM chunks")}
```

≈ `new HashSet<string>(rows.Select(r => r.Hash))`. The `{...}` with no key:value pairs is a set literal.

**Dict comprehensions ≈ `ToDictionary`:**

```python
{k: f(v) for k, v in items.items()}
```

≈ `items.ToDictionary(kv => kv.Key, kv => f(kv.Value))`.

**`defaultdict` ≈ `ConcurrentDictionary.GetOrAdd`:**

```python
from collections import defaultdict
scores = defaultdict(float)  # missing keys auto-initialize to 0.0
scores[doc_id] += 1.0  # works even if doc_id wasn't in scores
```

`defaultdict(factory)` auto-initializes missing keys via `factory()` on first access. The C# parallel is `GetOrAdd` but `defaultdict` is more transparent — you just access keys, and missing ones get filled in.

**Generic typing:**

```python
from typing import List, Tuple, Dict, Optional
stack: List[Tuple[int, str]] = []  # equivalent to Stack<(int, string)>
```

Python's `typing` module gives you the analogues of C# generics. `List[X]` ≈ `List<X>`, `Dict[K, V]` ≈ `Dictionary<K, V>`, `Optional[X]` ≈ `X?` (nullable).

## Strings, bytes, and encoding

**`str` vs `bytes` is hard-distinguished:**

```python
json.dumps({"x": 1})                    # returns str
json.dumps({"x": 1}).encode("utf-8")    # returns bytes (suitable for HTTP body)
resp.read()                              # returns bytes
json.loads(resp.read())                  # OK — loads accepts bytes OR str
```

C# is more forgiving — `string` and `byte[]` interoperate through `Encoding.UTF8.GetBytes()` / `GetString()`. Python forces you to be explicit.

**Raw string literals:**

```python
HEADER_RE = re.compile(r"^(#{1,6})\s+(.*)$")
```

The `r"..."` prefix means "treat backslashes literally" — equivalent to C# `@"..."`. Use it for regex patterns so you don't have to double every backslash.

## Sorting

**`sorted()` vs `.sort()`:**

```python
ranked = sorted(scores.items(), key=lambda kv: kv[1], reverse=True)  # returns new list
some_list.sort(key=lambda x: x.score)  # mutates in-place
```

C# parallel: `OrderBy()` returns a new sequence; `List<T>.Sort()` mutates. Python is the same split.

**Lambda syntax:**

```python
key=lambda kv: kv[1]
```

≈ `kv => kv[1]` in C#. Python lambdas can only contain a single expression — no block bodies. Use a `def` if you need more.

## Entry-point guards

```python
if __name__ == "__main__":
    main()
```

The `__name__` global gets set to `"__main__"` when the file is run directly, and to the module name (`"kb_index"`) when imported. C# equivalent: it's the difference between `Program.Main` running because the assembly was launched as an executable vs. being loaded as a library.

## Time

**Use `perf_counter` for elapsed time, not `time.time`:**

```python
t0 = time.perf_counter()
# ... work ...
elapsed_ms = (time.perf_counter() - t0) * 1000
```

`time.time()` can jump (NTP sync, daylight saving, manual clock change). `time.perf_counter()` is monotonic and high-resolution — the equivalent of `Stopwatch.GetTimestamp()`.

## Path objects

**`__file__` and pathlib:**

```python
SCRIPT_DIR = Path(__file__).resolve().parent       # script's own directory
REPO_ROOT = Path(__file__).resolve().parents[2]   # two dirs up
```

`Path(__file__)` is the script's location. `.resolve()` makes it absolute. `.parent` walks one level up; `.parents[N]` walks N levels. C# equivalent: `Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location)` and then more `GetDirectoryName` calls.

## Things that don't quite map

- **No `var`** — Python is dynamically typed. Variables don't have declared types; you can assign different types to the same name. Type annotations (`x: int = 5`) are hints for tools, not enforced at runtime.
- **No `null` — but `None`** is the equivalent. Checks: `if x is None:` (identity check, idiomatic) vs `if x == None:` (equality, also works but less Pythonic).
- **No `interface`** — Python uses duck typing. If it walks like a duck, it's a duck. `abc.ABC` exists for explicit abstract base classes if you really want them.
- **No method overloading by signature** — Python resolves by name only. You can simulate overloading with default args, `*args`, `**kwargs`, or `singledispatch` from `functools`.
- **No `await` without `async def`** — async functions are explicit, not opt-in per call.

## When you're stuck

Most "how does C# do this in Python" questions are well-covered by:

- `dir(obj)` — list all attributes on an object (for poking around)
- `help(obj)` — show docs
- `type(obj)` — runtime type
- `isinstance(obj, type)` — type check
- `vars(obj)` — instance `__dict__` (like reflection over fields)
