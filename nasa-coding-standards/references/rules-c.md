# NASA JPL "Power of 10" Rules -- C/C++

Original rules by Gerard Holzmann, NASA Jet Propulsion Laboratory (2006).
These target C code for safety-critical embedded systems where static
verification must be tractable.

---

## Rule 1: Simple Control Flow

**No goto, setjmp, longjmp. No direct or indirect recursion.**

All code must use simple, structured control flow. The call graph must be
acyclic (a DAG) so static analyzers can verify stack depth and termination.

### Non-compliant

```c
int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1); // recursion -- unbounded stack
}
```

### Compliant

```c
int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; i++) {
        result *= i;
    }
    return result;
}
```

---

## Rule 2: Fixed Upper Bounds on All Loops

**Every loop must have a statically provable upper bound on iterations.**

A checking tool must be able to prove that a preset maximum iteration count
cannot be exceeded. If the bound cannot be proved statically, the rule is
violated.

### Non-compliant

```c
int i = 0;
while (array[i] != 0) {   // no provable upper bound
    process(array[i]);
    i++;
}
```

### Compliant

```c
#define MAX_LEN 256
for (int i = 0; i < MAX_LEN; i++) {
    if (array[i] == 0) break;
    process(array[i]);
}
```

---

## Rule 3: No Dynamic Memory Allocation After Initialization

**Do not call malloc, calloc, realloc, or free after program initialization.**

All memory must be statically allocated or allocated only during a clearly
defined initialization phase. This prevents fragmentation, allocation failures,
and use-after-free bugs at runtime.

### Non-compliant

```c
void store_data(int size) {
    int *buf = malloc(size * sizeof(int)); // runtime allocation
    if (!buf) return;
    // ...
    free(buf);
}
```

### Compliant

```c
#define MAX_SIZE 256
static int buf[MAX_SIZE];

void store_data(void) {
    // use buf directly -- fixed size, no runtime allocation
}
```

---

## Rule 4: Functions Max 60 Lines

**No function longer than approximately 60 lines of code.**

One line per statement, one line per declaration. This is roughly one printed
page. Short functions are easier to understand, test, and verify as a unit.

### Non-compliant

A function spanning 150+ lines that handles parsing, validation, transformation,
and output in a single block.

### Compliant

Break the work into focused sub-functions:

```c
void process_all(void) {
    preprocess();
    validate();
    transform();
    output();
}
```

Each sub-function handles one concern and stays under 60 lines.

---

## Rule 5: Minimum 2 Assertions Per Function

**The assertion density must average at least 2 assertions per function.**

Assertions check preconditions, postconditions, parameter validity, and loop
invariants. They must be side-effect free boolean tests. When an assertion
fails, the function must take an explicit recovery action (e.g., return an
error code). Trivially true assertions like `assert(true)` do not count.

### Non-compliant

```c
int get_element(int *arr, size_t size, size_t idx) {
    return arr[idx]; // no checks at all
}
```

### Compliant

```c
int get_element(int *arr, size_t size, size_t idx) {
    assert(arr != NULL);
    assert(idx < size);
    if (arr == NULL || idx >= size) return -1;
    return arr[idx];
}
```

---

## Rule 6: Smallest Possible Data Scope

**Declare every data object at the smallest possible level of scope.**

Do not use file-scope or global variables when a local variable suffices.
This limits unintended coupling and makes side effects explicit.

### Non-compliant

```c
int status_flag; // global -- visible everywhere

void set_status(int f) {
    status_flag = f;
}
```

### Compliant

```c
void set_status(int f) {
    int status_flag = f;
    // use status_flag only within this function
}
```

If shared state is truly necessary, confine it to the narrowest file scope
with `static`.

---

## Rule 7: Check All Return Values and Validate Parameters

**The return value of every non-void function must be checked by the caller.
Every function must validate its input parameters.**

If a return value is intentionally unused, cast it to `(void)` to document
the decision.

### Non-compliant

```c
int launch(int velocity, int time) {
    int distance;
    calculate_trajectory(velocity, time, &distance); // unchecked
    return distance;
}
```

### Compliant

```c
int launch(int velocity, int time) {
    assert(velocity > 0);
    assert(time > 0);
    int distance;
    int status = calculate_trajectory(velocity, time, &distance);
    if (status != 0) return -1;
    return distance;
}
```

---

## Rule 8: Limit Preprocessor Usage

**The preprocessor may only be used for header file inclusion and simple
macro definitions.**

No token pasting, no variadic macros, no recursive macros. All macros must
expand into complete syntactic units. Conditional compilation should be rare
and justified.

### Non-compliant

```c
#define DECLARE_FUNC(name) void func_##name(void) // token pasting
DECLARE_FUNC(init);
```

### Compliant

```c
static inline int square(int x) { return x * x; }
#define MAX_BUFFER 256
#include "utils.h"
```

Replace function-like macros with `static inline` functions for type safety
and debuggability.

---

## Rule 9: Limit Pointer Usage

**No more than one level of pointer dereference. No function pointers.**

Pointer dereference may not be hidden in macros or typedefs. Multiple levels
of indirection make static analysis intractable and increase the risk of
aliasing bugs.

### Non-compliant

```c
int **matrix;                           // double pointer
int (*callback)(int) = some_function;   // function pointer
```

### Compliant

```c
int *row;                               // single level
int result = some_function(5);           // direct call
```

For data structures that seem to require `**`, restructure using flat arrays
with index arithmetic or single-level accessor functions.

---

## Rule 10: Compile With All Warnings; Zero Warnings Allowed

**Enable the compiler's most pedantic warning level. All code must compile
with zero warnings.**

Check code daily with at least one (preferably more) static source code
analyzers. Zero warnings from all tools.

### Non-compliant

```c
int x;
if (x = 5) {            // warning: assignment in condition
    printf("%d\n", x);  // warning: x may be uninitialized
}
```

### Compliant

```c
int x = 0;
if (x == 5) {
    printf("%d\n", x);
}
```

Recommended flags: `-Wall -Wextra -Werror -pedantic` (GCC/Clang).
