# NASA "Power of 10" Rules -- Adapted for Interpreted Languages

Adapted from Gerard Holzmann's original C rules for use with Python,
JavaScript, TypeScript, Go, Java, Ruby, PHP, and similar languages.
The core principles (bounded resources, verifiable code, defensive checks)
remain identical -- the implementation details change.

---

## Rule 1: Simple Control Flow -- No Recursion, No goto Equivalents

**No recursion. Avoid goto-like constructs and fall-through switch.**

Recursion creates unbounded stack depth and cyclic call graphs that are
hard to reason about. Avoid `switch` fall-through (languages that require
explicit `break` are especially prone to bugs here). Use `if`/`else if`
chains or pattern matching instead.

### Python -- Non-compliant

```python
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)  # recursion
```

### Python -- Compliant

```python
def factorial(n):
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result
```

### JavaScript -- Non-compliant

```javascript
switch (status) {
    case "pending":
        notify();       // falls through to "active" -- bug-prone
    case "active":
        process();
        break;
}
```

### JavaScript -- Compliant

```javascript
if (status === "pending") {
    notify();
} else if (status === "active") {
    process();
}
```

---

## Rule 2: All Iteration Must Be Provably Bounded

**Every loop must operate on a finite, known-size collection or have an
explicit upper bound.**

Use pagination, chunking, or explicit limits. If iterating over external
input, enforce a maximum. Never write `while (true)` without a guaranteed
exit within a fixed number of iterations.

### Python -- Non-compliant

```python
while True:
    data = fetch_next()
    if not data:
        break  # no upper bound if fetch_next never returns None
    process(data)
```

### Python -- Compliant

```python
MAX_ITERATIONS = 10_000
for _ in range(MAX_ITERATIONS):
    data = fetch_next()
    if not data:
        break
    process(data)
```

### JavaScript -- Non-compliant

```javascript
let i = 0;
while (arr[i] !== undefined) {  // unbounded if arr has no undefined
    handle(arr[i]);
    i++;
}
```

### JavaScript -- Compliant

```javascript
const MAX_LEN = 10000;
for (let i = 0; i < Math.min(arr.length, MAX_LEN); i++) {
    handle(arr[i]);
}
```

---

## Rule 3: Bound Memory Usage

**Ensure memory consumption has a provable upper bound. Work on finite
pages, chunks, or streams -- not unbounded collections.**

Do not load an entire dataset of unknown size into memory. Use pagination
for database queries, streaming for file I/O, and fixed-size buffers for
network input.

### Python -- Non-compliant

```python
rows = cursor.fetchall()  # loads entire table into memory
for row in rows:
    process(row)
```

### Python -- Compliant

```python
PAGE_SIZE = 500
while True:
    rows = cursor.fetchmany(PAGE_SIZE)
    if not rows:
        break
    for row in rows:
        process(row)
```

### JavaScript -- Non-compliant

```javascript
const data = await fetch("/api/records").then(r => r.json());
// could be millions of records
```

### JavaScript -- Compliant

```javascript
const PAGE_SIZE = 100;
let page = 0;
let batch;
do {
    batch = await fetch(`/api/records?limit=${PAGE_SIZE}&offset=${page * PAGE_SIZE}`)
        .then(r => r.json());
    batch.forEach(process);
    page++;
} while (batch.length === PAGE_SIZE && page < 1000);
```

---

## Rule 4: Functions Max 60 Lines

**No function or method longer than approximately 60 lines.**

If a function does too many things, break it into focused sub-functions.
Each function should handle one concern. This makes code easier to test,
review, and verify.

### Non-compliant

A 200-line function that fetches data, validates it, transforms it,
writes to a file, and sends a notification.

### Compliant

```python
def handle_report():
    data = fetch_data()
    validated = validate(data)
    transformed = transform(validated)
    write_output(transformed)
    send_notification()
```

Each sub-function stays under 60 lines.

---

## Rule 5: Fail Loudly -- Assertions and Exceptions

**Never silently swallow errors. Use assertions for preconditions. Let
exceptions propagate until they can be properly handled.**

In languages with exceptions, do not catch and ignore them. Use `assert`
for internal invariants. For external input, validate and raise descriptive
errors. Report failures using monitoring tools (Sentry, etc.).

### Python -- Non-compliant

```python
def get_user(user_id):
    try:
        return db.query(user_id)
    except Exception:
        return None  # silent failure -- caller has no idea what went wrong
```

### Python -- Compliant

```python
def get_user(user_id):
    assert isinstance(user_id, int), "user_id must be an integer"
    assert user_id > 0, "user_id must be positive"
    return db.query(user_id)  # let exceptions propagate
```

### JavaScript -- Non-compliant

```javascript
function parseConfig(raw) {
    try {
        return JSON.parse(raw);
    } catch {
        return {};  // silent fallback hides corrupt input
    }
}
```

### JavaScript -- Compliant

```javascript
function parseConfig(raw) {
    if (typeof raw !== "string" || raw.length === 0) {
        throw new Error("parseConfig: input must be a non-empty string");
    }
    return JSON.parse(raw); // parse errors propagate naturally
}
```

---

## Rule 6: Smallest Possible Variable Scope

**No global variables. Declare data in the narrowest scope possible.**

Confine variables to functions, blocks, or modules. If state must be shared,
use classes, closures, or module-level encapsulation with clear access
boundaries.

### Python -- Non-compliant

```python
result_cache = {}  # module-level global, mutable by anyone

def compute(x):
    result_cache[x] = x * 2
    return result_cache[x]
```

### Python -- Compliant

```python
def compute(x):
    return x * 2

class ComputeCache:
    def __init__(self):
        self._cache = {}

    def compute(self, x):
        if x not in self._cache:
            self._cache[x] = x * 2
        return self._cache[x]
```

---

## Rule 7: Never Silently Discard Errors or Return Values

**Check every meaningful return value. Let exceptions bubble up until you
can properly recover from or report them.**

Do not catch exceptions too early. Do not ignore Promise rejections. Do not
discard function return values that indicate success or failure.

### JavaScript -- Non-compliant

```javascript
await saveRecord(data);  // returns { ok: boolean } but never checked
```

### JavaScript -- Compliant

```javascript
const result = await saveRecord(data);
if (!result.ok) {
    throw new Error(`saveRecord failed: ${result.error}`);
}
```

### Python -- Non-compliant

```python
os.remove(filepath)  # if file doesn't exist, exception is swallowed nowhere
```

This is actually fine -- the exception will propagate. The anti-pattern is:

```python
try:
    os.remove(filepath)
except:
    pass  # bare except, silent discard
```

### Python -- Compliant

```python
try:
    os.remove(filepath)
except FileNotFoundError:
    logger.warning("File already removed: %s", filepath)
except OSError as e:
    raise RuntimeError(f"Failed to remove {filepath}") from e
```

---

## Rule 8: Minimize Build/Transpile Complexity

**Keep the toolchain as simple as possible. Every transpiler or build step
must justify its existence by solving more problems than it introduces.**

Stay close to the language standard. Avoid exotic preprocessing or code
generation that hides logic. If you use a transpiler (e.g., TypeScript,
Babel), stay mainstream and keep configuration minimal.

### Non-compliant

A project with Babel + 15 plugins + custom Webpack loaders + PostCSS +
a Dart-to-JS transpiler + a macro preprocessor.

### Compliant

TypeScript compiled with `tsc` using strict mode. CSS processed with a
single standard tool. No custom code generation steps.

---

## Rule 9: Don't Mutate Arguments; Use Type Safety

**Functions should not modify their input arguments unless that is the
explicit documented purpose. Use type annotations and immutability
features.**

Return new values instead of mutating inputs. Avoid pass-by-reference
tricks. Use language features like `Object.freeze()`, `typing.NamedTuple`,
`readonly`, or `const` to enforce immutability.

### JavaScript -- Non-compliant

```javascript
function addDiscount(order, pct) {
    order.total *= (1 - pct);  // mutates the caller's object
    return order;
}
```

### JavaScript -- Compliant

```javascript
function addDiscount(order, pct) {
    return { ...order, total: order.total * (1 - pct) };
}
```

### Python -- Non-compliant

```python
def normalize(items):
    for i in range(len(items)):
        items[i] = items[i].strip().lower()  # mutates the input list
```

### Python -- Compliant

```python
def normalize(items):
    return [item.strip().lower() for item in items]
```

Use type annotations everywhere:

```python
def normalize(items: list[str]) -> list[str]:
    return [item.strip().lower() for item in items]
```

```typescript
function normalize(items: readonly string[]): string[] {
    return items.map(s => s.trim().toLowerCase());
}
```

---

## Rule 10: Use All Available Linters and Analyzers; Zero Warnings

**Enable every available linting and static analysis tool at maximum
strictness. Fix all warnings before release.**

Treat warnings as errors. Configure CI to block merges with warnings.
Use spell-checkers on identifiers to catch typos.

### Recommended tools by language

| Language   | Linters / Analyzers                              |
|------------|--------------------------------------------------|
| Python     | ruff, mypy, pylint, bandit                       |
| JavaScript | eslint (strict config), typescript --strict       |
| TypeScript | eslint, tsc --strict, typescript-eslint           |
| Go         | go vet, staticcheck, golangci-lint                |
| Java       | SpotBugs, PMD, Error Prone, CheckStyle            |
| Ruby       | rubocop, brakeman                                 |
| PHP        | phpstan, psalm                                    |
| Rust       | clippy (deny all warnings), cargo check            |

### Non-compliant

```bash
# "It's just a warning, ship it"
eslint src/ --max-warnings 50
```

### Compliant

```bash
eslint src/ --max-warnings 0
mypy src/ --strict
ruff check src/
```
