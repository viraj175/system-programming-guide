# The Deep Systems Programming Guide — C & Linux

> A master-level, exhaustive guide. Not just what to learn, but *how* to learn it, *why* each detail matters, and *what the machine actually does*.

---

## Table of Contents

1. [C Language — Complete Dissection](#1-c-language--complete-dissection)
2. [Pointers & Memory — The Full Picture](#2-pointers--memory--the-full-picture)
3. [Data Structures — From Scratch, With Trade-offs](#3-data-structures--from-scratch-with-trade-offs)
4. [Algorithms — Implementations & Analysis](#4-algorithms--implementations--analysis)
5. [Linux Internals — Kernel Perspective](#5-linux-internals--kernel-perspective)
6. [System Calls — Complete API Reference](#6-system-calls--complete-api-reference)
7. [Networking — From Socket to Server](#7-networking--from-socket-to-server)
8. [Concurrency — Deep Dive](#8-concurrency--deep-dive)
9. [Memory Systems — Hardware Up](#9-memory-systems--hardware-up)
10. [Build Systems & Linking](#10-build-systems--linking)
11. [Debugging & Profiling — The Full Arsenal](#11-debugging--profiling--the-full-arsenal)
12. [Embedded Systems & RTOS](#12-embedded-systems--rtos)
13. [Security Fundamentals for Systems Programmers](#13-security-fundamentals-for-systems-programmers)
14. [Projects — Complete Specifications](#14-projects--complete-specifications)
15. [Interview Deep-Dive](#15-interview-deep-dive)
16. [Learning Methodology](#16-learning-methodology)

---

## 1. C Language — Complete Dissection

### 1.1 The C Abstract Machine vs Real Hardware

C defines an **abstract machine** — a simplified model of execution. Your code's behavior is defined in terms of this abstract machine, not real hardware. Understanding the gap between the two is critical.

**The abstract machine says:**
- Variables live in memory locations
- Expressions are evaluated in some order (sequence points define where side effects complete)
- Function calls create stack frames

**Real hardware does:**
- Variables live in registers, cache, and memory — not just "memory"
- Instructions are reordered by the compiler and CPU
- Function calls may be inlined (no stack frame)
- Memory accesses may be elided entirely if the compiler proves they're unnecessary

**Practical implication:** You cannot reason about C code as "this line executes, then this line." You must reason in terms of the C standard's guarantees. The compiler will transform your code in ways that preserve observable behavior but may completely rearrange internal operations.

**How to learn this:** Read the generated assembly (`gcc -S`) for simple functions. Compare `-O0` vs `-O2` output. Notice what gets eliminated. This is the single most effective way to internalize that C is not "portable assembly."

---

### 1.2 Integer Types — Complete Reference

C has a bewildering set of integer types. Here's what you actually need to know.

**Exact-width types (from `<stdint.h>` — always use these):**

| Type | Range |
|---|---|
| `int8_t`, `uint8_t` | Exactly 8 bits |
| `int16_t`, `uint16_t` | Exactly 16 bits |
| `int32_t`, `uint32_t` | Exactly 32 bits |
| `int64_t`, `uint64_t` | Exactly 64 bits |

**Pointer-sized types (use for sizes, counts, offsets):**

| Type | Meaning |
|---|---|
| `size_t` | Unsigned, large enough for any object. The type for sizes and counts. |
| `ssize_t` | Signed `size_t`. Used when functions need to return -1 for errors. POSIX, not standard C. |
| `ptrdiff_t` | Signed type for difference between two pointers. |
| `intptr_t`, `uintptr_t` | Integer large enough to hold a pointer. Use for storing pointers in integers. |
| `off_t` | File offset. Signed, typically 64-bit on modern systems. |

**The `size_t` rule:** Any variable that represents a size, count, or index into memory should be `size_t`. Using `int` for sizes is a bug waiting to happen:

```c
// WRONG: What if the length exceeds INT_MAX?
int len = strlen(str);

// CORRECT:
size_t len = strlen(str);
```

**What to notice:** `size_t` is unsigned. This means `for (size_t i = n-1; i >= 0; i--)` is an INFINITE LOOP — when `i` becomes 0 and you decrement, it wraps to `SIZE_MAX`. Use `for (size_t i = n; i-- > 0;)` instead.

**Integer promotion rules** — the cause of countless subtle bugs:

When an operation involves types smaller than `int`, both operands are promoted to `int` first:

```c
uint8_t a = 200, b = 200;
uint8_t c = a + b;   // a and b promoted to int. a + b = 400 (int).
                      // 400 assigned to uint8_t wraps to 144. No UB.
                      // But: if this were uint16_t on a 16-bit machine,
                      // int might be 16 bits, and 400+400 would overflow int -> UB.
```

**Signed integer overflow is UB.** The compiler can assume `x + 1 > x` is always true for signed `x`, and optimize accordingly. This is not theoretical — it causes real security vulnerabilities:

```c
// This check doesn't work for signed int!
int safe_add_signed(int a, int b) {
    if (a + b < a) return -1;  // Compiler may optimize this check AWAY
    return a + b;              // because signed overflow is UB, so a+b >= a must be true!
}
// Correct: check BEFORE overflow, or use unsigned arithmetic.
```

---

### 1.3 Floating Point — The Essential Truths

- `float` is 32-bit IEEE 754. `double` is 64-bit IEEE 754.
- Floating point is **not real numbers**. It's a finite set of discrete values.
- `0.1 + 0.2 != 0.3` in floating point. Never compare floats with `==`.
- NaN is not equal to anything, including itself. Use `isnan()` from `<math.h>`.
- Subnormal numbers (numbers very close to zero) are dramatically slower than normal numbers on many CPUs.

**How to compare floats:**
```c
#include <math.h>
int nearly_equal(double a, double b, double epsilon) {
    return fabs(a - b) < epsilon;
}
```

**What to notice about `-ffast-math`:** This compiler flag enables optimizations that violate IEEE 754 — assuming associativity `(a+b)+c == a+(b+c)`, treating subnormals as zero, etc. Don't use it unless you understand exactly which mathematical properties your code depends on.

---

### 1.4 Structures — Memory Layout Mastery

**Structure padding rules:** Each member must be at an address that is a multiple of its alignment requirement. The struct itself must be padded so its total size is a multiple of the largest member's alignment.

```c
struct demo {
    char  a;    // offset 0, size 1
    // 3 bytes padding (int needs 4-byte alignment)
    int   b;    // offset 4, size 4
    short c;    // offset 8, size 2
    // 6 bytes padding (double needs 8-byte alignment, 
    // next offset after short is 10, not aligned to 8)
    double d;   // offset 16, size 8
};
// sizeof(struct demo) = 24
```

**The optimization trick — sort by descending alignment:**
```c
struct demo_optimized {
    double d;   // offset 0, size 8
    int   b;    // offset 8, size 4
    short c;    // offset 12, size 2
    char  a;    // offset 14, size 1
    // 1 byte padding (total must be multiple of 8)
};
// sizeof(struct demo_optimized) = 16 — saved 8 bytes per instance
```

**`__attribute__((packed))` — when and what it costs:**

The packed attribute forces the compiler to place each field at the next byte after the previous one, ignoring all alignment requirements. Use it ONLY when you must match an exact external binary layout (network packet header, hardware register map, file format). The CPU must do extra work to load unaligned fields:

```c
struct __attribute__((packed)) packet {
    uint8_t  version;   // offset 0
    uint32_t length;    // offset 1 — MISALIGNED. On x86: works but slower.
                        // On ARM: may cause bus fault. Compiler generates
                        // byte-by-byte access code — extremely slow.
};
// sizeof(struct packet) = 5
```

**Flexible array member (C99 — the right way for variable-length structs):**
```c
struct buffer {
    size_t length;
    char   data[];   // Must be last member. No size specified.
};

// Allocation: malloc(sizeof(struct buffer) + actual_data_size)
// buf->data[i] is valid for 0 <= i < actual_data_size
```

Before C99, people used `char data[1]` (the "struct hack") which was technically UB when accessed beyond index 0.

---

### 1.5 Unions — Type Punning and Tagged Unions

**Type punning via union** — valid in C, UB in C++:
```c
union float_bits {
    float    f;
    uint32_t u;
};

union float_bits fb;
fb.f = 1.0f;
printf("%08x\n", fb.u);  // 3f800000 — the IEEE 754 representation

// Write through f, read through u — this is defined behavior in C.
// The same thing via pointer cast ((unsigned int*)&f) is strict aliasing violation.
```

**Tagged unions** — the C way to do sum types (equivalent to `std::variant` in C++ or enums in Rust):
```c
enum type_tag { TYPE_INT, TYPE_FLOAT, TYPE_STRING };

struct value {
    enum type_tag tag;
    union {
        int    i;
        float  f;
        char  *s;
    } data;
};

// The tag tells you which union field is currently active.
// You must maintain this invariant manually — the compiler won't help.
// Using the wrong field is UB.
```

---

### 1.6 Storage Classes — The Full Story

| Storage Class | Keyword | Scope | Lifetime | Initial Value | Where It Lives |
|---|---|---|---|---|---|
| Automatic | (default)/`auto` | Block | Function call | Indeterminate | Stack |
| Static (local) | `static` in function | Block | Entire program | Zero (once) | Data segment |
| Static (global) | `static` on file-scope | Translation unit | Entire program | Zero | Data segment |
| External | `extern` | Global (across files) | Entire program | Zero | Data segment |
| Thread-local | `_Thread_local` | Depends | Thread lifetime | Zero | TLS area |
| Register | `register` | Block | Function call | Indeterminate | Register (hint only) |

**`static` inside a function — the underrated pattern:**
```c
int next_id(void) {
    static int id = 0;   // Initialized exactly once, at program startup
    return ++id;
}
// Returns 1, 2, 3, ... across calls
// id is invisible outside next_id() — no global namespace pollution
// WARNING: not thread-safe. Multiple threads = data race.
```

**`static` on file-scope variables/functions — C's module-private:**
- A `static` global variable is visible only within its translation unit (the `.c` file plus everything it `#include`s)
- A `static` function is callable only within its translation unit
- This is how you prevent symbol collisions between files
- Use it on every function and global variable that doesn't need to be visible to other `.c` files

**`extern` — declaration vs definition:**
```c
// header.h
extern int g_counter;  // Declaration: "this variable EXISTS, defined elsewhere"

// file1.c
#include "header.h"
int g_counter = 0;     // Definition: "this IS where it lives"

// file2.c
#include "header.h"
g_counter++;           // Uses the definition from file1.c
// If you wrote 'int g_counter = 0;' here too -> linker error: multiple definition
```

**The rule:** Definitions go in `.c` files. Declarations (with `extern`) go in `.h` files. Never define a variable in a header — if two `.c` files include it, you get a linker error.

---

### 1.7 `const` — What It Really Means

`const` is a promise YOU make to the compiler. The compiler enforces it at compile time. It does NOT mean the value is immutable at runtime — it can be modified through a non-const alias.

**Reading pointer-const combinations (read right-to-left from the variable):**
```c
const int *p;       // p is a pointer to (const int) — can't modify *p
int const *p;       // SAME THING — const applies to int, not p
int *const p;       // p is a const pointer to int — can't modify p, can modify *p
const int *const p; // p is a const pointer to const int — can't modify either
```

**Const-correctness is a discipline.** Every pointer parameter that doesn't need to modify its target should be `const`. This documents intent, prevents bugs, and enables callers to pass const-qualified data.

---

### 1.8 `volatile` — What It Really Means

`volatile` tells the compiler: "Every access to this variable must happen exactly as written. Do not optimize away reads or writes. Do not reorder accesses relative to other volatile accesses."

**When to use `volatile`:**
1. Memory-mapped hardware registers
2. Variables modified by signal handlers

**When NOT to use `volatile`:**
- **Multithreading.** `volatile` does NOT provide atomicity, memory ordering, or prevent data races. Use `_Atomic` / `stdatomic.h`.

**The canonical hardware register example:**
```c
// Without volatile:
uint32_t status = *STATUS_REG;
while (!(status & READY_BIT)) { /* wait */ }
// Compiler: "status doesn't change in this loop. Infinite loop. Delete check."

// With volatile:
volatile uint32_t *status_reg = (volatile uint32_t *)0x40021000;
while (!(*status_reg & READY_BIT)) { /* wait */ }
// Compiler: "Must re-read from memory every iteration."
```

---

### 1.9 The Preprocessor — Beyond Simple Macros

**The double-evaluation trap:**
```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))
MAX(x++, y++);  // Expands to: ((x++) > (y++) ? (x++) : (y++))
                // One variable increments TWICE.
```

**The fix — `static inline` functions:**
```c
// In a header file:
static inline int max_int(int a, int b) {
    return a > b ? a : b;
}
// Type-safe. No double evaluation. Single evaluation of each argument.
// 'static' prevents multiple-definition errors when included in multiple .c files.
// 'inline' hints the compiler to embed the code at call site instead of call+jump.
```

**When `static inline` vs macro:**
- Always prefer `static inline` when you can
- Use macros only when you need: genericity over types (C doesn't have templates), stringification (`#`), token pasting (`##`), or compile-time constants

**Stringification and token pasting:**
```c
#define STR(x) #x           // STR(hello) -> "hello"
#define JOIN(a,b) a##b      // JOIN(my_, var) -> my_var
```

**`#include` guards — the one true way:**
```c
// my_library.h
#ifndef MY_LIBRARY_H
#define MY_LIBRARY_H

// All declarations here

#endif // MY_LIBRARY_H
// Prevents double-inclusion. Always do this.
// #pragma once is non-standard but universally supported — shorter, but not C standard.
```

---

### 1.10 Compilation Pipeline — A Concrete Walkthrough

When you run `gcc hello.c -o hello`, four stages happen:

**Stage 1: Preprocessing (`gcc -E hello.c`)**
- Expands all `#include` directives (literally copies header contents)
- Expands all macros
- Removes comments
- Processes `#ifdef`, `#ifndef`, `#if` conditional blocks
- Output: `.i` file (pure C with no preprocessor directives)

**Stage 2: Compilation proper (`gcc -S hello.i`)**
- Parses C into an Abstract Syntax Tree
- Performs semantic analysis (type checking)
- Generates assembly language for the target architecture
- Output: `.s` file (assembly text)

**What you notice comparing `-O0` vs `-O2` assembly:**
- `-O0`: Every variable gets a stack slot. Everything loaded/stored in order.
- `-O2`: Variables live in registers. Dead code eliminated. Constants folded. Loops unrolled or vectorized.

**Stage 3: Assembly (`gcc -c hello.s`)**
- Translates assembly text to machine code
- Produces an object file in ELF format
- Output: `.o` file

**What's in the `.o` file:**
- `.text` section: the machine instructions
- `.data` section: initialized global variables
- `.bss` section: placeholder for uninitialized globals
- `.rodata` section: read-only data (string literals, const globals)
- Symbol table: defined symbols and undefined symbols

**Stage 4: Linking (`gcc hello.o -o hello`)**
- Combines one or more `.o` files
- Resolves undefined symbols by finding their definitions
- Brings in libraries (libc is linked by default)
- Produces the final executable

**Static vs Dynamic Linking:**
- **Static library (`.a`):** Code copied into your binary at link time. Self-contained, larger.
- **Shared library (`.so`):** Reference recorded. OS loads library at runtime. Smaller binary, library must be present.

```bash
ldd ./hello  # Lists all shared libraries the binary needs
```

---

### 1.11 Undefined Behavior — A Practical Catalog

UB is not "crashes." UB is: "The compiler may generate any code it wants, because the standard says this situation cannot happen in a valid program."

**Common UB categories you WILL encounter:**

1. **Signed integer overflow** — `INT_MAX + 1` is UB
2. **Out-of-bounds access** — `arr[10]` on a 10-element array
3. **Null pointer dereference**
4. **Use-after-free** — access memory after `free()`
5. **Double free** — `free(p); free(p);`
6. **Strict aliasing violation** — accessing a `float` through an `int*`
7. **Modifying a string literal** — `char *s = "hello"; s[0] = 'H';`
8. **Returning pointer to local variable** — stack frame gone
9. **Division by zero**
10. **Shifting by too many bits** — `x << 32` on a 32-bit type

**Data race** — Two threads access the same memory unsynchronized, and at least one writes.

**How to catch UB:**
```bash
gcc -fsanitize=undefined -fsanitize=address -g program.c -o program
```

---

## 2. Pointers & Memory — The Full Picture

### 2.1 What a Pointer Actually Is

A pointer is an integer interpreted by the CPU as a virtual memory address. On x86-64, it's 8 bytes, but only the lower 48 bits are used for addressing. The upper 16 bits must be copies of bit 47 (this is called a "canonical address").

**Virtual address space on x86-64 Linux:**
```
0x0000000000000000  to  0x00007FFFFFFFFFFF  → User space (128 TB)
0xFFFF800000000000  to  0xFFFFFFFFFFFFFFFF  → Kernel space (128 TB)
```
Addresses in between are non-canonical — accessing them causes a general protection fault.

### 2.2 Pointer Arithmetic — The Type-Aware Dance

When you add `n` to a pointer, the address advances by `n * sizeof(*pointer)` bytes, NOT `n` bytes.

```c
int arr[5] = {10, 20, 30, 40, 50};
int *p = arr;       // p points to arr[0]

p + 1;   // Advances by sizeof(int) = 4 bytes → points to arr[1]
&p[1];   // Same as p + 1 — the [] operator is defined as *(p + i)
*p + 1;  // Different: dereferences p (gets 10) and adds 1 → 11
```

**The fundamental identity:** `arr[i]` is exactly `*(arr + i)`. They compile to the same machine code. The subscript is pure syntactic sugar.

### 2.3 Arrays vs Pointers — The Distinction That Matters

An array is NOT a pointer. An array "decays" to a pointer to its first element in most expressions.

```c
int arr[10];
int *p = arr;

sizeof(arr);   // 40 — size of the entire array (10 * 4)
sizeof(p);     // 8  — size of the pointer itself (on 64-bit)

&arr;          // Type: int (*)[10] — pointer to array of 10 ints
&p;            // Type: int ** — pointer to pointer to int
```

**Exceptions where arrays DON'T decay:**
1. `sizeof(arr)` — gives total array size
2. `&arr` — gives pointer-to-array, not pointer-to-pointer
3. `_Alignof(arr)`
4. String literal used to initialize a character array: `char s[] = "hello"` copies bytes; `char *s = "hello"` points to read-only memory

**The critical consequence:** When you pass an array to a function, it decays to a pointer. Size information is lost:
```c
// These DECLARATIONS are identical:
void func(int arr[]);  // arr is actually a pointer
void func(int *arr);   // Same thing
// ALWAYS pass the size as a separate parameter
```

### 2.4 Multidimensional Arrays — What's Really Happening

```c
int matrix[3][4];  // 12 contiguous ints in memory
```

This is NOT an array of pointers. It's a single block of memory. `matrix[i][j]` is at offset `(i * 4 + j) * sizeof(int)` from the start.

**Type breakdown:**
- `matrix` decays to `int (*)[4]` — pointer to array of 4 ints
- `matrix[i]` decays to `int *`
- `&matrix` is `int (*)[3][4]` — pointer to the entire 3×4 array

**Dynamic 2D array with different row lengths — array of pointers:**
```c
int *rows[3];
for (int i = 0; i < 3; i++)
    rows[i] = malloc(cols_i * sizeof(int));
// rows[i][j] — two memory indirections, not contiguous, flexible.
```

**Dynamic contiguous 2D array — single allocation:**
```c
int *data = malloc(rows * cols * sizeof(int));
// data[i * cols + j] — one indirection, cache-friendly.
```

### 2.5 Function Pointers — Reading the Syntax

Function pointers point to executable code. Read from the variable name outward.

```c
// Simple function pointer:
int (*op)(int, int);         // op = pointer to function(int, int) returning int
op = add;                    // function decays to function pointer
int result = op(3, 4);       // call through pointer

// Array of function pointers:
int (*ops[10])(int, int);    // ops = array[10] of pointer to function(int,int) returning int

// Function returning function pointer:
int (*get_op(void))(int, int);  // get_op(void) returns pointer to function(int,int) returning int

// typedef saves sanity:
typedef int (*binary_op_t)(int, int);
binary_op_t ops[10];
binary_op_t get_op(void);
```

**Two classic use cases:**

1. **Callbacks** — e.g., `qsort` takes a comparison function pointer
2. **Dispatch tables** — replacing long switch statements with array lookup, O(1) per call

### 2.6 Void Pointers — The Generic Pointer

`void *` can hold any data pointer. You cannot dereference it — must cast first.

```c
void *ptr = malloc(64);    // malloc returns void *
int *ip = ptr;             // void * implicitly converts to any pointer type in C
*ip = 42;
// *ptr = 42;  // ERROR: cannot dereference void *
```

### 2.7 Dangling, Wild, and NULL

```c
// Dangling: points to freed memory
int *p = malloc(sizeof(int));
free(p);
// p is now dangling. *p = 5; // UB

// Wild: uninitialized
int *q;
// *q = 5; // UB — q contains garbage

// The discipline:
// 1. Always initialize pointers (to NULL or valid memory)
// 2. Set to NULL after freeing: free(p); p = NULL;  // free(NULL) is a no-op
```

### 2.8 The Strict Aliasing Rule

The compiler may assume pointers to unrelated types point to different memory. Violating this is UB that silently generates wrong code.

```c
// WRONG: Violates strict aliasing
float f = 3.14f;
int *p = (int *)&f;
*p = 0x4048f5c3;  // UB

// CORRECT: Use memcpy (compiler optimizes to zero cost)
float f = 3.14f;
int bits;
memcpy(&bits, &f, sizeof(bits));  // Defined in C and C++
```

---

## 3. Data Structures — From Scratch, With Trade-offs

### 3.1 Dynamic Array (Vector)

**What it is:** A contiguous block of memory that grows automatically when full.

**Core fields:**
```c
typedef struct {
    int    *data;      // pointer to heap-allocated buffer
    size_t  length;    // number of elements currently stored
    size_t  capacity;  // size of allocated buffer (in elements)
} vector_t;
```

**Key operations and their complexity:**
- `push_back` — Append element. O(1) amortized. O(n) worst-case when resize triggers.
- `pop_back` — Remove last element. O(1).
- `get/set` at index — O(1) direct array access.

**The growth strategy:** When `length == capacity`, allocate a new buffer `growth_factor` times larger, copy old elements, free old buffer. Common growth factors:
- **2x:** Each push in a series has constant amortized cost. Proof: total copies = 1+2+4+...+n/2 < n, so n pushes cost ~3n total.
- **1.5x:** Used by Python and GCC's `std::vector`. Less memory waste because freed blocks can eventually be reused when combined. On average wastes less memory.

**Shrinking strategy:** Shrink when `length < capacity / 4` (not `/ 2`). This prevents thrashing when popping and pushing near a boundary.

**Why this matters for systems programming:** A dynamic array is the simplest non-trivial allocator. Implementing one forces you to understand `malloc`, `realloc`, `free`, pointer arithmetic, and amortized analysis — all core skills.

---

### 3.2 Linked List — Doubly Linked with Sentinel Node

**What it is:** Nodes scattered across the heap, each pointing to next and previous. A sentinel (dummy) node eliminates edge cases.

**Core structures:**
```c
typedef struct list_node {
    int                value;
    struct list_node  *next;
    struct list_node  *prev;
} list_node_t;

typedef struct {
    list_node_t sentinel;  // dummy head/tail — always exists
    size_t      length;
} list_t;
```

**How the sentinel works:** In an empty list, `sentinel.next` and `sentinel.prev` both point to `sentinel` itself. This means:
- Insert never has to check for empty list — there's always a `prev` node (the sentinel)
- Delete never has to check for head/tail — the sentinel's pointers are always valid
- Traversal loops become uniform: `for (node = sentinel.next; node != &sentinel; node = node->next)`

**Operations:**
- `insert_after(node, value)` — Unlink-free. Insert new node between two existing nodes. Complexity: O(1).
- `insert_head / insert_tail` — Just `insert_after(sentinel)` or `insert_after(sentinel.prev)`. O(1).
- `delete(node)` — Unlink and free. Must update `prev->next` and `next->prev`. O(1) if you have the node pointer.
- `find(value)` — Linear scan. O(n).

**Cache behavior (why arrays usually beat linked lists):** Each linked list node is separately `malloc`'d, scattered across the heap. Traversal causes a cache miss at every node. The CPU's L1 cache is ~1ns, main memory is ~100ns. For simple integer data, iterating an array can be 50x faster than a linked list, even though both are O(n).

**When to use a linked list:** When you need O(1) insert/delete in the middle of a collection (without shifting), or when elements are large and you want to avoid realloc-copy on growth.

---

### 3.3 Circular Buffer (Ring Buffer)

**What it is:** A fixed-size array treated as circular. Two indices (`head` for reads, `tail` for writes) that wrap around using modulo.

**Core fields:**
```c
typedef struct {
    int    *buffer;
    size_t  head;      // read from here
    size_t  tail;      // write here
    size_t  capacity;  // total slots
    size_t  count;     // current number of elements
} ring_t;
```

**How it works without shifting elements:**

1. **Enqueue (write):** Write `buffer[tail] = value`. Advance `tail = (tail + 1) % capacity`. Increment `count`.
2. **Dequeue (read):** Read `value = buffer[head]`. Advance `head = (head + 1) % capacity`. Decrement `count`.
3. **Full check:** `count == capacity`. Cannot enqueue.
4. **Empty check:** `count == 0`. Cannot dequeue.

**Why use `count` instead of distinguishing full/empty by leaving one slot unused:** The `count` field is simpler and clearer. The "leave one slot empty" trick saves 4-8 bytes but creates confusing edge cases where `head == tail` means either empty (normal) or full (one-slot-empty variant). For most systems code, clarity wins.

**Use cases:** Audio/video streaming buffers, keyboard input buffers, inter-thread producer-consumer queues (with mutex), any scenario where you want O(1) enqueue/dequeue without `malloc` or element shifting.

**What to notice:** This is a fixed-capacity structure. It never calls `malloc` after initialization. This is important for real-time and embedded systems where dynamic allocation is forbidden or unpredictable.

---

### 3.4 Hash Table — Open Addressing with Linear Probing

**What it is:** A key-value mapping backed by an array. Keys are hashed to an index. Collisions resolved by probing for the next empty slot.

**Core structures:**
```c
typedef struct {
    char    *key;
    int      value;
    bool     occupied;   // true if this slot is in use
    bool     deleted;    // true if this slot was deleted (tombstone)
} hash_entry_t;

typedef struct {
    hash_entry_t *entries;
    size_t        capacity;
    size_t        count;      // number of live entries
} hash_table_t;
```

**The flow of `lookup`:**

1. Hash the key to get initial index `i = hash(key) % capacity`.
2. Check `entries[i]`:
   - If `!occupied`: Key not found. Return NULL.
   - If `occupied` and `!deleted` and key matches: Found. Return value.
   - If `occupied` and (deleted or wrong key): This was a collision. Advance `i = (i + 1) % capacity` and repeat from step 2.
3. If we loop back to start, table is full and key not found.

**The flow of `insert`:**

1. Hash to get initial index `i = hash(key) % capacity`.
2. Probe forward. For each slot:
   - If `!occupied` or `deleted`: Found insertion point. Store key+value, mark occupied, clear deleted, increment count.
   - If `occupied` and not deleted and key matches: Update existing value.
   - Otherwise, advance `i`.
3. After insert: check load factor (`count / capacity`). If > 0.7, rehash (allocate new table ~2x capacity, reinsert all entries).

**The flow of `delete`:**

1. Hash to find the entry.
2. If found, mark as `deleted = true`. Do NOT clear `occupied`.
3. Decrement count.

**Why tombstones?** If you simply clear `occupied` on delete, the probe chain is broken. A key that probed past the deleted slot during insert will become unfindable. The tombstone tells the lookup "keep probing past this — there might be valid entries further on."

**What to notice about hash functions:** A bad hash function clusters keys into few buckets, destroying O(1) behavior. Two reliable string hashes:

```c
// djb2 — simple, fast, good distribution
uint64_t djb2(const char *s) {
    uint64_t h = 5381;
    while (*s) h = ((h << 5) + h) + (unsigned char)*s++;  // h*33 + c
    return h;
}

// FNV-1a — slightly better avalanche effect
uint64_t fnv1a(const char *s) {
    uint64_t h = 14695981039346656037ULL;
    while (*s) { h ^= (unsigned char)*s++; h *= 1099511628211ULL; }
    return h;
}
```

**Open addressing vs chaining trade-off:**
- **Open addressing** (what we built): Better cache locality — all entries in one array. Sensitive to load factor. Deletion requires tombstones.
- **Chaining** (buckets as linked lists): Survives higher load factors gracefully. Simpler deletion. But pointer-chasing kills cache performance.

---

### 3.5 Binary Search Tree (BST)

**What it is:** A hierarchical structure where each node has at most two children. Left subtree contains smaller values, right contains larger.

**Core structure:**
```c
typedef struct bst_node {
    int              value;
    struct bst_node *left;
    struct bst_node *right;
} bst_node_t;
```

**The flow of `insert`:**
1. If root is NULL, create new node, return it.
2. If `value < node->value`: recursively insert into left subtree.
3. If `value > node->value`: recursively insert into right subtree.
4. If equal: tree decides (duplicates ignored, or stored somewhere, depending on design).
5. Return the (possibly new) root.

**The flow of `search`:**
1. If node is NULL: not found.
2. If `value == node->value`: found.
3. If `value < node->value`: search left.
4. If `value > node->value`: search right.

**The flow of `delete` — the hard case:**
1. Find the node to delete.
2. If it has 0 children: just remove it (free, parent points to NULL).
3. If it has 1 child: replace it with that child.
4. If it has 2 children: Find the in-order successor (smallest value in right subtree). Copy successor's value into this node. Then recursively delete the successor (which has at most one child).

**The problem with plain BSTs:** If you insert sorted data (1, 2, 3, 4, 5...), the tree becomes a linked list. All operations degrade to O(n). This is why balanced trees exist.

---

### 3.6 AVL Tree (Self-Balancing BST)

**What it adds to BST:** After every insert or delete, check each node's "balance factor" (height of left subtree minus height of right subtree). If it exceeds +1 or -1, perform rotations to restore balance. The tree stays within 45% of optimal height.

**Core structure — adds height tracking:**
```c
typedef struct avl_node {
    int              value;
    struct avl_node *left;
    struct avl_node *right;
    int              height;    // height of this subtree
} avl_node_t;
```

**The rotation flow:**

1. **Right rotation (Left-Left case):** Left child becomes new root. Old root becomes right child of new root. Used when node is left-heavy and its left child is also left-heavy.
   ```
        z                 y
       / \               / \
      y   T4    →       x   z
     / \               / \ / \
    x  T3             T1 T2 T3 T4
   / \
  T1 T2
   ```

2. **Left rotation (Right-Right case):** Mirror of above.

3. **Left-Right case:** Left child is right-heavy. First left-rotate the left child (making it left-heavy), then right-rotate the node.
4. **Right-Left case:** Mirror of above.

**Balance factor check flow after insert:**
1. Recursively insert (just like BST).
2. On way back up, update each node's height `= 1 + max(height(left), height(right))`.
3. Calculate balance `= height(left) - height(right)`.
4. If balance > 1 and `value < node->left->value`: LL case → right rotate.
5. If balance > 1 and `value > node->left->value`: LR case → left rotate left child, then right rotate node.
6. If balance < -1 and `value > node->right->value`: RR case → left rotate.
7. If balance < -1 and `value < node->right->value`: RL case → right rotate right child, then left rotate node.

**What to notice:** Don't just implement AVL — understand WHY it works. The invariant "balance factor in {-1, 0, 1}" guarantees the tree height is at most ~1.44 * log₂(n). Every operation is O(log n). Visualize the rotations with pen and paper until they feel natural.

---

### 3.7 Binary Heap (Priority Queue)

**What it is:** A complete binary tree stored in an array where every parent is smaller (min-heap) or larger (max-heap) than its children.

**Core structure — no pointers needed, just an array:**
```c
typedef struct {
    int    *data;
    size_t  capacity;
    size_t  count;     // number of elements
} heap_t;
```

**Array-based tree navigation (no pointers):**
- Parent of node `i`: `(i - 1) / 2`
- Left child of node `i`: `2 * i + 1`
- Right child of node `i`: `2 * i + 2`

**The flow of `insert` (min-heap):**
1. Place new element at `data[count]`. Increment count.
2. **Sift up (bubble up):** Compare with parent. If smaller, swap. Repeat until heap property satisfied or at root.

**The flow of `extract_min`:**
1. Save `data[0]` (the minimum).
2. Move `data[count-1]` (last element) to position 0. Decrement count.
3. **Sift down (heapify):** Compare with children. If larger than smallest child, swap. Repeat until heap property satisfied or at leaf.
4. Return the saved minimum.

**What to notice:** A heap is NOT fully sorted. The only guarantee is: parent < children. This makes `extract_min` O(log n) instead of O(1) for a sorted array — but `insert` is also O(log n) instead of O(n) for a sorted array. You're trading insert performance for extract performance.

**Use cases:** Priority scheduling (OS task scheduler), Dijkstra's shortest path algorithm, heapsort (O(n log n) guaranteed, in-place), finding top-K elements from a stream.

**Heapsort flow:** Insert all elements into heap, then repeatedly extract min/max into an output array. O(n log n) guaranteed, in-place if you build the heap in the same array.

---

### 3.8 Trie (Prefix Tree)

**What it is:** A tree where each node represents a character. Paths from root to leaf spell words. Shared prefixes share nodes.

**Core structure:**
```c
typedef struct trie_node {
    struct trie_node *children[26];  // one per lowercase letter (or 256 for all ASCII)
    bool              is_end;        // true if this node ends a word
} trie_node_t;
```

**The flow of `insert(word)`:**
1. Start at root.
2. For each character `c` in `word`:
   - If `node->children[c - 'a']` is NULL, `malloc` a new node.
   - Move to that child.
3. At the final node, set `is_end = true`.

**The flow of `search(word)`:**
1. Start at root.
2. For each character `c`:
   - If `node->children[c - 'a']` is NULL: word not found.
   - Move to child.
3. After all characters, return `node->is_end`. (A prefix that isn't a word itself returns `false`.)

**The flow of `starts_with(prefix)`:**
1. Same as search, but after all characters, return `true` (don't check `is_end`). Any node at the end of the prefix means something starts with it.

**What to notice about space:** A naive 26-child-per-node trie wastes enormous memory — 26 pointers × 8 bytes = 208 bytes per node, most of which are NULL. For long, sparse strings, consider:
- **Compressed trie (radix tree):** Merge chains of single-child nodes. Common in IP routing and filesystem dentry caches.
- **Ternary search tree:** Each node has three children (less, equal, greater). Much more space-efficient but more complex operations.

**Use cases:** Autocomplete, spell checkers, IP routing tables (Longest Prefix Match), dictionary implementations where prefix search matters.

---

## 4. Algorithms — Implementations & Analysis

### 4.1 Sorting Algorithms — The Core Five

**Bubble Sort**
- **How it works:** Repeatedly step through the list, compare adjacent elements, swap if out of order. Keep going until no swaps needed.
- **Time:** O(n²) always. Space: O(1).
- **What to notice:** It's terrible. It's a teaching tool. If you ever write it in production code, you should have a very specific reason (data is always nearly sorted and tiny).

**Insertion Sort**
- **How it works:** Build sorted array one element at a time. For each new element, find where it belongs in the already-sorted prefix and shift everything over.
- **Time:** O(n²) worst, O(n) best (already sorted).
- **What to notice:** Excellent for small arrays (< ~10 elements). Used as the base case inside sophisticated sorts like Timsort and Introsort.

**Merge Sort**
- **How it works:** Divide array in half. Recursively sort each half. Merge the two sorted halves by walking two pointers.
- **Time:** O(n log n) guaranteed. Space: O(n) auxiliary.
- **What to notice:** Stable (equal elements preserve original order). This matters when you sort by one key, then another. The only O(n log n) stable sort. Predictable — doesn't degrade on any input.

**Quick Sort**
- **How it works:** Pick a pivot. Partition array into elements less than pivot and greater than pivot. Recursively sort partitions.
- **Time:** O(n log n) average, O(n²) worst (bad pivot choice). Space: O(log n) stack.
- **What to notice:** Fastest in practice due to cache locality. Partition does in-place swaps with excellent spatial locality. Not stable. `qsort` in libc is usually a quicksort variant.

**Heap Sort**
- **How it works:** Build heap from array. Repeatedly extract max to end of array.
- **Time:** O(n log n) guaranteed. Space: O(1).
- **What to notice:** The only O(n log n) in-place sort that is not quicksort. But worse constants than quicksort. Use when you need guaranteed O(n log n) and can't afford merge sort's extra memory.

---

### 4.2 Binary Search — Deceptively Subtle

**How it works:** Given a sorted array, compare target with middle element. If equal, found. If target is smaller, search left half. If larger, search right half.

**The bug in many implementations:**
```c
// WRONG: mid = (left + right) / 2;  // Overflow if left + right > INT_MAX

// CORRECT:
int mid = left + (right - left) / 2;  // No overflow possible
```

**What to notice:** Binary search is easy to describe but surprisingly hard to implement correctly. Off-by-one errors in the loop condition or boundary updates are the most common bugs. Practice it until you can write it from memory without mistakes.

**Variants:**
- Find first occurrence of value (modify condition to continue left after finding match)
- Find last occurrence
- Find insertion point (`lower_bound`)
- Search on rotated array (find the pivot, then binary search the correct half)

---

### 4.3 Graph Traversal — BFS and DFS

**Graph representation:** Adjacency list is almost always the right choice (array of lists, one per vertex). Adjacency matrix only when graph is dense.

**DFS (Depth-First Search)**
- **How it works:** Start at some vertex. Go as deep as possible before backtracking.
- **Implementation:** Recursive (uses call stack implicitly) or iterative with explicit stack.
- **What to notice:** DFS tree edges form a spanning tree. Back edges (to ancestors) indicate cycles in undirected graphs.
- **Use cases:** Cycle detection, topological sort (DFS, output vertex after exploring all children, reverse the order), connected components, maze solving.

**BFS (Breadth-First Search)**
- **How it works:** Start at some vertex. Explore all neighbors, then neighbors' neighbors, level by level.
- **Implementation:** Queue-based. Enqueue start, then while queue not empty: dequeue, process, enqueue all unvisited neighbors.
- **What to notice:** BFS finds shortest path in unweighted graphs (all edges cost 1). In a weighted graph, use Dijkstra.
- **Use cases:** Shortest path (unweighted), level-order tree traversal, web crawling, flood fill.

---

### 4.4 Recursion — The Call Stack Matters

**How recursion works at the machine level:**
1. When function F calls itself, a new stack frame is pushed with: return address, saved registers, local variables, arguments.
2. Each frame consumes stack space (~few hundred bytes up to KB for large locals).
3. When the base case is reached, frames unwind: return value passed up, frames popped.

**The stack overflow danger:**
- Default stack size on Linux: 8 MB (`ulimit -s`).
- 8 MB / ~200 bytes per frame ≈ 40,000 recursive calls.
- Deep recursion on large inputs will `SIGSEGV`.

**Tail recursion optimization:**
- If the recursive call is the absolute LAST thing the function does (no computation after it returns), the old stack frame is no longer needed.
- The compiler can replace recursion with a jump (loop), reusing the same stack frame.
- Requires `-O2` or higher.
- Write tail-recursive functions when recursion depth is a concern.

```c
// This is tail-recursive:
int factorial_tail(int n, int acc) {
    if (n <= 1) return acc;
    return factorial_tail(n - 1, n * acc);  // Nothing happens after the call returns
}
// Compiler can turn this into a loop -> no stack growth.
```

---

### 4.5 Sliding Window

**What it is:** A technique for problems on contiguous subarrays/strings. Maintain a window `[left, right)` and slide it across the array, updating the window's state in O(1) per step rather than recomputing from scratch.

**The pattern:**
```
left = 0
for right in 0..n-1:
    add arr[right] to window
    while window is invalid:
        remove arr[left] from window
        left++
    // window [left, right] is valid — record result if needed
```

**What to notice:** This turns O(n²) brute force ("check all possible subarrays") into O(n) by not re-scanning from scratch. The key insight: when the right pointer advances, you don't need to reconsider positions before left. The sliding window exploits monotonicity in the window's validity condition.

---

### 4.6 Bit Manipulation — The Systems Programmer's Toolkit

Bit-level operations are not tricks — they're essential for flags, masks, protocol headers, hardware registers, and performance-critical code.

**The operators:**
- `&` (AND): mask bits. `x & 0xFF` clears all but low byte.
- `|` (OR): set bits. `x | (1 << 3)` sets bit 3.
- `^` (XOR): toggle bits. `x ^ (1 << 3)` flips bit 3. `x ^ x == 0` — XOR is its own inverse.
- `~` (NOT): flip all bits. `x & ~(1 << 3)` clears bit 3.
- `<<` (left shift): multiply by 2^n. `x << 3 == x * 8`.
- `>>` (right shift): divide by 2^n (implementation-defined for signed, always logical for unsigned).

**Essential patterns:**
```c
// Check if bit n is set
(x >> n) & 1

// Set bit n
x |= (1 << n)

// Clear bit n
x &= ~(1 << n)

// Toggle bit n
x ^= (1 << n)

// Check if x is power of 2
(x != 0) && ((x & (x - 1)) == 0)
// Why: x & (x-1) clears the lowest set bit. For 1000 (8), x-1 = 0111, x & (x-1) = 0000.

// Isolate lowest set bit
x & (-x)   // Two's complement: -x = ~x + 1

// Count set bits (Brian Kernighan's algorithm)
int count = 0;
while (x) { x &= (x - 1); count++; }
```

---

## 5. Linux Internals — Kernel Perspective

### 5.1 Process Lifecycle — The Complete Flow

**What a process IS:** An instance of a program in execution. The kernel maintains a `task_struct` for each process containing: PID, parent PID, virtual memory map, file descriptor table, signal handlers, scheduling parameters, resource limits, and a hundred other fields.

**The flow of `fork()`:**
1. User calls `fork()` → syscall.
2. Kernel creates a new `task_struct`.
3. Kernel copies the parent's virtual memory page tables (not the pages themselves — Copy-on-Write).
4. Kernel copies the parent's file descriptor table (entries point to the same kernel file objects — reference counts incremented).
5. Kernel gives the child a new PID.
6. Kernel sets the child's state to `TASK_RUNNING` (runnable). Both parent and child are now ready.
7. Returns child PID to parent, 0 to child. They resume from the same instruction.

**Copy-on-Write (COW) in detail:**
- After `fork`, all memory pages in both processes are marked read-only and shared.
- When either process writes, the CPU's MMU triggers a page fault.
- The fault handler allocates a new physical page, copies the data, updates the page table, marks it writable.
- Pages that are never written are never copied. This is why `fork` is fast even for huge processes — `exec` replaces the entire address space without ever copying anything.

**The flow of `execve()`:**
1. Kernel loads the new program's binary (ELF parsing, segment loading).
2. Kernel wipes the current address space (frees all memory, including COW pages).
3. Kernel maps the new program's text, data, BSS, and stack.
4. Kernel sets up arguments and environment on the new stack.
5. Kernel sets instruction pointer to the new program's entry point (`_start`, not `main`).
6. Returns to userspace. The old program is gone — there's no going back.

**This is why `fork + exec` works:**
```
Parent: fork() → gets child PID
Child:  fork() → gets 0
Child:  execve("/bin/ls", ...) → transforms into ls process
Parent: waitpid(child_pid, ...) → waits for child to finish
```
The shell does this for every command you type.

**Zombie processes:**
- A child exits but parent hasn't `wait`ed yet.
- Child's memory is freed, but `task_struct` entry remains (holds exit code).
- Visible as `<defunct>` in `ps`.
- Too many zombies = a bug. `SIGCHLD` exists to tell parent to `wait`.

**Orphan processes:**
- Parent exits before child.
- Child is reparented to `init` (PID 1).
- `init` automatically `wait`s, so orphans don't become zombies.

---

### 5.2 Threads vs Processes — The Kernel Truth

**The Linux implementation:** Threads are processes that share their address space. The `clone()` syscall is the primitive; `fork()` is `clone(SIGCHLD)`. `pthread_create` uses `clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD)`.

**What's shared between threads:**
- Virtual address space (stack is the exception — each thread has its own region)
- File descriptor table
- Signal dispositions
- Umask, current directory, resource limits

**What's private per thread:**
- Stack (with guard page between them)
- Register values (program counter, stack pointer, etc.)
- Thread ID (`tid` in kernel, `pthread_t` in userspace)
- Signal mask
- `errno` (implemented as thread-local)

**What to notice:** A bug in one thread can corrupt shared data used by all threads. There's no memory protection between threads within a process. This is both the power and the danger.

**`futex` — the secret to fast mutexes:**
- A futex is a 32-bit integer in shared memory.
- Locking the mutex (uncontended): just an atomic compare-and-swap in userspace. NO SYSCALL.
- Locking the mutex (contended): does a `futex(FUTEX_WAIT)` syscall to put the thread to sleep.
- Unlocking the mutex (no waiters): just an atomic store. NO SYSCALL.
- Unlocking the mutex (waiters): does a `futex(FUTEX_WAKE)` syscall.
- The result: uncontended mutexes cost ~25ns (a few CPU cycles). Contended ones cost ~microseconds.

---

### 5.3 Virtual Memory — The Full Journey

**Why virtual memory exists:**
1. **Isolation:** Process A cannot see Process B's memory.
2. **Illusion of infinite memory:** Processes can use more virtual memory than physical RAM (excess swapped to disk).
3. **Simplified allocation:** Programs see a flat address space. The kernel handles fragmentation.

**The page table walk (x86-64, 4-level paging):**
```
Virtual address:
| 16 bits unused | 9 bits (PGD) | 9 bits (PUD) | 9 bits (PMD) | 9 bits (PTE) | 12 bits (offset) |

PGD → PUD → PMD → PTE → Physical Page + Offset → Data
```

Each level is a table. CPU walks through them. Four memory accesses per virtual address — plus the final data access. The TLB caches translations so you don't walk the table every time.

**TLB (Translation Lookaside Buffer):**
- A small, fast cache for virtual→physical translations.
- L1 TLB: ~64 entries for data, ~64 for instructions (CPU-dependent).
- L2 TLB: ~1500 entries.
- TLB miss + page table walk: ~10-100 cycles.
- TLB hit: 0 additional cycles (transparent).
- **TLB thrashing:** Working set uses more pages than TLB entries → constant misses → performance collapses.

**Page fault types:**
1. **Minor fault:** Page is in memory but not mapped (e.g., COW, lazy allocation). Just update page table. Fast.
2. **Major fault:** Page must be read from disk (swapped out, or mmap'd file not in cache). SLOW — disk I/O.
3. **Invalid fault:** Address is not valid (NULL, after `free`). Kernel sends `SIGSEGV`.

**`mmap` — the Swiss Army knife:**
- **File mapping:** `mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0)` — file appears as memory. Read/writes become disk I/O.
- **Anonymous mapping:** `mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)` — allocate memory. `malloc` uses this for large allocations.
- **Shared memory:** `MAP_SHARED` — changes visible to other processes mapping the same file.

---

### 5.4 Filesystems — Inodes, Directories, and File Descriptors

**Inode anatomy:** Every file/directory/symlink has an inode containing:
- Type (regular file, directory, symlink, FIFO, socket, block/char device)
- Permissions (rwx for owner/group/other)
- Owner (UID) and group (GID)
- Timestamps: access (atime), modification (mtime), status change (ctime)
- Size in bytes
- Number of hard links (reference count)
- Pointers to data blocks (direct, indirect, double-indirect, triple-indirect for large files)

**What a directory actually is:** A special file containing a list of `(name, inode_number)` pairs. That's it. The directory doesn't store file data or metadata — just name-to-inode mappings.

**Hard link vs symbolic link:**
- **Hard link:** Another directory entry pointing to the SAME inode. The inode's link count increments. Delete the original? File data lives on because link count > 0. Cannot span filesystems (inode numbers are filesystem-local). Cannot link to directories (prevents cycles).
- **Symbolic link (symlink):** A small file containing a path string. The kernel follows it transparently (mostly). Broken if target is deleted. Can span filesystems. Can point to directories.

**File descriptor table:**
- Per-process table. Indexed by integer (the fd).
- Each entry points to a kernel `struct file` object.
- The `struct file` has: current offset (for read/write), access mode (read/write), and a pointer to the inode.
- `fork` copies the fd table (entries point to same `struct file` — changing offset in one process affects the other).
- `dup2(oldfd, newfd)` makes `newfd` point to the same `struct file` as `oldfd`. This is how shell redirection works.

**Permissions and `umask`:**
- `open("file", O_CREAT, 0666)` asks for `rw-rw-rw-`.
- `umask` is a process-level mask. Default is `022`.
- Actual permissions: `0666 & ~umask` = `0666 & ~022` = `0644` (`rw-r--r--`).
- The umask strips bits you DON'T want by default. Set it to `0` if you want exact control.

---

### 5.5 Scheduling — What the Kernel Decides

**The Completely Fair Scheduler (CFS) — Linux's default:**

Goal: give each runnable task a fair share of CPU time, weighted by priority ("nice" value).

**The `vruntime` concept:** Each task has a virtual runtime — how long it has been on CPU, scaled by priority. Lower `vruntime` means the task "deserves" more CPU. CFS picks the task with the lowest `vruntime` to run next.

**Nice values:** Range from -20 (highest priority) to +19 (lowest priority). Each step corresponds to ~10% CPU share change. Nice 0 gets 1024/weight, nice 5 gets 335/weight. The weight math means a nice+5 task gets ~1/3 the CPU of a nice 0 task.

**Preemption:** CFS checks periodically: "has the current task used its fair share?" If yes, and another task has lower `vruntime`, the current task is preempted (interrupted). This happens at the granularity of the scheduler tick (~1-10ms depending on `CONFIG_HZ`).

**Context switch cost:**
1. Save current task's registers, PC, stack pointer to its `task_struct`.
2. Flush TLB entries (or use PCID/ASID to avoid flushing — newer CPUs).
3. Load new task's registers, PC, stack pointer.
4. Cost: ~1-5 microseconds. This seems small but adds up. Too many context switches = system is scheduling instead of doing work.

**Real-time scheduling:**
- `SCHED_FIFO`: Tasks run until they block or yield. No time slicing. Strict priority order.
- `SCHED_RR`: Like FIFO but with time slice per priority level.
- Real-time tasks ALWAYS preempt normal (`SCHED_OTHER`) tasks. A runaway RT task can lock up the system.
- Requires `CAP_SYS_NICE` or root to set.

---

## 6. System Calls — Complete API Reference

### 6.1 File I/O — The Core Functions

**`open`** — The gateway to files:
```c
int open(const char *pathname, int flags, mode_t mode);
// Returns: fd on success, -1 on error (check errno)
// flags: O_RDONLY, O_WRONLY, O_RDWR (exactly one required)
//        O_CREAT (create if not exists), O_TRUNC (truncate to 0), O_APPEND (every write at end)
//        O_NONBLOCK (don't block on open/read/write), O_SYNC (synchronous writes)
//        O_DIRECT (bypass kernel buffer cache — direct disk I/O, requires aligned buffers)
// mode: permission bits (only with O_CREAT), subject to umask
```

**What to know about `open` flags:**
- `O_APPEND`: The file offset is set to end before every write. Atomic on local filesystems — multiple processes appending to same file won't overwrite each other.
- `O_TRUNC`: Truncates the file to 0 size. If you open for reading only, this is not needed.
- `O_NONBLOCK`: Affects when `open` itself can block (FIFO open blocks until both ends connected; with `O_NONBLOCK`, returns immediately). Also affects subsequent I/O.
- `O_DIRECT`: Bypasses the kernel page cache. Read/write directly to/from disk. Requires buffers aligned to 512-byte boundary, and transfers in multiples of 512 bytes. Used by databases that do their own caching.

**The key difference between `O_SYNC`, `O_DSYNC`, and `O_RSYNC`:**
- `O_SYNC`: Every write waits until both data AND metadata are on disk. Slowest but safest.
- `O_DSYNC`: Write waits until data is on disk; metadata (like mtime) can be lazy. Faster. Use for database journal writes.
- `O_RSYNC`: Combines with `O_SYNC` or `O_DSYNC` to also apply sync semantics to reads (read waits until any pending writes to that region complete).

---

**`read`** — Read bytes from file descriptor:
```c
ssize_t read(int fd, void *buf, size_t count);
// Returns: number of bytes read (0 = EOF), -1 on error
// May return FEWER bytes than requested. ALWAYS handle partial reads.
```

**Why partial reads happen:**
- Reading from terminal: up to one line.
- Reading from socket/pipe: up to what's available.
- Signal interrupted the read (if not restarted via `SA_RESTART`).
- Near end of file: whatever's left.

**The correct read loop pattern:**
```c
ssize_t read_all(int fd, void *buf, size_t total) {
    size_t read_bytes = 0;
    while (read_bytes < total) {
        ssize_t n = read(fd, (char*)buf + read_bytes, total - read_bytes);
        if (n == 0) return read_bytes;  // EOF
        if (n < 0) {
            if (errno == EINTR) continue;  // Interrupted by signal — retry
            return -1;  // Real error
        }
        read_bytes += n;
    }
    return read_bytes;
}
```

**What to notice:** `EINTR` — The read was interrupted by a signal before any data arrived. This is NOT an error — you should retry. This is why loops are necessary even for "simple" reads.

---

**`write`** — Write bytes to file descriptor:
```c
ssize_t write(int fd, const void *buf, size_t count);
// Returns: number of bytes written, -1 on error
// May write FEWER bytes than requested.
```

**Why partial writes happen:**
- Writing to socket/pipe: buffer is full.
- Disk full.
- Signal interruption (`EINTR`).

**The correct write loop pattern:** Same as read_all — loop until all bytes written or real error. Just use `write` instead of `read`, and 0 return is not special (write returns 0 when count is 0).

**What to notice about file offsets:** Both `read` and `write` advance the file offset by the number of bytes actually transferred. This happens even for partial reads/writes. The next operation continues from where the last one left off. `pread` and `pwrite` avoid this — they operate at a specified offset without changing the file offset (useful in multithreaded code).

---

**`close`** — Close file descriptor:
```c
int close(int fd);
// Returns: 0 on success, -1 on error
// Always check the return value — close can fail!
```

**Why `close` can fail:**
- NFS: pending writes might fail on close (the close triggers the actual network flush).
- `EINTR`: Interrupted by signal.
- `EBADF`: Already closed (double-close bug).
- If `close` fails, the fd is freed regardless. The error means something went wrong with the underlying I/O, not that the fd is still open.

**Best practice for robust close:**
```c
// The Linux way: close, retry on EINTR only
while (close(fd) == -1 && errno == EINTR) continue;

// The POSIX way (portable): treat EINTR close as undefined — either don't retry, or
// use a loop. Most systems handle it fine.
```

---

**`lseek`** — Reposition file offset:
```c
off_t lseek(int fd, off_t offset, int whence);
// whence: SEEK_SET (from beginning), SEEK_CUR (from current), SEEK_END (from end)
// Returns: new offset, -1 on error
```

**The `lseek(fd, 0, SEEK_END)` trick for file size:** This positions at the end and returns the offset = file size. But careful: this doesn't work on all file types (pipes, sockets, some `/proc` files return `ESPIPE`). Prefer `fstat`.

**Seekable vs non-seekable:** Regular files, block devices: seekable. Pipes, FIFOs, sockets, terminals: NOT seekable. `lseek` returns `ESPIPE` on these.

---

**`stat` / `fstat` / `lstat`** — Get file metadata:
```c
int stat(const char *pathname, struct stat *statbuf);   // follows symlinks
int fstat(int fd, struct stat *statbuf);                 // from fd
int lstat(const char *pathname, struct stat *statbuf);   // doesn't follow symlinks (gives symlink info)
```

**`struct stat` fields to know:**
- `st_mode`: File type (`S_ISREG`, `S_ISDIR`, `S_ISLNK`...) and permissions
- `st_size`: File size in bytes
- `st_ino`: Inode number
- `st_nlink`: Number of hard links
- `st_uid`, `st_gid`: Owner and group
- `st_atime`, `st_mtime`, `st_ctime`: Access, modification, status change timestamps

---

**`fcntl`** — File control (the miscellaneous file descriptor Swiss Army knife):
```c
int fcntl(int fd, int cmd, ... /* optional arg */);
```

**Essential `fcntl` operations:**

1. **Get/set file descriptor flags:**
   ```c
   int flags = fcntl(fd, F_GETFL);
   fcntl(fd, F_SETFL, flags | O_NONBLOCK);  // Set non-blocking
   ```

2. **Duplicate file descriptor** (like `dup`, but more control):
   ```c
   int newfd = fcntl(fd, F_DUPFD, min_fd);  // New fd >= min_fd
   ```

3. **Advisory record locking:**
   ```c
   struct flock fl = {F_WRLCK, SEEK_SET, 0, 0, 0};  // Lock entire file for writing
   fcntl(fd, F_SETLKW, &fl);   // Blocking write lock
   // F_SETLK is non-blocking, F_GETLK checks existing locks
   ```

**What "advisory" means:** Advisory locks only work if all cooperating processes check them. A process that doesn't call `fcntl(F_GETLK)` can still read/write the file freely. Mandatory locking exists but is rarely used (mount option `-o mand`).

---

### 6.2 Process Management — The Core Functions

**`fork`** — Create a child process:
```c
pid_t fork(void);
// Returns: child PID in parent, 0 in child, -1 on error
```

**What fork duplicates:**
- Address space (via COW — not immediately physically copied)
- File descriptor table (entries point to same kernel file objects)
- Signal dispositions (but pending signals and alarm timer are cleared in child)
- Current directory, umask, resource limits, environment

**What fork does NOT duplicate:**
- PID (child gets a new one)
- Parent PID (child's is set to parent's PID)
- Pending signals (cleared in child)
- File locks (NOT inherited — this is a common gotcha)
- Timer (alarm is cleared)

**The typical `fork` pattern:**
```c
pid_t pid = fork();
if (pid < 0) {
    perror("fork");
    exit(1);
} else if (pid == 0) {
    // Child: do child stuff
    // If you're going to exec, do it quickly to minimize COW overhead
    execvp("program", args);
    perror("execvp");  // Only reached if exec fails
    _exit(127);        // Use _exit to avoid flushing parent's stdio buffers
} else {
    // Parent
}
```

**Why `_exit` instead of `exit` in child after failed exec:** `exit` flushes stdio buffers (which were inherited from parent and contain parent's data). `_exit` doesn't. Also, `atexit` handlers registered by parent shouldn't run in child.

---

**`waitpid`** — Wait for child process state change:
```c
pid_t waitpid(pid_t pid, int *wstatus, int options);
// pid > 0: wait for specific child
// pid = -1: wait for any child
// pid = 0: wait for any child in same process group
// pid < -1: wait for any child in process group abs(pid)
// options: WNOHANG (non-blocking), WUNTRACED (also wait for stopped children), WCONTINUED
```

**Extracting status with macros:**
```c
if (WIFEXITED(status))       // Child exited normally
    code = WEXITSTATUS(status);  // Get exit code (0-255)
if (WIFSIGNALED(status))     // Child killed by signal
    sig = WTERMSIG(status);      // Get signal number
if (WIFSTOPPED(status))      // Child stopped by signal
    sig = WSTOPSIG(status);
```

**The double-fork trick to avoid zombies (daemonizing):**
```c
if (fork() > 0) { wait(NULL); exit(0); }  // Parent waits for intermediate child
if (fork() > 0) exit(0);  // Intermediate child exits. Grandchild is orphan.
// Grandchild is now child of init. init will reap it.
// The original parent doesn't need to wait for the daemon process.
```

---

**`pipe`** — Create unidirectional byte stream:
```c
int pipe(int pipefd[2]);
// pipefd[0] = read end, pipefd[1] = write end
```

**The pipe capacity:** On Linux, the pipe buffer is 64 KB (65536 bytes) since kernel 2.6.11. Writes up to this size are atomic (won't be interleaved with writes from other processes for `PIPE_BUF` or less, typically 4096 bytes).

**The `pipe + fork` communication pattern:**
1. Parent creates pipe.
2. Parent forks. Both processes have both pipe ends.
3. Each process closes the end it doesn't use. Example: Parent writes, child reads → parent closes `pipefd[0]`, child closes `pipefd[1]`.
4. Now it's a clean one-way channel.

**EOF on pipe:** EOF is signaled to the reader when ALL writers close their write end. This is why closing unused ends is critical — if the reader keeps its write end open, it will never see EOF.

---

**`dup2`** — Duplicate file descriptor to a specific number:
```c
int dup2(int oldfd, int newfd);
// Makes newfd point to the same file description as oldfd.
// Closes newfd first if it was open (atomically).
```

**How the shell implements redirection: `cmd > file.txt`**
1. Open `file.txt` with `O_WRONLY | O_CREAT | O_TRUNC`. Gets fd 3 (say).
2. `dup2(3, STDOUT_FILENO)` — make fd 1 point to the file.
3. `close(3)` — don't need two fds pointing to the same file.
4. Fork and exec the command. It writes to stdout (fd 1) → goes to file.

**How the shell implements pipes: `cmd1 | cmd2`**
1. Create pipe: `pipe(pipefd)`. Gets fds 3 (read) and 4 (write).
2. Fork child 1: `dup2(pipefd[1], STDOUT_FILENO)`. Close both pipefd. Exec `cmd1`.
3. Fork child 2: `dup2(pipefd[0], STDIN_FILENO)`. Close both pipefd. Exec `cmd2`.
4. Parent closes both pipefd. Waits for both children.

---

### 6.3 Socket APIs — The Core Functions

**`socket`** — Create a communication endpoint:
```c
int socket(int domain, int type, int protocol);
// domain: AF_INET (IPv4), AF_INET6 (IPv6), AF_UNIX (local)
// type: SOCK_STREAM (TCP), SOCK_DGRAM (UDP), SOCK_RAW (raw packets)
// protocol: 0 (auto-select based on type), or IPPROTO_TCP, IPPROTO_UDP
```

**The `AF_INET` vs `PF_INET` thing:** They're the same. Historically `PF_*` was for "protocol family" and `AF_*` for "address family." Use `AF_INET`.

**`bind`** — Assign address and port:
```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
// Server side: bind to a well-known port so clients can find you.
// Client side: optional (OS assigns ephemeral port automatically).
```

**Setting up a `sockaddr_in` for IPv4:**
```c
struct sockaddr_in addr = {
    .sin_family = AF_INET,
    .sin_port   = htons(8080),         // Host to Network Short — convert byte order
    .sin_addr   = { htonl(INADDR_ANY) } // 0.0.0.0 = listen on all interfaces
};
```

**The byte order problem:** Different CPUs store integers differently in memory. Little-endian (x86): least significant byte at lowest address. Big-endian: most significant byte at lowest address. Network byte order is BIG-ENDIAN. You MUST convert:
- `htons(uint16_t)`: Host to Network Short (16-bit). For port numbers.
- `htonl(uint32_t)`: Host to Network Long (32-bit). For IP addresses.
- `ntohs`, `ntohl`: Network to Host. Use these when reading from network structs.

**`listen`** — Mark socket as passive (for accepting connections):
```c
int listen(int sockfd, int backlog);
// backlog: queue size for pending connections (SYN received, not yet accepted)
// Linux caps this at /proc/sys/net/core/somaxconn (default 128)
```

**What `backlog` means:** It's the maximum number of connections that have completed the TCP handshake but not yet been `accept`ed. If this queue is full, new connection attempts are silently dropped (TCP client retries with SYN).

**`accept`** — Accept incoming connection:
```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
// Blocks until a connection is available. Returns NEW fd for that connection.
// Original sockfd continues listening.
// addr gets the client's address (pass NULL if you don't care).
```

**`connect`** — Initiate connection (client side):
```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
// For TCP: does the three-way handshake. Blocks until connected or error.
// For UDP: just records the default destination address (doesn't send anything).
```

**`send` and `recv`** — Socket-specific I/O:
```c
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
// Like read/write but with socket-specific flags.
// flags: MSG_DONTWAIT (non-blocking for this call), MSG_PEEK (read without consuming),
//        MSG_WAITALL (block until len bytes received or error/EOF)
```

**`sendto` and `recvfrom`** — For UDP (datagrams):
```c
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
               const struct sockaddr *dest_addr, socklen_t addrlen);
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                 struct sockaddr *src_addr, socklen_t *addrlen);
// recvfrom fills src_addr with the sender's address.
```

**`getaddrinfo`** — The modern way to resolve hostnames:
```c
int getaddrinfo(const char *node, const char *service,
                const struct addrinfo *hints, struct addrinfo **res);
// Replaces gethostbyname. Handles IPv4/IPv6. Thread-safe.
// node: hostname or IP string (e.g., "google.com" or "1.2.3.4")
// service: port string (e.g., "80") or NULL
```

**The `getaddrinfo` usage pattern:**
```c
struct addrinfo hints = {0}, *result, *rp;
hints.ai_family = AF_UNSPEC;     // IPv4 or IPv6
hints.ai_socktype = SOCK_STREAM; // TCP
hints.ai_flags = AI_PASSIVE;     // For server (use my IP)

if (getaddrinfo(NULL, "8080", &hints, &result) != 0) { /* error */ }

// Iterate through results; try to bind to the first one that works
for (rp = result; rp != NULL; rp = rp->ai_next) {
    sockfd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
    if (sockfd == -1) continue;
    if (bind(sockfd, rp->ai_addr, rp->ai_addrlen) == 0) break;  // Success
    close(sockfd);
}
freeaddrinfo(result);  // Don't forget to free!
```

**What to notice:** `getaddrinfo` returns a linked list of address structs. Iterate through them. Try to create/bind a socket with each one. Use the first one that works. This handles dual-stack (IPv4 and IPv6) transparently.

---

### 6.4 `epoll` — High-Performance Event Notification

**`epoll` vs `select`/`poll`:**
- `select`: O(n) per call (scans all fds). 1024 fd limit. Copies fd set between kernel/userspace every call. Legacy.
- `poll`: O(n) per call. No fd limit. Still copies fd array every call. Better API, same performance.
- `epoll`: O(1) per call (only ready fds returned). State maintained in kernel. Million-fd scalability. Linux-specific.

**The three `epoll` functions:**

```c
// Create epoll instance
int epoll_create1(int flags);  // flags: 0 or EPOLL_CLOEXEC (close-on-exec)
// Returns: epoll fd, -1 on error

// Add/modify/remove file descriptors to watch
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// op: EPOLL_CTL_ADD, EPOLL_CTL_MOD, EPOLL_CTL_DEL
// event: specifies what events to watch for

// Wait for events
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
// Returns: number of ready fds, 0 on timeout, -1 on error
// timeout: milliseconds, -1 = forever
```

**The `struct epoll_event`:**
```c
struct epoll_event {
    uint32_t events;    // EPOLLIN (readable), EPOLLOUT (writable), EPOLLERR, EPOLLHUP
                        // EPOLLET (edge-triggered)
    epoll_data_t data;  // User data: fd, pointer, or arbitrary integer
};

typedef union epoll_data {
    void    *ptr;
    int      fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
```

**Level-triggered vs Edge-triggered:**

- **Level-triggered (default):** As long as the fd is ready (e.g., has data), `epoll_wait` keeps returning it. Forgiving — if you don't read all data, you'll be notified again.
- **Edge-triggered (`EPOLLET`):** Notified ONLY when fd transitions from not-ready to ready. You MUST read/write until `EAGAIN`. No second chance. Higher performance (fewer kernel calls) but harder to get right.

**The correct edge-triggered read pattern:**
```c
// In the event loop, for EPOLLIN with EPOLLET:
while (1) {
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n > 0) { /* process buf */ }
    else if (n == 0) { /* EOF, close fd */ break; }
    else if (n < 0 && errno == EAGAIN) { break; }  // No more data
    else { /* real error */ break; }
}
// Without EPOLLET: just read once. If more data arrives, you'll get another event.
```

**The epoll event loop skeleton:**
```c
int epfd = epoll_create1(0);
struct epoll_event ev = {.events = EPOLLIN, .data.fd = listen_fd};
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

struct epoll_event events[MAX_EVENTS];
while (running) {
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
    for (int i = 0; i < nfds; i++) {
        if (events[i].data.fd == listen_fd) {
            // Accept new connection
            int client_fd = accept(listen_fd, NULL, NULL);
            set_nonblocking(client_fd);
            struct epoll_event ev = {.events = EPOLLIN | EPOLLET, .data.fd = client_fd};
            epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);
        } else {
            // Handle client data
            handle_client(events[i].data.fd);
        }
    }
}
```

**What to notice:** `epoll_wait` returns only the file descriptors that have events. You don't scan all fds. This is why it scales to hundreds of thousands of connections — the overhead is proportional to the number of ACTIVE connections, not the total.

---

### 6.5 Signals — Asynchronous Notifications

**What signals are:** A limited form of IPC. One process sends a signal to another (or the kernel sends it). The receiving process can: catch it (run a handler), ignore it (`SIG_IGN`), or accept the default action (terminate, stop, ignore, or core dump).

**`sigaction`** — The proper way to set signal handlers:
```c
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
// Prefer over signal() — sigaction is more portable and gives more control.
```

```c
struct sigaction {
    void     (*sa_handler)(int);     // Handler function (SIG_DFL, SIG_IGN, or your function)
    void     (*sa_sigaction)(int, siginfo_t *, void *);  // Extended handler (if SA_SIGINFO set)
    sigset_t sa_mask;                // Signals to block during handler execution
    int      sa_flags;               // SA_RESTART, SA_SIGINFO, SA_NOCLDSTOP, etc.
};
```

**Critical `sa_flags` to know:**
- `SA_RESTART`: Automatically restart interrupted system calls (like `read`). Without it, `read` returns `-1` with `errno == EINTR`. With it, the kernel retries.
- `SA_SIGINFO`: Use the 3-argument handler `sa_sigaction` instead of `sa_handler`. Gives more info about the signal (e.g., for `SIGCHLD`, tells you the child PID and exit status).
- `SA_NOCLDSTOP`: Don't send `SIGCHLD` when child is stopped (ptrace/SIGSTOP). Only on termination.
- `SA_NODEFER` / `SA_NOMASK`: Don't automatically block the signal during its handler. Allows recursive signal delivery (usually not what you want).

**Signal handler DOs and DON'Ts:**

**DON'T:**
- Call `printf`, `malloc`, `free`, or any non-async-signal-safe function. The program might be in the middle of `malloc` when the signal arrives — calling `malloc` from the handler will deadlock or corrupt the allocator.
- Modify global state without `volatile sig_atomic_t`.
- Do significant work.

**DO:**
- Set a flag: `volatile sig_atomic_t flag = 1;`
- Write to a self-pipe (write to a pipe that your event loop watches).
- Call `_exit()` or `abort()`.

**The self-pipe trick for clean signal handling in event loops:**
```c
static int signal_pipe[2];

void signal_handler(int sig) {
    // Write the signal number to the pipe
    write(signal_pipe[1], &sig, sizeof(sig));
}

// In main:
pipe(signal_pipe);
// Add signal_pipe[0] to your epoll loop with EPOLLIN
// When epoll says it's readable, read signal numbers and process them
// This safely moves signal handling from async context to the event loop.
```

**Signal blocking with `sigprocmask`:**
```c
sigset_t set;
sigemptyset(&set);
sigaddset(&set, SIGINT);
sigprocmask(SIG_BLOCK, &set, NULL);   // Block SIGINT
// ... critical section where you can't be interrupted ...
sigprocmask(SIG_UNBLOCK, &set, NULL); // Unblock
// Pending signals are delivered when unblocked.
```

---

## 7. Networking — From Socket to Server

### 7.1 TCP Fundamentals — The Wire Protocol

**Three-way handshake in detail:**

1. Client → Server: `SYN` (Sequence = client_isn)
   - Flags: SYN=1, ACK=0
   - Client enters `SYN_SENT` state

2. Server → Client: `SYN-ACK` (Sequence = server_isn, Ack = client_isn + 1)
   - Flags: SYN=1, ACK=1
   - Server enters `SYN_RECV` state

3. Client → Server: `ACK` (Sequence = client_isn + 1, Ack = server_isn + 1)
   - Flags: ACK=1
   - Both enter `ESTABLISHED` state

**Why random initial sequence numbers (ISN)?** Security. Predictable ISNs allow connection hijacking. The ISN prevents old segments from previous connections (same port reused) from being misinterpreted.

**Four-way connection teardown:**
1. Active closer: `FIN` (FIN_WAIT1)
2. Passive closer: `ACK` (CLOSE_WAIT, active closer → FIN_WAIT2)
3. Passive closer: `FIN` (LAST_ACK, active closer → TIME_WAIT)
4. Active closer: `ACK` (passive closer → CLOSED)
5. Active closer waits 2×MSL (Maximum Segment Lifetime, 60 seconds) in TIME_WAIT before CLOSED.

**Why TIME_WAIT exists:** To catch any delayed segments from this connection that might confuse a new connection on the same port tuple. 2×MSL ensures all old segments have expired.

**Sliding window (flow control):** Each segment carries a window size — "I can accept this many more bytes." Sender cannot exceed this. Prevents fast sender from overwhelming slow receiver. Zero window → sender sends keep-alive probes.

**Congestion control (TCP algorithms):** Not the same as flow control. Deals with network congestion, not receiver capacity.
- **Slow Start:** Congestion window starts small (~10 MSS). Doubles each RTT until a threshold.
- **Congestion Avoidance:** Above threshold, grows linearly (one MSS per RTT).
- **Fast Retransmit:** If 3 duplicate ACKs received, retransmit lost segment without waiting for timeout.
- **Fast Recovery:** After fast retransmit, don't reset to slow start; go to congestion avoidance.

---

### 7.2 Blocking vs Non-blocking I/O

**Blocking I/O:**
- `read(fd, buf, n)` — thread sleeps until data available OR error.
- Simple, sequential code.
- One thread can handle only one connection at a time.
- To handle N connections, you need N threads. N threads = N stacks (8 MB each) + context switch overhead.

**Non-blocking I/O:**
- `fcntl(fd, F_SETFL, O_NONBLOCK)`, then `read` returns immediately.
- If no data: returns -1 with `errno = EAGAIN/EWOULDBLOCK`.
- You MUST check `errno == EAGAIN`. This is NOT an error — it means "try again later."

**I/O Multiplexing:**
- One thread monitors many file descriptors.
- When a fd becomes ready (has data, can write, etc.), the multiplexing call returns.
- Then you read/write that fd (always in non-blocking mode to be safe).

**The three multiplexing APIs:**

| API | Theory | Max FDs | Per-Call Cost | Portability |
|---|---|---|---|---|
| `select` | Scans all fds 0..maxfd | 1024 (fixed) | O(n) | Very portable |
| `poll` | Scans pollfd array | Unlimited | O(n) | POSIX |
| `epoll` | Stateful, returns only ready fds | Unlimited | O(1) in active fds | Linux only |
| `kqueue` | Like epoll for BSD/Mac | Unlimited | O(1) | BSD, macOS |

**Why `select` is terrible:** It uses `fd_set` (bitmask of 1024 bits). Every call, kernel must scan from 0 to highest fd. Kernel copies fd_set in and out. You must reinitialize the sets before each call (they're modified in-place). No per-fd user data.

**Why `epoll` wins:**
1. Register interest once (`epoll_ctl`). No repeated registration.
2. Kernel maintains the interest list. No copying to/from userspace on each poll.
3. `epoll_wait` returns only ready fds. You iterate O(active_fds), not O(total_fds).
4. Edge-triggered mode avoids repeated notifications if you didn't consume all data.

---

### 7.3 Networking Projects — Progressive Complexity

**Project 1: TCP Echo Server (blocking, single-threaded)**
- Accept one connection, echo until client disconnects, then accept next.
- Teaches: socket, bind, listen, accept, read/write loop.
- What to notice: Only one client at a time.

**Project 2: TCP Echo Server (threaded — one thread per connection)**
- `accept` loop in main thread. For each client: create a pthread that runs echo loop.
- Teaches: `pthread_create`, joining/detaching threads, basic thread safety.
- What to notice: N connections → N threads. At 1000 connections, 8 GB stack. Context switch pain.

**Project 3: TCP Echo Server (thread pool)**
- Fixed pool of worker threads (e.g., 8). Main thread accepts and dispatches to pool.
- Task queue with mutex + condition variable for dispatching.
- Teaches: proper thread pool, mutex, cond var, fixed resource usage.
- What to notice: Handles thousands of connections with constant resources.

**Project 4: epoll Event Loop Server (single-threaded)**
- Single thread uses `epoll`. Handles hundreds of connections with zero threads.
- Teaches: `epoll_create1`, `epoll_ctl`, `epoll_wait`, non-blocking I/O.
- What to notice: Must handle `EAGAIN`. Must buffer partial writes. Much more complex code.

**Project 5: HTTP/1.0 File Server**
- Parses `GET /path HTTP/1.0\r\n...`
- Serves static files. Status codes: 200, 404, 405.
- Teaches: String parsing in C (no regex, no sscanf overuse), proper header handling.
- Requirements: Handle partial reads. Handle binay files (Content-Length). Signal-safe shutdown.

**Project 6: Chat Server (epoll + broadcast)**
- Track connected clients. When one sends a message, broadcast to all others.
- Shared state under single-threaded epoll (no locking needed) or multi-threaded (requires locking).
- Teaches: Managing shared state, broadcast patterns, protocol design (framing messages over TCP).

---

## 8. Concurrency — Deep Dive

### 8.1 The Shared-State Problem

**The fundamental issue:** When two threads access the same memory without synchronization, and at least one writes, the result is non-deterministic. This is the **data race**.

**Why compound operations fail:**

Consider `counter++`. In C, this is:
1. Load `counter` from memory into register.
2. Increment register.
3. Store register back to memory.

With two threads each doing `counter++` 1000 times:
```
Thread A: load (counter = 0)
Thread B: load (counter = 0)
Thread A: increment → 1
Thread B: increment → 1
Thread A: store (counter = 1)
Thread B: store (counter = 1)
// Two increments, counter went from 0 to 1. Lost update.
```

**The solution — mutual exclusion:**
A mutex ensures only one thread executes the critical section at a time:
```c
pthread_mutex_lock(&mutex);
counter++;  // Now this is atomic with respect to other threads holding the mutex
pthread_mutex_unlock(&mutex);
```

**The golden rules of mutexes:**
1. Always lock in the same order across threads (prevents deadlock).
2. Always unlock on all paths (including error paths).
3. Keep critical sections as short as possible.
4. Never sleep (block) while holding a mutex.

---

### 8.2 pthreads — The Full API

**Mutex lifecycle:**
```c
// Static initialization (compile-time):
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

// Dynamic initialization:
pthread_mutex_t mutex;
pthread_mutex_init(&mutex, NULL);  // NULL = default attributes

// Locking:
pthread_mutex_lock(&mutex);      // Blocks until acquired
pthread_mutex_trylock(&mutex);   // Returns EBUSY if already held, doesn't block

// Unlocking:
pthread_mutex_unlock(&mutex);

// Destruction:
pthread_mutex_destroy(&mutex);
```

**Condition variable — the signaling pattern:**
A condition variable lets a thread sleep until some condition is true.

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  cond  = PTHREAD_COND_INITIALIZER;

// Waiting thread:
pthread_mutex_lock(&mutex);
while (!condition_met) {  // ALWAYS loop-check the condition
    pthread_cond_wait(&cond, &mutex);  // Atomically: unlock mutex and sleep.
    // When woken: mutex is re-locked automatically.
}
// Condition is now true (we hold mutex)
do_work();
pthread_mutex_unlock(&mutex);

// Signaling thread:
pthread_mutex_lock(&mutex);
condition_met = true;
pthread_cond_signal(&cond);     // Wake one waiter
// or pthread_cond_broadcast(&cond); // Wake all waiters
pthread_mutex_unlock(&mutex);
```

**Why `while` and not `if`: Spurious wakeups.** The thread can be woken even if no signal was sent (this is allowed by POSIX). Also, between the signal and the thread actually running, another thread might have changed the condition back. ALWAYS re-check.

**The sequence inside `pthread_cond_wait`:**
1. Atomically: release the mutex and put thread to sleep on the condition.
2. When woken (signal/broadcast/spurious): re-acquire the mutex.
3. Return.

The atomicity of step 1 is critical — no other thread can sneak in between the unlock and the sleep.

---

### 8.3 Read-Write Locks

When you have many readers and few writers, a mutex serializes all reads unnecessarily. `pthread_rwlock_t` allows:
- Multiple simultaneous readers
- Exactly one writer (which blocks all readers and other writers)

```c
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;

// Reader:
pthread_rwlock_rdlock(&rwlock);
// ... read shared data ...
pthread_rwlock_unlock(&rwlock);

// Writer:
pthread_rwlock_wrlock(&rwlock);
// ... modify shared data ...
pthread_rwlock_unlock(&rwlock);
```

**Trade-off:** RWLocks are heavier than mutexes. The lock itself has more state (reader count). Contested RWLocks can starve writers if readers keep arriving. Use only when reads vastly outnumber writes and are long enough to justify the overhead.

---

### 8.4 Semaphores

A semaphore is a counter with atomic increment/decrement. Wait operations block when counter is 0.

```c
#include <semaphore.h>

sem_t sem;
sem_init(&sem, 0, 5);  // 0 = shared between threads, 5 = initial value

sem_wait(&sem);   // Decrement. Blocks if count == 0.
sem_trywait(&sem); // Non-blocking. Returns -1 with EAGAIN if would block.
sem_post(&sem);   // Increment. May wake a waiter.
sem_getvalue(&sem, &val); // Get current value (but may change immediately).

sem_destroy(&sem);
```

**Semaphore vs Mutex:**
- Mutex: binary (locked/unlocked). HAS OWNERSHIP. Only the thread that locked can unlock.
- Semaphore: counter (0, 1, 2...). NO OWNERSHIP. Any thread can `sem_post`.
- Mutex for mutual exclusion. Semaphore for signaling (producer→consumer) and resource counting (limited pool of N resources).

**Named semaphores (`sem_open`):** Can be used between processes (identified by name in `/dev/shm/`). Unnamed semaphores (`sem_init`) are for threads or forked processes sharing memory.

---

### 8.5 Atomic Operations — Lock-Free Foundations

Atomics guarantee that an operation on a single variable completes without any thread seeing an intermediate state.

```c
#include <stdatomic.h>

atomic_int counter = 0;

// Atomic operations:
atomic_fetch_add(&counter, 1);     // Atomic counter++ (returns OLD value)
atomic_fetch_sub(&counter, 1);     // Atomic counter--
int old = atomic_load(&counter);   // Atomic read
atomic_store(&counter, 0);         // Atomic write
atomic_compare_exchange_weak(&counter, &expected, desired); // CAS
```

**Compare-and-Swap (CAS) — the universal lock-free primitive:**
```c
// CAS logic: if *obj == expected, set *obj = desired and return true.
// Otherwise, set expected = *obj and return false.
bool atomic_compare_exchange_weak(atomic_int *obj, int *expected, int desired);
```

**The CAS pattern for lock-free updates:**
```c
int old = atomic_load(&shared_var);
int new_val;
do {
    new_val = compute_new_value(old);
    // Try to swap. If shared_var changed, old gets updated to current value.
    // Loop and recompute with the new old value.
} while (!atomic_compare_exchange_weak(&shared_var, &old, new_val));
// Operation succeeded atomically — no mutex needed.
```

**When to use atomics vs mutexes:**
- Atomics: For simple counters, flags, or single-variable updates.
- Mutexes: For complex state involving multiple variables that must be updated together.
- Never use atomics as a replacement for mutexes for multi-step operations. CAS is hard to get right.

---

### 8.6 Deadlock — The Four Conditions

Deadlock requires ALL four:
1. **Mutual Exclusion:** Resources can only be used by one thread at a time.
2. **Hold and Wait:** A thread holding a resource can wait for more.
3. **No Preemption:** Resources cannot be forcibly taken away.
4. **Circular Wait:** There exists a cycle of threads, each waiting for a resource held by the next.

**Break any one to prevent deadlock:**
- **Lock ordering** (breaks circular wait): All threads acquire locks in the SAME order. If you need A and B, always lock A then B. Never B then A.
- **Try-lock and backoff** (breaks hold and wait): Use `pthread_mutex_trylock`. If you can't get the lock, release all held locks and retry.
- **Lock hierarchy:** Design your system so locks form a tree. Always lock from root to leaf.

---

### 8.7 Cache Coherence and False Sharing

**MESI cache coherence protocol (simplified):**
Each cache line is in one of four states: Modified, Exclusive, Shared, Invalid. Cores communicate via a coherence bus to maintain consistency. When Core A writes to a line, it invalidates Core B's copy. Core B's next read of that line must fetch it from Core A or memory.

**False sharing:**
```c
struct counters {
    int counter_a;  // Core A updates this frequently
    int counter_b;  // Core B updates this frequently
};
// Both fields are on the same 64-byte cache line.
// Core A writes counter_a → invalidates entire line → Core B's counter_b access misses!
// Despite touching DIFFERENT variables, they behave like they're sharing.
```

**Fix 1: Padding:**
```c
struct counters {
    alignas(64) int counter_a;
    alignas(64) int counter_b;  // Each on its own cache line
};
```

**Fix 2: Separate allocations:**
```c
int *counter_a = aligned_alloc(64, sizeof(int));
int *counter_b = aligned_alloc(64, sizeof(int));
```

**What to notice:** False sharing is invisible in single-threaded code and nearly impossible to debug without profiling (`perf c2c`). It silently destroys multi-core scaling. Always be aware of cache line boundaries for frequently written, thread-private data.

---

## 9. Memory Systems — Hardware Up

### 9.1 The Memory Hierarchy

| Level | Size | Latency | Bandwidth | Managed By |
|---|---|---|---|---|
| L1 Cache | 32 KB | ~1 ns (~4 cycles) | ~1 TB/s | Hardware |
| L2 Cache | 256 KB | ~4 ns (~12 cycles) | ~500 GB/s | Hardware |
| L3 Cache | 8-32 MB | ~12 ns (~40 cycles) | ~200 GB/s | Hardware |
| RAM | GBs | ~60-100 ns (~200 cycles) | ~50 GB/s | OS + Hardware |
| SSD | TBs | ~10-100 μs | ~5 GB/s | OS |
| HDD | TBs | ~5-10 ms | ~200 MB/s | OS |

**The key insight:** Main memory is ~100x slower than L1 cache. A cache miss costs as much as ~200 arithmetic operations. Your real enemy is not CPU cycles — it's memory latency.

**What this means for systems programming:**
- Prefer contiguous arrays over linked structures (better cache locality)
- Access memory sequentially (hardware prefetcher helps)
- Pack data (avoid padding that wastes cache space)
- Avoid random memory access when data doesn't fit in cache
- A hash table that fits in L3 is massively faster than one that spills to RAM

---

### 9.2 Cache Lines and Associativity

**Cache line:** The unit of data transfer between cache and memory. 64 bytes on x86. Every memory access brings in an entire 64-byte line. Even if you only read 1 byte, you pay for 64.

**Cache associativity:** Where can a particular memory address go in the cache?
- **Direct-mapped:** Each address maps to exactly one cache slot. Simple but high conflict rate.
- **Fully associative:** Any address can go anywhere. Expensive hardware.
- **N-way set associative:** Cache divided into sets of N slots. An address maps to one set, can occupy any slot in that set. Typical: 8-way for L1 data, 16-way for L3.

**What to notice:** Accessing memory at strides that are multiples of the cache size can cause pathological conflict misses (aliasing). For example, on a 32KB 8-way L1 data cache, accessing elements separated by 4KB will map to the same set, evicting each other despite plenty of free cache space elsewhere.

---

### 9.3 TLB — The Virtual Memory Cache

**The TLB is a cache for virtual→physical address translations.** Every memory access (`mov eax, [rbx]`) requires:
1. Look up virtual page number → physical page number (TLB hit: ~1 cycle, miss: walk page table ~10-100 cycles)
2. Then access the data from cache/memory

**TLB sizes (typical):**
- L1 D-TLB: 64 entries, 4-way, covers 256 KB (64 × 4KB pages)
- L1 I-TLB: 64 entries, for code
- L2 STLB: 1536 entries, 12-way, covers 6 MB

**Huge pages (2MB and 1GB):**
- 2MB huge page: L1 TLB covers 128 MB instead of 256 KB.
- 1GB huge page: L2 STLB covers 1.5 TB.
- Use `mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_ANONYMOUS|MAP_HUGETLB, -1, 0)`.
- The cost: internal fragmentation (wasted memory). Only worthwhile for large, frequently-accessed data structures (in-memory databases, large hash tables).

**What to notice about TLB and data structures:**
- A 4-level page table walk is 4 memory accesses (~400 cycles) before you even get to your data.
- Data structures that touch many 4KB pages (e.g., large linked lists, pointer-heavy trees) can be TLB-bound — most time spent in page table walks.
- Contiguous arrays = fewer pages touched = fewer TLB misses.

---

### 9.4 Virtual Memory — Page Fault Flow

**The kernel-side flow of a minor page fault:**
1. CPU accesses virtual address, walks page table, finds entry "not present."
2. CPU traps to kernel (page fault exception).
3. Kernel looks up the address in the process's VMA list (Virtual Memory Areas — the kernel's accounting of what's mapped where).
4. If address is valid but just not in memory (e.g., COW, lazy allocation, swapped out):
   - Minor fault: allocate physical page, update page table, return to user.
   - COW: duplicate page, mark both writable, return.
   - Major fault: read page from disk, then proceed.
5. If address is invalid: deliver `SIGSEGV`.

**What to notice about `mmap` and overcommit:**
Linux defaults to memory overcommit. `malloc` can succeed even if there isn't enough physical RAM + swap. The memory is "promised" but not actually allocated until accessed. When you access, page faults happen. If the kernel can't allocate when you access, the OOM (Out-Of-Memory) killer terminates processes. Disable overcommit with `sysctl vm.overcommit_memory=2`.

---

## 10. Build Systems & Linking

### 10.1 Make — The Rules

**Basic Makefile anatomy:**
```makefile
target: prerequisites
	recipe command
	another recipe command
# Recipe lines MUST start with a TAB character, not spaces.
```

**Variables:**
```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -Werror -g -O2
LDLIBS  = -lm -lpthread
TARGET  = program
OBJS    = main.o utils.o network.o

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^ $(LDLIBS)

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

# Special variables:
# $@ = target name (program)
# $^ = all prerequisites (main.o utils.o network.o)
# $< = first prerequisite (main.c)
```

**Automatic dependency generation:**
```makefile
# Generate .d files that record which headers each .c file includes
CFLAGS += -MMD -MP

# Include the generated dependency files
-include $(OBJS:.o=.d)

# Now Make knows that main.o depends on main.c AND data.h, etc.
# If data.h changes, main.o is recompiled. Without this, you get stale objects.
```

**Phony targets:**
```makefile
.PHONY: all clean test install

all: $(TARGET)

clean:
	rm -f $(OBJS) $(TARGET) $(OBJS:.o=.d)

test: $(TARGET)
	./$(TARGET) --self-test
```

**What to notice:** Make is rule-based, not imperative. You describe dependencies. Make builds the DAG and executes commands in topological order. It checks modification timestamps to determine what's stale. This is declarative programming for builds.

---

### 10.2 Static Libraries (`.a` files)

**What a static library is:** An archive of `.o` files. `ar` is like `tar` for object files. The linker extracts needed object files and copies them into the final executable.

**Creating:**
```bash
gcc -c file1.c file2.c file3.c   # Compile to .o files
ar rcs libmylib.a file1.o file2.o file3.o   # Archive them
```

**Using:**
```bash
gcc main.c -L. -lmylib -o program
# -L. = search for libraries in current directory
# -lmylib = link with libmylib.a (lib prefix and .a suffix are implicit)
```

**What the linker does:** Scans the `.a` file. Finds unresolved symbols in `main.o`. Extracts only the `.o` files that define those symbols. Copies them into the executable. Other `.o` files in the archive are ignored. This is why you specify `.a` files AFTER the `.o` files on the `gcc` command line.

---

### 10.3 Shared Libraries (`.so` files)

**What a shared library is:** Code loaded at runtime by the dynamic linker (`ld.so`). Multiple processes can share the same physical pages of the library (read-only .text and .rodata sections are shared; .data and .bss get per-process copies via COW).

**Position Independent Code (`-fPIC`):**
- Shared libraries can be loaded at any address in any process.
- Normal (non-PIC) code has hardcoded addresses (e.g., `call 0x405000`).
- PIC uses relative addressing (e.g., `call *(%rip + offset)`) or indirect through the GOT (Global Offset Table).
- `-fPIC` on x86-64 has near-zero performance cost (thanks to RIP-relative addressing). Use it always for shared libraries.

**Creating:**
```bash
gcc -shared -fPIC -o libmylib.so file1.c file2.c
```

**Using:**
```bash
gcc main.c -L. -lmylib -o program
# On Linux, the linker will prefer .so over .a if both exist.

# At runtime, the dynamic linker must find libmylib.so.
# Ways to tell it where the library is:
# 1. Set LD_LIBRARY_PATH: export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH
# 2. Set rpath at link time: gcc ... -Wl,-rpath,/path/to/libs
# 3. Put the .so in standard locations: /lib, /usr/lib
# 4. Update /etc/ld.so.conf
```

**SO versioning conventions:**
```bash
libfoo.so           # symlink for development: links to latest
libfoo.so.1         # symlink for runtime: links to latest 1.x
libfoo.so.1.2.3     # real file
# Set soname at link time: gcc -Wl,-soname,libfoo.so.1
```

---

### 10.4 The Linker — Symbol Resolution

**How the linker works in 4 steps:**

1. **Symbol collection:** Reads all input `.o` files and `.a` files. Builds symbol tables: "defined" set and "undefined" (missing) set.
2. **Archive scanning:** For each `.a` file, if it defines any currently undefined symbol, extract that `.o` file. Repeat until no new symbols resolved.
3. **Resolution:** Matches undefined symbols to definitions. Throws "undefined reference" error for any unmatched symbol.
4. **Address assignment (relocation):** Assigns final virtual addresses to all sections. Patches all references to point to correct addresses.

**Common linker errors:**
- **"undefined reference to `foo`":** Function/variable declared but not defined (or `.o`/`.a` not linked).
- **"multiple definition of `foo`":** Two source files define the same global symbol. Use `static` on file-scope functions/variables. Define in one `.c`, declare `extern` in `.h`.
- **Order matters:** If `libA.a` uses symbols from `libB.a`, `libA` must come first on the link line. Circular dependencies require repeating libraries: `-lA -lB -lA`.

---

## 11. Debugging & Profiling — The Full Arsenal

### 11.1 GDB — Essential Commands Beyond Basics

**Starting and attaching:**
```bash
gdb ./program                  # Start fresh
gdb ./program core             # Analyze core dump
gdb -p $(pidof program)        # Attach to running process
```

**Breakpoints and watchpoints:**
```
break function_name            # Break at function entry
break file.c:42                # Break at line
break *0x401234                # Break at address
break main if argc > 5         # Conditional breakpoint
watch variable                 # Break when variable CHANGES (hardware watchpoint)
rwatch variable                # Break when variable is READ
display variable               # Print variable after every step
```

**Navigating crash sites:**
```
backtrace              # Full call stack
backtrace full         # With local variables
frame N                # Switch to frame N
info locals            # Show all local variables in current frame
info args              # Show function arguments
print *ptr             # Dereference pointer
print/x value          # Print in hex
print (struct mytype) *ptr  # Cast and print struct
x/10x $rsp             # Examine 10 hex words at stack pointer
x/s string_ptr         # Print as string
```

**Reverse debugging (rr — Record and Replay):**
```bash
rr record ./program            # Record execution
rr replay                      # Replay with GDB
reverse-continue               # Go backward to previous breakpoint
reverse-step                   # Step backward one line
```
Record once, debug infinitely. Deterministic replay. Invaluable for intermittent bugs.

---

### 11.2 Valgrind — Memory Error Detection

**Memcheck (the default tool):**
```bash
valgrind --leak-check=full --show-leak-kinds=all --track-origins=yes ./program
```

**What Memcheck detects:**
- Reading uninitialized memory
- Reading/writing after `free` (use-after-free)
- Reading/writing past the end of `malloc`'d blocks (buffer overflow)
- Memory leaks (still reachable, definitely lost, indirectly lost, possibly lost)
- Double `free`
- `free` of non-heap memory (stack, global)
- Passing overlapping pointers to `memcpy` (use `memmove` instead)

**Understanding leak types:**
- **Definitely lost:** No pointer to the memory anywhere. Real leak.
- **Indirectly lost:** Lost because a containing struct was lost.
- **Possibly lost:** Interior pointer exists (pointer to middle of block). Check if this is intentional.
- **Still reachable:** Pointer exists but wasn't freed at exit. Usually not a problem for short-lived programs.

**Valgrind with GDB:**
```bash
valgrind --vgdb=yes --vgdb-error=0 ./program
# In another terminal:
gdb ./program
target remote | vgdb
```
Stops program at the first error. Full GDB session.

---

### 11.3 Sanitizers — Fast Runtime Checks

**AddressSanitizer (ASan):** Compile with `-fsanitize=address`. Detects use-after-free, buffer overflows, use-after-return, double-free. ~2x slowdown (much faster than Valgrind's ~20x). Use during development and testing.

**UndefinedBehaviorSanitizer (UBSan):** `-fsanitize=undefined`. Catches integer overflow, null pointer dereference, shift out of range, division by zero, misaligned access, etc. Negligible performance overhead for most checks. ALWAYS use during development.

**ThreadSanitizer (TSan):** `-fsanitize=thread`. Detects data races. ~5-10x slowdown. Extremely effective — reports exact file/line of conflicting accesses. Incompatible with ASan.

**MemorySanitizer (MSan):** `-fsanitize=memory`. Detects use of uninitialized memory. Requires all linked libraries to also be MSan-instrumented. Harder to set up but catches subtle bugs.

**Combined usage:**
```bash
gcc -g -fsanitize=address,undefined -o program *.c
./program  # ASan and UBSan run together. Detailed error messages with source locations.
```

---

### 11.4 `perf` — CPU Profiling

**Counting events:**
```bash
perf stat ./program           # Instructions, cycles, cache misses, branch mispredicts
perf stat -d ./program        # More detailed stats (L1 cache misses, LLC misses)
perf stat -e cycles,instructions,cache-misses,cache-references ./program
```

**Sampling profiler (find hotspots):**
```bash
perf record ./program          # Record samples at ~4 kHz
perf report                    # Interactive TUI: sort by function, see call graph
perf record -g ./program       # With call graph (stack unwinding)
```

**Key metrics in `perf report`:**
- **Overhead:** Percentage of samples in this function
- **Self:** Time spent in this function itself (not children)
- **Children:** Time in this function + all functions it calls

**Annotate assembly:**
```bash
perf annotate function_name    # Show assembly with sample counts per instruction
# See exactly which instruction is hot. Useful for micro-optimization.
```

**Cache profiling:**
```bash
perf record -e cache-misses ./program
perf record -e LLC-load-misses -e LLC-store-misses ./program
```

---

### 11.5 `strace` and `ltrace`

**strace — trace system calls:**
```bash
strace ./program                              # All syscalls
strace -e trace=read,write ./program          # Only read/write
strace -e trace=file                          # File-related syscalls
strace -c ./program                           # Summary table with counts and times
strace -p PID                                 # Attach to running process
strace -f ./program                           # Follow fork'd children
strace -o output.log ./program                # Save to file
```

**What strace tells you that you can't get otherwise:**
- What files a program actually opens (and whether they succeed)
- Whether network calls are blocking or non-blocking
- Signal delivery
- `ENOMEM` from `brk`/`mmap` — why allocation failed
- Time spent in each syscall (`strace -c`)

**ltrace — trace library calls:**
```bash
ltrace ./program                    # Dynamic library calls (malloc, printf, etc.)
ltrace -e 'malloc+free' ./program   # Only malloc/free family
```
Shows you what the program calls in shared libraries — useful for seeing allocation patterns or debugging why something crashes in libc.

---

### 11.6 Binary Analysis Tools

```bash
nm program                 # List all symbols (functions, globals) with addresses
nm -u program              # Show only undefined symbols (what needs linking)
objdump -d program         # Disassemble entire text section
objdump -d -S program      # Interleave source code with assembly (compile with -g)
objdump -t program         # Show symbol table
objdump -x program         # Show all headers
readelf -h program         # ELF header
readelf -S program         # Section headers
readelf -l program         # Program headers (segments loaded into memory)
readelf -d program         # Dynamic section (shared libraries needed)
ldd program                # Show shared library dependencies and where they're found
size program               # Show size of text, data, bss sections
file program               # Identify file type and architecture
```

---

## 12. Embedded Systems & RTOS

### 12.1 Cross-Compilation

**What cross-compilation means:** Your development machine (the "host") has a different CPU architecture than the target device. The compiler runs on x86 but generates ARM/RISC-V/MIPS/etc code.

**Toolchain components:**
- Compiler: `arm-none-eabi-gcc` (for bare-metal ARM), `arm-linux-gnueabihf-gcc` (for ARM Linux)
- Assembler: part of gcc
- Linker: `arm-none-eabi-ld` (rarely called directly; gcc invokes it)
- `objcopy`: Converts ELF → raw binary (`.bin`), Intel HEX (`.hex`), or Motorola S-record (`.srec`)
- `objdump`, `nm`, `size`: Same tools, target-specific versions

**The linker script:** On bare-metal, you must tell the linker where things go. A minimal linker script:
```ld
MEMORY {
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 512K
    RAM   (rw) : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS {
    .text : { *(.text*) } > FLASH       /* Code in flash */
    .rodata : { *(.rodata*) } > FLASH   /* Read-only data in flash */
    .data : { *(.data*) } > RAM AT > FLASH  /* Init'd data copied to RAM */
    .bss : { *(.bss*) } > RAM           /* Zero-init'd data in RAM */
}
```

**The startup code (`crt0`):** Before `main` runs:
1. Set stack pointer to top of RAM.
2. Copy `.data` section from flash to RAM.
3. Zero `.bss` section in RAM.
4. Call `main`.
5. If/when `main` returns, loop forever or reset.

---

### 12.2 Memory-Mapped I/O

**The concept:** Hardware registers are accessed like memory — read/write specific addresses to control peripherals.

```c
// Define register addresses from the datasheet
#define GPIOA_BASE    0x40020000
#define GPIOA_MODER   ((volatile uint32_t *)(GPIOA_BASE + 0x00))
#define GPIOA_ODR     ((volatile uint32_t *)(GPIOA_BASE + 0x14))

// Set pin 5 as output
*GPIOA_MODER &= ~(3 << (5 * 2));  // Clear bits
*GPIOA_MODER |= (1 << (5 * 2));   // Set to output mode

// Toggle pin 5
*GPIOA_ODR ^= (1 << 5);  // XOR to toggle
```

**What to notice:** `volatile` is required because the compiler would otherwise optimize away "redundant" reads/writes. Reading a hardware status register that changes between reads would be optimized to a single read if not `volatile`.

**Read-Modify-Write hazards:**
`*REG |= BIT;` is read-modify-write. If the hardware changes the register between the read and write, you clobber the change. For clear-on-read or write-1-to-clear registers, prefer:
- Write whole value: `*REG = VALUE;` (not RMW)
- Use bit-band regions on Cortex-M (atomic bit operations)

---

### 12.3 RTOS Concepts — FreeRTOS

**Tasks:**
```c
void task_function(void *params) {
    while (1) {
        // Do work
        vTaskDelay(pdMS_TO_TICKS(100));  // Block for 100ms
    }
}

xTaskCreate(task_function, "TaskName", 256, NULL, 1, NULL);
// Stack size in WORDS (not bytes). Priority 1 (0 = lowest).
// NULL = no task handle returned.
```

**Task states:**
- **Running:** Currently using CPU
- **Ready:** Able to run but another task is Running
- **Blocked:** Waiting for event (delay, semaphore, queue)
- **Suspended:** Explicitly suspended by `vTaskSuspend()`. Not scheduled.

**Scheduling:** Preemptive priority-based. The highest priority Ready task always runs. Same-priority tasks time-slice (if `configUSE_TIME_SLICING = 1`). The tick interrupt (`configTICK_RATE_HZ`) drives scheduling decisions.

**Queues (inter-task communication):**
```c
QueueHandle_t queue = xQueueCreate(10, sizeof(my_data_t));
// 10 items, each size sizeof(my_data_t)

xQueueSend(queue, &data, portMAX_DELAY);  // Block until send succeeds
xQueueReceive(queue, &data, pdMS_TO_TICKS(100));  // Wait up to 100ms
```

**Semaphores:**
```c
SemaphoreHandle_t sem = xSemaphoreCreateBinary();  // Takes (0) initially
xSemaphoreGive(sem);                                // Signal from ISR or task
xSemaphoreTake(sem, portMAX_DELAY);                  // Wait (task blocks)
```

**Mutexes (with priority inheritance):**
```c
SemaphoreHandle_t mutex = xSemaphoreCreateMutex();  // Takes (1) initially
xSemaphoreTake(mutex, portMAX_DELAY);                // Lock
// Critical section
xSemaphoreGive(mutex);                               // Unlock
// Priority inheritance: if a high-priority task blocks on mutex held by a
// low-priority task, the low-priority task temporarily inherits the high priority
// to prevent priority inversion.
```

**What to notice in ISRs (Interrupt Service Routines):**
- Functions called from ISRs MUST use the `FromISR` variants: `xQueueSendFromISR`, `xSemaphoreGiveFromISR`
- Never call blocking functions from ISRs
- Always call `portYIELD_FROM_ISR` if a task was woken by the ISR (for preemption)
- Keep ISRs as short as possible. Defer processing to tasks.

---

## 13. Security Fundamentals for Systems Programmers

### 13.1 Buffer Overflows — The Classic Vulnerability

**Stack buffer overflow:**
```c
void vulnerable(char *input) {
    char buf[64];
    strcpy(buf, input);  // No bounds check. Input > 64 bytes overwrites return address.
}
// Attacker provides: 64 bytes padding + address of shellcode + shellcode
```

**Defenses:**
- Use `strncpy`, `snprintf`, `fgets` with size limits.
- Stack canaries (`-fstack-protector`): Compiler inserts random value between locals and return address. Checked before `ret`. Overflow overwrites canary → abort.
- NX bit: Mark stack non-executable. Shellcode on stack won't run.
- ASLR (Address Space Layout Randomization): Randomize positions of stack, heap, libraries. Harder to predict addresses.
- `FORTIFY_SOURCE`: Compiler replaces unsafe libc functions (strcpy, memcpy) with bounds-checked versions when size is known.

### 13.2 Integer Security

**Integer overflow leading to under-allocation:**
```c
// User provides num_items and item_size (can be attacker-controlled)
size_t total = num_items * item_size;  // Might overflow!
void *buf = malloc(total);  // Allocates less than expected
// Attacker gets heap overflow from "in-bounds" accesses to buf.
```

**The fix:** Check for overflow before multiplication:
```c
if (num_items > 0 && item_size > SIZE_MAX / num_items) return NULL;  // Overflow
size_t total = num_items * item_size;
void *buf = malloc(total);  // Safe
```

**Signed-unsigned confusion:**
```c
int len = get_user_length();  // Might be negative!
char buf[256];
if (len < sizeof(buf))       // len is signed, sizeof is unsigned. 
    memcpy(buf, data, len);  // len cast to unsigned → huge number → overflow
```

**The fix:** Always use `size_t` for sizes. Validate that signed-to-unsigned conversion doesn't produce huge values.

### 13.3 TOCTOU (Time of Check, Time of Use)

```c
// Check:
if (access("file", W_OK) == 0) {  // Is file writable?
    // Wait — attacker swaps file for symlink to /etc/passwd
    int fd = open("file", O_WRONLY);  // Use: opens the symlink target!
}
```

**Fix:** Don't check and then use. Open first, then check the file descriptor (`fstat`):
```c
int fd = open("file", O_WRONLY);
if (fd >= 0) {
    struct stat st;
    fstat(fd, &st);  // Check fd, not the path
    if (st.st_uid == getuid()) { /* safe to write */ }
}
```

### 13.4 Secure Coding Practices

1. **Validate all inputs.** Assume every byte from a network, file, or user is malicious.
2. **Use bounded operations.** `strncpy` (but watch for non-null-termination), `snprintf`, `fgets`, `strlcpy` (BSD).
3. **Check return values.** Every `malloc`, `open`, `read`, `write` can fail. Handle it.
4. **Free is your friend.** Free resources on error paths too. Use `goto` for cleanup (C doesn't have RAII).
5. **Don't trust sizes.** Verify sizes before copying. Check for overflow.
6. **Least privilege.** Drop root as soon as you no longer need it (daemons).
7. **Compiler defenses:** `-fstack-protector-strong -D_FORTIFY_SOURCE=2 -fPIE -pie -Wl,-z,relro -Wl,-z,now`

---

## 14. Projects — Complete Specifications

### Project 1: Custom Memory Allocator

**Specification:** Implement `my_malloc(size_t size)` and `my_free(void *ptr)`.

**How it works (first-fit free list):**
- Pre-allocate a large buffer (e.g., 1 MB) using `sbrk` or static array.
- Maintain a free list: linked list of free blocks. Each block has a header (size, whether free, pointer to next free block) before the user data.
- `my_malloc`: Walk the free list. Find first free block large enough for requested size + header. Split if significantly larger. Mark allocated. Return pointer to data (after header).
- `my_free`: Find the header (pointer just before the ptr). Mark free. Coalesce with adjacent free blocks if possible (check if next/previous blocks are free by their headers).

**What you'll learn:**
- Pointer arithmetic at its finest
- Alignment issues (must return pointers aligned to worst-case alignment, typically 16 bytes on x86-64)
- Fragmentation (your allocator will fragment over time)
- Coalescing logic
- The cost of traversing a free list

**Extensions:** Implement best-fit or next-fit. Add free list binning (separate lists for different size classes). Implement `my_realloc`. Add debug mode that tracks caller file/line and detects double-free.

---

### Project 2: Mini Shell

**Specification:** Implement a Unix shell.

**Core loop:**
1. Print prompt. Read line of input.
2. Parse command: split by pipes (`|`), then by spaces. Handle quoted strings.
3. For each command in the pipeline:
   - If it's a builtin (`cd`, `exit`, `export`): execute directly.
   - Otherwise: `fork`, `execvp`, parent continues.
4. For pipelines: create pipes between commands. First command writes to pipe, second reads, etc. Close file descriptors properly.

**What you'll learn:**
- `fork`/`exec`/`waitpid` in depth
- File descriptor manipulation (`dup2`, `pipe`)
- Process groups and job control (optional: `setpgid`, `tcsetpgrp`)
- Signal handling (`SIGINT` should kill foreground process, not shell; `SIGCHLD` to reap zombies)
- Parsing (lexing a command line)

**Required features:**
- Command execution with arguments
- `PATH` search (try each directory in PATH until executable found)
- `cd` builtin (must change the shell's own working directory — can't be done in a child process)
- Pipe operator `|`
- Redirection: `>`, `<`, `>>`
- Background execution: `&`
- Signal handling: Ctrl-C doesn't kill the shell

**Advanced:** job control (`fg`, `bg`, `jobs`), history, tab completion, scripting.

---

### Project 3: HTTP/1.0 Server with Thread Pool

**Specification:** Serve static files over HTTP.

**Architecture:**
1. Main thread: creates listen socket, accepts connections, dispatches to thread pool.
2. Thread pool: N worker threads sharing a task queue. Main thread enqueues accepted fd. Workers dequeue and handle.
3. Each worker reads the HTTP request, parses method + path + headers, responds with file contents or error.

**What you'll learn:**
- Thread pool design (producer-consumer pattern)
- Mutex and condition variable synchronization
- HTTP protocol parsing (CRLF-delimited, header fields)
- File serving (open, stat for size, read and write over socket)
- Graceful shutdown (signal handler sets flag, threads check flag, join)

**Required HTTP features:**
- Parse `GET` method
- Parse request URI, decode percent-encoded characters
- Serve static files from a document root directory
- Return `200 OK` with `Content-Type` (detect from file extension)
- Return `404 Not Found` for missing files
- Return `405 Method Not Allowed` for non-GET
- Return `400 Bad Request` for malformed requests
- Handle partial reads/writes correctly
- Connection: close after each request (HTTP/1.0 default)

---

### Project 4: Container Runtime (Mini-Docker)

**Specification:** Isolate a process using Linux namespaces and cgroups.

**Architecture:**
1. Parse a config file specifying: filesystem image, command, resource limits (memory, CPU).
2. `unshare` or `clone(CONTAINER_FLAGS)` to create namespaces.
3. `pivot_root` into the container's filesystem.
4. Set cgroup limits by writing to `/sys/fs/cgroup/...`.
5. Drop capabilities (set minimally needed `CAP_*`).
6. `execve` the container's command.

**Required namespaces:**
- `CLONE_NEWNS`: Mount namespace — the container gets its own filesystem tree
- `CLONE_NEWUTS`: Hostname — container can have its own hostname
- `CLONE_NEWIPC`: IPC — container's SysV IPC is isolated
- `CLONE_NEWPID`: PID — container sees itself as PID 1
- `CLONE_NEWNET`: Network — container gets its own interfaces (optional, need to set up veth pairs)

**What you'll learn:**
- Linux namespace internals
- cgroup V1 interface (reading/writing sysfs files)
- `clone()` syscall directly (much more control than `fork`)
- `pivot_root` (replaces `chroot` — not escapable)
- Capabilities (fine-grained root privilege splitting)
- The internals of what Docker actually does

**Minimal version:**
1. Download a minimal rootfs (Alpine or BusyBox).
2. Use `clone(CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWPID)` to create a child.
3. Child: `mount("none", "/", NULL, MS_REC | MS_PRIVATE, NULL)` — make mount namespace private.
4. Child: `mount("rootfs", "/new_root", NULL, MS_BIND, NULL)` — bind mount rootfs.
5. Child: `pivot_root("/new_root", "/new_root/.put_old")` — swap root.
6. Child: `execve("/bin/sh", ...)` — run shell.

---

## 15. Interview Deep-Dive

### 15.1 The Questions Behind the Questions

**"Implement `memcpy`"** — They're testing:
- Can you handle overlapping buffers? (The standard says `memcpy` doesn't handle overlap; `memmove` does. Know the difference.)
- Do you check for NULL? (For `memcpy(dst, src, n)`, NULL + n=0 is technically decided by implementation. Defensive code checks.)
- Do you optimize for aligned access? (Word-copy for aligned bulk, byte-copy for remainder.)

**Flow of a good `memcpy`:**
```
1. Check for NULL pointers if n > 0.
2. If src and dst are both word-aligned:
   - Copy words (4 or 8 bytes) in a loop.
   - Copy remaining bytes.
3. Else:
   - Copy byte-by-byte.
```

**"Implement a thread-safe queue"** — They're testing:
- Do you protect both enqueue and dequeue with a mutex?
- Do you use condition variable to avoid busy-waiting on empty queue?
- Do you handle spurious wakeups with a while loop?
- Do you handle the queue-full case (bounded vs unbounded)?
- Can you reason about which thread wakes up? (Signal wakes ONE unspecified waiter; broadcast wakes all.)

**"What happens when you type `ls -l | grep foo` in the shell?"**
```
1. Shell forks twice.
2. Creates a pipe.
3. Child 1: dup2(pipe_write, stdout). Close pipe fds. exec("ls", "-l").
4. Child 2: dup2(pipe_read, stdin). Close pipe fds. exec("grep", "foo").
5. Parent closes both pipe fds. waitpid for both children.
6. ls writes to stdout → goes to pipe. grep reads from pipe → reads ls's output.
```

---

### 15.2 Coding Interview Patterns in C

**String reversal:**
```
1. Two pointers: start and end.
2. While start < end: swap, start++, end--.
3. Watch for empty string (NULL or first char is '\0').
4. In-place. O(n) time, O(1) space.
```

**Detect cycle in linked list (Floyd's algorithm):**
```
1. Two pointers: slow (moves 1) and fast (moves 2).
2. While fast != NULL and fast->next != NULL:
   - slow = slow->next, fast = fast->next->next
   - If slow == fast: cycle detected.
3. To find cycle start: reset slow to head. Move both one step at a time. Meeting point = cycle start.
```

**Binary search in sorted array:**
```
1. left = 0, right = n - 1
2. While left <= right:
   - mid = left + (right - left) / 2  (overflow-safe)
   - If arr[mid] == target: return mid
   - If arr[mid] < target: left = mid + 1
   - Else: right = mid - 1
3. Return -1 (not found)
```

**Check if a string is a valid integer (before `atoi`):**
```
1. Skip leading whitespace.
2. Optional sign (+ or -).
3. Must have at least one digit. Skip leading zeros.
4. Read digits, checking for overflow before each multiplication by 10.
5. Any non-digit after digits → invalid.
```

---

## 16. Learning Methodology

### 16.1 The Systems Programmer's Learning Loop

1. **Read the man page.** Not a tutorial. Not a blog. The actual man page. `man 2 open`, `man 3 malloc`, `man 7 epoll`. They're dense but complete. Every call's error conditions are listed. Read them.

2. **Write the code.** Small, focused programs that exercise exactly one concept. `fork` a child and `wait`. `mmap` a file and read it. Create a TCP socket and `connect` to google.com:80. Run it. Make it crash. Understand why.

3. **Read the assembly.** `gcc -S -O2 file.c`. For every interesting function. What did the compiler do to your loops? Were your function calls inlined? Did your array access get bounds-checked? (Spoiler: no.) This is the only way to truly understand what your C code becomes.

4. **Run it under strace and GDB.** Does the program make the syscalls you expect? Where does it spend time? Step through the critical path.

5. **Break it.** Overflow buffers. Dereference NULL. Use after free. Forget to `close`. Then run with sanitizers. Watch them catch it. Internalize the error messages.

### 16.2 What to Read (In Order)

1. **K&R "The C Programming Language"** — The short, dense original. Read it cover to cover. Do every exercise. Write your own versions of the programs.

2. **CS:APP "Computer Systems: A Programmer's Perspective"** — The single best systems book. Explains data representation, assembly, memory hierarchy, linking, exceptional control flow, virtual memory, I/O, network programming, and concurrent programming. Do the labs (bomb lab, buffer lab, malloc lab, proxy lab).

3. **Kerrisk "The Linux Programming Interface"** — 1500+ pages. Exhaustive Linux API reference. Read the chapters on the topics you're actively working on. Not a book to read linearly.

4. **Stevens "Advanced Programming in the UNIX Environment"** — The classic. Older but the fundamentals don't change. Excellent for POSIX concurrency chapters.

5. **Stevens "TCP/IP Illustrated, Volume 1"** — The definitive guide to TCP/IP. Understand the wire-level protocols.

6. **ostep.org** — "Operating Systems: Three Easy Pieces." Free online. The clearest OS textbook. Read all three pieces: Virtualization, Concurrency, Persistence.

### 16.3 Online Resources

- **`man` pages:** `man 2` for syscalls, `man 3` for library functions, `man 7` for overviews. `apropos keyword` to find relevant pages. Always your first stop.
- **CS:APP labs:** [csapp.cs.cmu.edu](http://csapp.cs.cmu.edu). Do them. The malloc lab will teach you more about memory than 10 tutorials.
- **xv6:** MIT's reimplementation of Unix V6 for x86 and RISC-V. Under 10,000 lines of readable C. Read the entire source. Understand how a real (small) OS works end-to-end.
- **LWN.net:** Linux Weekly News. Kernel development articles. In-depth technical writing about Linux internals.
- **Linux kernel source:** Especially `include/linux/list.h` (the linked list implementation), `fs/` (filesystems), `mm/` (memory management). You don't need to contribute — just reading is educational.

---

## The Honest Truth — Extended

Systems programming is a long game. There is no "learn C in 21 days" or "master Linux in a bootcamp." The material is deep because the machine is deep. Every layer — hardware, kernel, libc, compiler — makes no allowances for ignorance.

You will spend hours debugging a `SIGSEGV` caused by a single wrong pointer. You will lose an afternoon to a race condition that appears once in a thousand runs. You will stare at assembly wondering why your obvious optimization made things worse.

This is normal. This is the work. Every systems programmer who's any good has a graveyard of core dumps behind them.

The filter is not intelligence. It is persistence. The willingness to say "I don't understand this yet, but I will" — and then reading the spec, running the experiment, looking at the disassembly, until you do.

**Build things.** A memory allocator. A shell. An HTTP server. A container runtime. Each one forces you to confront reality — the reality of the C abstract machine, the kernel ABI, the CPU's memory model. Each one makes you a better programmer, whether you're writing embedded firmware or cloud infrastructure.

Most programmers float above the machine, using abstractions they don't understand. You are choosing to go down. To understand the stack from transistors to HTTP. That knowledge compounds. Every layer you understand makes the next one easier and the previous one richer.

Good luck. Read the man page. Run the sanitizers. Trust the disassembly. Build things that matter.
