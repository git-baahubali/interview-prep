# Python Fundamentals

Core Python behaviour that interviewers use to check whether you understand the language, not just the syntax.

**Questions:** 8

---

## Easy

---

## Q1: What are mutable and immutable data types in Python?

### Answer

Mutable objects can be modified in place after creation. Immutable objects cannot be changed once created — any "modification" creates a new object.

**Mutable examples:**
- List
- Dictionary
- Set
- `bytearray`
- Most custom class instances

**Immutable examples:**
- Integer
- Float
- String
- Tuple
- Boolean
- `frozenset`
- `bytes`

**Why it matters:** mutability decides whether a function can change a caller's data, whether an object can be a dictionary key, and whether an object is safe to use as a default argument.

### Example

```python
numbers = [1, 2, 3]
print(id(numbers))
numbers.append(4)          # same object mutated
print(id(numbers))         # identical id

text = "abc"
print(id(text))
text += "d"                # new object created
print(id(text))            # different id
```

### Interview Follow-ups

- Why are strings immutable? (Hashability, safe sharing/interning, thread safety, and cheap slicing guarantees.)
- Can a tuple contain mutable objects? (Yes — the tuple's *references* are fixed, the referenced objects are not.)
- Why can tuples sometimes be dictionary keys while lists cannot? (Dict keys must be hashable; a tuple is hashable only if all its elements are.)

---

## Q2: What is the difference between a list, a tuple, and a set?

### Answer

| Feature | List | Tuple | Set |
|---|---|---|---|
| Mutable | Yes | No | Yes (elements must be immutable) |
| Ordered | Yes | Yes | No (insertion order not guaranteed) |
| Duplicates | Allowed | Allowed | Removed |
| Indexing | Yes | Yes | No |
| Hashable | No | Yes (if elements are) | No (`frozenset` is) |
| Typical use | Homogeneous sequence you will modify | Fixed record / return value / dict key | Membership tests, dedup, set algebra |
| Membership test | O(n) | O(n) | O(1) average |

**Explanation:** a list is a growable array optimised for iteration and appends. A tuple is a fixed-size record — using it signals "this will not change" and unlocks hashability. A set is a hash table with no values, so it trades ordering and indexing for O(1) membership.

### Example

```python
rows = [1, 2, 2, 3]

point = (10, 20)                 # fixed record
cache = {point: "visited"}       # tuple as dict key

unique = set(rows)               # {1, 2, 3}
print(2 in unique)               # O(1) average
```

### Interview Follow-ups

- Why is `x in some_set` far faster than `x in some_list`? (A set is a hash table: hash `x` once, jump to a bucket, compare a handful of candidates — O(1) average. A list has no index, so membership is a linear scan comparing element by element — O(n). At 1M elements that is the difference between microseconds and tens of milliseconds, which matters inside a loop.)
- When would you prefer a tuple over a list even though both work? (When the sequence is a fixed-size record rather than a collection — coordinates, a `(model, version)` pair, a function returning multiple values. Immutability makes it hashable, so it can be a dict key or set member, and it signals intent to the reader: "this will not grow." Tuples are also marginally smaller and faster to construct.)

---

## Q3: What is the difference between `is` and `==`?

### Answer

`==` compares **values** (delegates to `__eq__`). `is` compares **identity** — whether two names point to the exact same object in memory.

Use `is` only for singletons: `None`, `True`, `False`, and sentinel objects. Use `==` for everything else.

### Example

```python
a = [1, 2, 3]
b = [1, 2, 3]

print(a == b)      # True  -> same contents
print(a is b)      # False -> two distinct objects

x = 256
y = 256
print(x is y)      # True  -> small ints are cached (CPython detail)

x = 1000
y = 1000
print(x is y)      # often False -> do not rely on this
```

**Common misconception:** `is` is *not* a faster `==`. Integer/string comparisons that "work" with `is` only do so because of CPython caching (small-int cache, string interning) and can silently break.

### Interview Follow-ups

- Why is `if value is None` preferred over `if value == None`? (`None` is a singleton, so identity is the exact semantics you want, and `is` cannot be overridden. `==` calls `__eq__`, which a class can define to return `True` against `None` or to raise — and NumPy arrays and pandas Series return an *array* of booleans, so `if arr == None` raises `ValueError: truth value is ambiguous`. `is` is faster and cannot be subverted.)
- What does implementing `__eq__` without `__hash__` do to a class? (Python sets `__hash__ = None`, making instances unhashable — they can no longer go in a set or be used as dict keys. The reason is the invariant that equal objects must have equal hashes; since your `__eq__` no longer matches the default identity hash, Python refuses to guess. Fix it by defining `__hash__` over the same fields, or use `@dataclass(frozen=True)`, which generates both consistently.)

---

## Intermediate

---

## Q4: What are `*args` and `**kwargs`?

### Answer

`*args` collects extra **positional** arguments into a tuple. `**kwargs` collects extra **keyword** arguments into a dict. They exist so a function can accept a variable number of arguments and so wrappers can forward arguments they do not know about.

### Example

```python
def log_call(fn):
    def wrapper(*args, **kwargs):        # accept anything
        print(f"calling {fn.__name__} with {args} {kwargs}")
        return fn(*args, **kwargs)       # forward everything (unpacking)
    return wrapper

@log_call
def embed(texts, model="text-embedding-3-small", batch_size=32):
    return len(texts)

embed(["a", "b"], batch_size=8)
# calling embed with (['a', 'b'],) {'batch_size': 8}
```

Note the two directions: in a **definition** `*`/`**` pack arguments; in a **call** they unpack them.

### Interview Follow-ups

- What do arguments after a bare `*` in a signature mean? (Keyword-only arguments.)
- How would you make a decorator preserve the wrapped function's name and docstring? (`functools.wraps`.)

---

## Q5: Why is a mutable default argument dangerous?

### Answer

Default argument values are evaluated **once**, when the function is defined — not on every call. A mutable default is therefore shared across all calls and accumulates state.

### Example

```python
# Bug
def add_item(item, bucket=[]):
    bucket.append(item)
    return bucket

print(add_item(1))   # [1]
print(add_item(2))   # [1, 2]  <-- same list reused

# Fix
def add_item(item, bucket=None):
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket
```

The same trap applies to `{}`, `set()`, and to expensive objects like a model client created in the default.

### Interview Follow-ups

- Why is `def f(x, cache={})` sometimes used deliberately? (Cheap memoisation — but `functools.lru_cache` is clearer.)

---

## Q6: What is the difference between shallow copy and deep copy?

### Answer

A **shallow copy** creates a new outer container but reuses references to the inner objects. A **deep copy** recursively copies inner objects too, so the result shares nothing with the original.

| | Shallow copy | Deep copy |
|---|---|---|
| How | `list(x)`, `x[:]`, `x.copy()`, `copy.copy(x)` | `copy.deepcopy(x)` |
| Nested objects | Shared | Duplicated |
| Cost | Cheap, O(n) references | Expensive, proportional to whole object graph |
| Risk | Mutating a nested object affects both | None (but slow; cycles handled via memo) |

### Example

```python
import copy

config = {"model": "gpt", "stop": ["\n", "###"]}

shallow = copy.copy(config)
shallow["stop"].append("END")
print(config["stop"])        # ['\n', '###', 'END']  <-- leaked

deep = copy.deepcopy(config)
deep["stop"].append("END")
print(config["stop"])        # unchanged
```

### Interview Follow-ups

- How does `deepcopy` handle circular references? (It keeps a `memo` dict mapping `id(original) -> copy`. Before copying an object it checks the memo; on a second encounter it returns the existing copy instead of recursing, so `a.ref = b; b.ref = a` terminates and the copied pair points at each other rather than at the originals. You can pass your own `memo` to share state across several `deepcopy` calls.)
- Why is deep-copying a large DataFrame or a loaded model usually the wrong call? (Cost and intent. A deep copy duplicates every underlying buffer, so a 4 GB DataFrame needs another 4 GB and a loaded model duplicates all its weights — often an OOM. Models frequently hold unpicklable handles (CUDA contexts, file descriptors, thread locks) that `deepcopy` either mangles or refuses. Prefer explicit narrow copies (`df[cols].copy()`), treat the model as read-only shared state, or reload from the checkpoint if you genuinely need an independent instance.)

---

## Advanced

---

## Q7: What is the GIL, and how does it affect CPU-bound vs I/O-bound work?

### Answer

The **Global Interpreter Lock** is a mutex in CPython that allows only one thread to execute Python bytecode at a time.

**Why it exists:** it makes CPython's reference-counting memory management thread-safe cheaply, and keeps the C API simple.

**How it behaves:**
- **CPU-bound Python code** gets no speedup from threads — threads take turns holding the GIL, and context switching adds overhead.
- **I/O-bound code** scales well with threads, because the GIL is released while a thread waits on a socket, disk, or `time.sleep`.
- Extension code (NumPy, PyTorch, Polars, `tokenizers`) releases the GIL during heavy C/CUDA work, so those operations do run in parallel.

**What to use instead:**

| Workload | Tool |
|---|---|
| Many HTTP/LLM API calls | `asyncio` or `ThreadPoolExecutor` |
| Pure-Python CPU work | `multiprocessing` / `ProcessPoolExecutor` |
| Numeric/tensor CPU work | Vectorised NumPy / PyTorch (already parallel) |
| Serving models | Multiple worker processes behind a load balancer |

Python 3.13+ ships an experimental free-threaded (no-GIL) build, but the GIL remains the default assumption in interviews.

### Example

```python
from concurrent.futures import ThreadPoolExecutor

# Good use of threads: 50 concurrent LLM/API calls, all I/O-bound.
with ThreadPoolExecutor(max_workers=16) as pool:
    results = list(pool.map(call_llm, prompts))
```

### Interview Follow-ups

- Why does an ML inference server usually run multiple processes rather than multiple threads? (Python-level request handling — deserialisation, validation, tokenisation, post-processing, prompt assembly — is CPU-bound and serialised by the GIL, so threads do not add throughput for it. Separate processes each get their own interpreter and GIL, giving true parallelism; this is exactly what `gunicorn -w 4` or `uvicorn --workers 4` does. The nuance worth stating: the heavy tensor math inside PyTorch/NumPy already *releases* the GIL and parallelises internally, so the processes are there to scale the Python glue, not the matmuls — and the cost is that each worker holds its own copy of the model in memory, which is why GPU serving usually pairs a small number of workers with request batching instead.)
- Does the GIL make your code automatically thread-safe? (No — a `+=` on a shared counter is still not atomic at the source level, and higher-level invariants can still tear.)

---

## Q8: What is the difference between a generator and a list, and when do you use `yield`?

### Answer

A list materialises all elements in memory immediately. A generator produces elements **lazily**, one at a time, holding only the current item and its own local state.

| | List | Generator |
|---|---|---|
| Memory | O(n) | O(1) |
| Evaluation | Eager | Lazy |
| Re-iterable | Yes | No (exhausted once consumed) |
| Indexing / `len()` | Yes | No |
| Best for | Small data reused repeatedly | Streams, large files, pipelines |

**When to use `yield`:** streaming a file or dataset too large for RAM, building a transformation pipeline, or forwarding tokens from a streaming LLM response.

### Example

```python
def read_chunks(path, size=1000):
    """Stream a huge corpus without loading it into memory."""
    with open(path, encoding="utf-8") as f:
        buffer = []
        for line in f:
            buffer.append(line)
            if len(buffer) == size:
                yield "".join(buffer)
                buffer = []
        if buffer:
            yield "".join(buffer)

for chunk in read_chunks("corpus.txt"):
    index(embed(chunk))          # constant memory
```

Generator expressions give the same laziness inline: `sum(len(x) for x in docs)`.

### Interview Follow-ups

- What does `yield from` do? (Delegates to another iterable: `yield from sub` is shorthand for looping over `sub` and yielding each item, but it also forwards `send()`, `throw()`, and `close()` to the sub-generator and evaluates to its `return` value. Practically, it is how you compose generator pipelines — `for path in paths: yield from read_chunks(path)` streams every chunk of every file without materialising anything.)
- What happens if you iterate an exhausted generator? (It simply stops — no error, no restart.)
- How do generators relate to `async def` coroutines? (Both use the same suspend-and-resume machinery — a coroutine is essentially a generator whose suspension points are `await` rather than `yield`, which is why early asyncio used `yield from` before `async`/`await` existed. The difference is what drives them: a generator advances when someone calls `next()`, a coroutine when the event loop sees its awaited I/O complete. Use a generator to stream data lazily, a coroutine to overlap waiting. `async def` containing `yield` gives an async generator — the right tool for streaming LLM tokens from an API.)

---
