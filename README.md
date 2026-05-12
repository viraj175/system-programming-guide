# Systems Programming Roadmap — C & Linux

> A deep, honest guide for becoming a genuine systems programmer.
> Not a checklist. A map with terrain.

---

## Table of Contents

1. [C Language Mastery](#1-c-language-mastery)
2. [Pointers — The Real Filter](#2-pointers--the-real-filter)
3. [Data Structures in Pure C](#3-data-structures-in-pure-c)
4. [Algorithms](#4-algorithms)
5. [Linux Internals](#5-linux-internals)
6. [System Calls & POSIX APIs](#6-system-calls--posix-apis)
7. [Networking](#7-networking)
8. [Tooling](#8-tooling)
9. [Build Systems](#9-build-systems)
10. [Memory Debugging](#10-memory-debugging)
11. [Concurrency](#11-concurrency)
12. [Embedded & RTOS (Optional)](#12-embedded--rtos-optional)
13. [Projects](#13-projects)
14. [Interview Preparation](#14-interview-preparation)
15. [Books & Resources](#15-books--resources)
16. [Expected Salary (India)](#16-expected-salary-india)

---

## 1. C Language Mastery

C is not a high-level language with training wheels. Every line you write has a direct machine consequence. You must understand not just what the code does but what the **compiler generates** and what the **CPU executes**.

### Variables & Storage Classes

Every variable in C has a **storage class** that determines its lifetime, visibility, and where in memory it lives.

| Storage Class | Keyword | Lives In | Lifetime | Default Value |
|---|---|---|---|---|
| Automatic | `auto` (implicit) | Stack | Function scope | Garbage |
| Static (local) | `static` | Data segment | Entire program | Zero |
| Static (global) | `static` on globals | Data segment | Entire program | Zero |
| External | `extern` | Data segment | Entire program | Zero |
| Register | `register` | Register (hint only) | Function scope | Garbage |

`static` on a local variable is deeply useful — it persists its value between function calls without being global. This is how you build counters, caches, or stateful functions without polluting global scope.

`register` is a hint to the compiler to keep this variable in a CPU register instead of memory for faster access. Modern compilers completely ignore it — they are better at register allocation than any human. The only practical effect today is that you cannot take the address of a `register` variable (`&x` is illegal). Do not use it in new code.

`extern` tells the compiler "this variable is defined in another translation unit." It is a declaration, not a definition.

**Translation unit:** A single `.c` file after the preprocessor has run — meaning all `#include`d headers are already expanded into it. The compiler processes one translation unit at a time and produces one `.o` object file per unit. This is why two `.c` files cannot see each other's `static` globals — `static` on a global limits its visibility to its own translation unit.

### Stack vs Heap

This is the most important memory concept in C.

**Stack:**
- Memory allocated automatically when a function is called, freed when it returns.
- Fast — it's just a pointer decrement.
- Fixed size (typically 8 MB on Linux, configurable with `ulimit -s`).
- Local variables, function arguments, and return addresses live here.
- Overflowing it causes a segmentation fault (stack overflow).

**Heap:**
- Memory you request manually with `malloc`/`calloc`/`realloc` and release with `free`.
- Slow compared to stack — involves system calls and bookkeeping.
- Size limited by available virtual memory.
- If you forget to `free`, you have a memory leak.
- If you `free` too early and access again, use-after-free — undefined behavior.

**Memory Layout of a C Program (low to high address):**
```
Text segment      ← compiled machine code (read-only)
Data segment      ← initialized globals and statics
BSS segment       ← uninitialized globals (zeroed at startup)
Heap              ← grows upward (malloc/free)
...               ← free space
Stack             ← grows downward (function calls)
Kernel space      ← not accessible from user space
```

### Structs, Unions, Enums

**Structs** group related data. Fields are laid out sequentially in memory, but the compiler may insert **padding** between fields to satisfy alignment requirements.

```c
struct example {
    char  a;    // 1 byte
                // 3 bytes padding (to align next field to 4-byte boundary)
    int   b;    // 4 bytes
    char  c;    // 1 byte
                // 7 bytes padding (to align struct size to 8-byte boundary on 64-bit)
    double d;   // 8 bytes
};
// sizeof(struct example) = 24, not 14
```

You can reorder fields (smallest to largest) to minimize padding naturally. Or use `__attribute__((packed))` to tell the compiler to eliminate all padding entirely.

**What `__attribute__((packed))` actually does:** It forces the compiler to place each field at the next byte after the previous one, ignoring alignment requirements. The struct becomes exactly `sizeof` all its fields summed. The cost: the CPU must do extra work to load a misaligned field (x86 handles this in hardware with a performance hit; ARM will trap and crash without special handling). Use it only when you must match an exact binary layout — like a network packet header or a hardware register map — never for performance.

**Unions** share the same memory for all members. The size is that of the largest member. Only one field is "valid" at a time.

**Type-punning** means reading memory you wrote as one type through a different type. For example: write a `float`, read it back as `unsigned int` to inspect its raw bits. This is what unions are legitimately used for in C. Write through one member, read through another — the standard permits this in C (unlike C++).

```c
union float_bits {
    float    f;
    uint32_t bits;
};

union float_bits u;
u.f = 3.14f;
printf("%08x\n", u.bits);  // inspect the raw IEEE 754 bits — valid C
```

Using pointer casting to type-pun (`*(int*)&my_float`) violates the strict aliasing rule and is undefined behavior. The union approach is the correct way in C.

**Enums** are just named integers. The underlying type is `int`. Use them for readability, not type safety — C does not enforce that you only assign valid enum values.

### typedef

`typedef` creates a type alias. It does not create a new type — it is purely a name.

```c
typedef struct { int x; int y; } point_t;
typedef unsigned long size_t;
typedef int (*compare_fn)(const void *, const void *);  // function pointer type
```

The `_t` suffix is a convention for typedefs. It is not required but widely used in C standard library and POSIX.

### Macros

Macros are handled by the **preprocessor** before compilation. They are text substitution, not functions. This has consequences.

```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))
```

Always parenthesize everything in macros. `MAX(x++, y++)` expands to `((x++) > (y++) ? (x++) : (y++))` — the increment happens twice. This is why macros are dangerous with side effects.

Use `static inline` functions instead of macros whenever possible. They are type-safe and debuggable.

**What `inline` does:** It is a hint to the compiler to copy the function's machine code directly into every call site instead of generating a real function call with a stack frame. This eliminates the call overhead — no pushing arguments, no `call` instruction, no `ret`. For tiny functions called in hot loops (like `MAX`, `MIN`, `clamp`), this matters. The compiler is free to ignore the hint.

**Why `static` is paired with it:** Without `static`, an `inline` function in a header creates a symbol visible to every translation unit that includes that header. The linker may then complain about multiple definitions. `static inline` makes each translation unit get its own private copy of the function, so there is no symbol conflict. The combination `static inline` in a header file is the standard pattern for small utility functions.

```c
// In a header file — correct
static inline int max(int a, int b) {
    return a > b ? a : b;
}
// vs macro: no double-evaluation, has a type, shows up in gdb
```

`#define`, `#include`, `#ifdef`, `#ifndef`, `#endif`, `#pragma` are all preprocessor directives. Understanding the preprocessing phase is part of understanding the full compilation pipeline.

### Compilation Pipeline

When you run `gcc file.c -o program`, four things happen:

1. **Preprocessing** (`gcc -E`) — expands macros, resolves `#include`, handles `#ifdef`. Produces a `.i` file.
2. **Compilation** (`gcc -S`) — translates C to assembly. Produces a `.s` file.
3. **Assembly** (`as`) — translates assembly to machine code. Produces a `.o` object file.
4. **Linking** (`ld`) — combines object files and libraries into an executable.

Understanding this pipeline lets you debug compilation errors at the right stage, inspect generated assembly, and understand why linking fails.

### Static vs Dynamic Linking

**Static linking:** Library code is copied directly into your executable at link time. The binary is self-contained but larger. No runtime dependency.

**Dynamic linking:** The executable only stores a reference to the library. The OS loads the shared library (`.so` file) at runtime. Smaller binary, but dependent on the library being present.

`ldd your_program` shows which shared libraries a binary depends on.

### Undefined Behavior

UB in C is not "crashes your program." It is **the compiler is allowed to assume this never happens**, and may generate code that produces any result — including no crash, subtly wrong output, or security vulnerabilities.

Common sources of UB:
- Signed integer overflow (use unsigned or check before overflow)
- Dereferencing a NULL or dangling pointer
- Out-of-bounds array access
- Using uninitialized variables
- Modifying a string literal
- Shifting by more than the bit width

Compile with `-fsanitize=undefined` to catch UB at runtime during development.

### `volatile` and `const`

`volatile` tells the compiler "do not optimize accesses to this variable — it may change outside the program's control." Used for memory-mapped hardware registers and variables modified by signal handlers.

`const` means you promise not to modify the value through this pointer/variable. It helps the compiler optimize and catches accidental modifications. `const` correctness — applying it consistently throughout your API — is a mark of a careful C programmer.

---

## 2. Pointers — The Real Filter

Pointers are what separates people who "know C syntax" from people who understand the machine.

### What a Pointer Actually Is

A pointer is just an integer that the CPU treats as a memory address. On a 64-bit system, every pointer is 8 bytes regardless of what it points to.

```c
int x = 42;
int *p = &x;   // p holds the address of x
*p = 99;       // dereference: writes 99 to the address p holds
```

`&` — address-of operator. Gives you the address of a variable.
`*` — dereference operator. Follows the pointer to the value it points to.

### Pointer Arithmetic

When you add `n` to a pointer, it advances by `n * sizeof(type)` bytes — not `n` bytes. This is why pointer arithmetic is type-aware.

```c
int arr[5] = {10, 20, 30, 40, 50};
int *p = arr;
p++;            // advances 4 bytes (sizeof int), points to arr[1]
*(p + 2);       // arr[3] = 40
```

`arr[i]` is literally defined as `*(arr + i)`. They are identical. The subscript is syntactic sugar for pointer arithmetic.

### Double Pointers

A pointer to a pointer. Necessary when you want a function to modify a pointer (e.g. allocate memory and return it through a parameter).

```c
void allocate(int **pp, int size) {
    *pp = malloc(size * sizeof(int));  // modifies the caller's pointer
}

int *arr = NULL;
allocate(&arr, 10);  // &arr is int**
```

Without `**`, the function would receive a copy of the pointer and any modification would be local only.

### Function Pointers

Pointers that point to executable code instead of data. Essential for callbacks, dispatch tables, and plugin architectures.

```c
int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }

int (*op)(int, int);  // function pointer: takes two ints, returns int
op = add;
op(3, 4);             // calls add(3, 4) → 7
op = sub;
op(3, 4);             // calls sub(3, 4) → -1
```

`qsort` in the standard library takes a function pointer for the comparator — this is the classic use case.

### Arrays vs Pointers — The Critical Distinction

Arrays are **not** pointers. They **decay** to pointers in most expressions, but they are different things.

```c
int arr[10];
int *p = arr;

sizeof(arr);   // 40 bytes — size of the whole array
sizeof(p);     // 8 bytes — size of the pointer
```

When you pass an array to a function, it decays to a pointer. The function loses the size information. This is why C APIs require you to pass the size separately.

A string literal `"hello"` is an array of `const char` in read-only memory. Modifying it is undefined behavior.

### Memory Alignment

Every type has an **alignment requirement** — the address must be a multiple of some power of 2. `int` typically requires 4-byte alignment, `double` requires 8.

Misaligned access on x86 still works (hardware handles it with a performance penalty). On ARM, it causes a bus error or hardware exception.

`alignof(type)` tells you the alignment requirement of a type at compile time. `alignas(n)` forces a variable to a specific alignment boundary.

```c
printf("%zu\n", alignof(int));     // prints 4
printf("%zu\n", alignof(double));  // prints 8

// Force a buffer to 64-byte alignment (one cache line)
alignas(64) char cache_line_buf[64];

// Useful when you need SIMD-aligned memory or when
// interfacing with hardware that requires aligned buffers
```

When you need dynamically allocated aligned memory: `aligned_alloc(alignment, size)` from `<stdlib.h>` (C11), or `posix_memalign(&ptr, alignment, size)` on POSIX systems.

### Void Pointers

`void *` is a generic pointer — it can point to anything. You cannot dereference it directly; you must cast it first. `malloc` returns `void *` for this reason.

```c
void *ptr = malloc(64);
int *ip = (int *)ptr;
*ip = 42;
```

### Dangling and Wild Pointers

**Dangling pointer:** Points to memory that has been freed or is out of scope. Accessing it is undefined behavior — the memory may be reused for something else.

```c
int *p = malloc(sizeof(int));
free(p);
*p = 5;   // dangling pointer access — UB
p = NULL; // always NULL the pointer after freeing
```

**Wild pointer:** An uninitialized pointer. Contains whatever garbage was in memory. Always initialize pointers.

```c
int *p;    // wild pointer
*p = 5;   // crash or silent corruption
```

### Aliasing

Two pointers **alias** if they point to the same or overlapping memory. The C standard's **strict aliasing rule** says the compiler can assume pointers of different types do not alias. This allows it to reorder loads and stores aggressively. Violating this assumption produces UB that is impossible to debug because the compiler silently generates wrong code.

Example of the violation:

```c
float f = 3.14f;
unsigned int *p = (unsigned int *)&f;  // UB — aliasing float through uint*
printf("%u\n", *p);  // compiler may "optimize" this to garbage
```

**Why `memcpy` is safe for type-punning:** `memcpy` copies raw bytes. The compiler knows it touches memory as raw bytes, so no aliasing rules apply. The compiler also recognizes the pattern and typically optimizes the `memcpy` away entirely, generating the same code as a direct cast — but without the UB.

```c
float f = 3.14f;
unsigned int bits;
memcpy(&bits, &f, sizeof(bits));  // defined behavior, compiler optimizes to zero cost
printf("%u\n", bits);  // safe
```

Use the union method (in C) or `memcpy` (in both C and C++) whenever you need to reinterpret bytes between types.

---

## 3. Data Structures in Pure C

Implement all of these from scratch. No libraries. No shortcuts.

### Dynamic Array

A contiguous block of memory that grows when full. `realloc` doubles the capacity when you exceed it.

Key operations: `push`, `pop`, `get`, `set`, `resize`.

**Amortized O(1) append:** A single `push` that triggers a resize is O(n) — you copy all elements. But if you double the capacity every time, the total cost of n pushes is O(n), so the average per push is O(1). This is what "amortized O(1)" means — expensive operations are rare enough that the average cost over many operations is still constant. If you grew by 1 each time instead, every push would eventually be O(n) and the total would be O(n²).

**Cache locality:** A contiguous array stores all elements in adjacent memory. When the CPU fetches one element, the hardware prefetcher loads the surrounding cache line (64 bytes) into L1 cache automatically. The next few elements are already in cache — no memory latency. A linked list scatters nodes across the heap. Every traversal step is potentially a cache miss, requiring a round trip to main memory (~100ns vs ~1ns for L1). For small elements, iterating an array can be 10–50x faster than a linked list even though both are O(n) by Big-O.

### Linked List

Nodes with a data field and a `next` pointer. Non-contiguous in memory — each node is separately allocated.

Variants: singly linked, doubly linked (add `prev` pointer), sentinel node (dummy head/tail).

**Sentinel node:** A dummy node that always exists at the head (or tail) of the list, containing no real data. The list is "empty" when the sentinel's `next` points to itself or to NULL. The benefit: your insert and delete functions never need special cases for an empty list or for modifying the head pointer. Every real node has a predecessor (at worst, the sentinel), so the code is uniform. Without a sentinel, inserting at the head is a special case requiring the caller to update their pointer to the list.

Key operations: `insert_head`, `insert_tail`, `delete`, `search`, `reverse`.
Key pitfalls: losing track of `prev` before deleting, forgetting to free all nodes.

### Circular Buffer (Ring Buffer)

A fixed-size array treated as circular. `front` and `rear` indices wrap around using modulo. No shifting needed — O(1) enqueue and dequeue.

```c
typedef struct {
    int data[CAPACITY];
    int front, rear, count;
} ring_t;
```

Differentiating full from empty: use a `count` field or leave one slot empty.

This is where your circular queue practice belongs — it is a natural fit for bounded buffers, audio streams, inter-thread communication, and undo history.

### Stack

LIFO — last in, first out. Trivially implemented on top of a dynamic array or linked list.

Key operations: `push`, `pop`, `peek`, `is_empty`.
Real-world use: function call stack, expression evaluation, backtracking (DFS), undo.

### Queue

FIFO — first in, first out. Implement with a circular buffer for O(1) enqueue/dequeue without shifting, or with a doubly linked list.

Key operations: `enqueue`, `dequeue`, `peek`, `is_empty`.

### Hash Table

Maps keys to values using a hash function. The hash function turns a key (a string, integer, etc.) into a bucket index. Multiple keys can hash to the same bucket — this is a **collision**.

**Collision resolution:**
- **Chaining:** Each bucket is a linked list. Collisions just append to the list. Simple, but pointer chasing kills cache performance on dense tables.
- **Open addressing:** All entries live in the array itself. On collision, probe for the next empty slot (linear probing = check next slot, quadratic probing = check +1, +4, +9...). Better cache behavior, but deletion is tricky (you cannot just clear a slot — it breaks probe chains).

**Load factor:** The ratio of stored entries to bucket count (n/capacity). As it grows, collisions become more frequent. Rehash (allocate a larger array, reinsert everything) when load factor exceeds ~0.7.

**Hash function quality:** A bad hash function clusters keys into few buckets, destroying O(1) performance. Two simple and effective hash functions for strings:

```c
// djb2 — simple, fast, good distribution
uint32_t djb2(const char *s) {
    uint32_t h = 5381;
    while (*s) h = h * 33 ^ (unsigned char)*s++;
    return h;
}

// FNV-1a — slightly better avalanche effect
uint32_t fnv1a(const char *s) {
    uint32_t h = 2166136261u;
    while (*s) { h ^= (unsigned char)*s++; h *= 16777619u; }
    return h;
}
```

Key operations: `insert`, `lookup`, `delete`.

A good hash table is one of the most useful data structures in practice. Get it right.

### Binary Search Tree

Each node has a left child (smaller) and right child (larger). In-order traversal gives sorted output.

Key operations: `insert`, `search`, `delete`, `inorder`, `height`.
Key pitfall: deletion is tricky (find in-order successor/predecessor for nodes with two children).

Unbalanced trees degrade to O(n) — build an AVL tree to learn self-balancing.

### AVL Tree

A self-balancing BST. After every insert/delete, check the balance factor (height difference between left and right subtrees). If it exceeds 1, rotate.

Key operations: `left_rotate`, `right_rotate`, `get_balance`, all BST operations.
Key concepts: balance factor, rotation types (LL, RR, LR, RL).

### Heap (Priority Queue)

A complete binary tree stored in an array where every parent is larger (max-heap) or smaller (min-heap) than its children.

Parent of node `i`: `(i - 1) / 2`. Left child: `2*i + 1`. Right child: `2*i + 2`.

Key operations: `insert` (add to end, sift up), `extract_max/min` (swap root with last, sift down).
Key uses: scheduling, Dijkstra's algorithm, heapsort.

### Trie

A tree for storing strings where each node represents a character. Efficient prefix search and autocomplete.

Key operations: `insert`, `search`, `starts_with`.
Key use: dictionary, autocomplete, IP routing tables.

---

## 4. Algorithms

You do not need to be a competitive programming god. You need **correctness and reasoning about cost**.

### Time & Space Complexity

Big-O describes how cost scales with input size. Know these intuitively:

| Complexity | Name | Example |
|---|---|---|
| O(1) | Constant | Array index, hash lookup |
| O(log n) | Logarithmic | Binary search, BST ops |
| O(n) | Linear | Linear scan |
| O(n log n) | Linearithmic | Merge sort, heapsort |
| O(n²) | Quadratic | Bubble sort, naive search |
| O(2ⁿ) | Exponential | Recursive subsets |

Know the difference between worst case, average case, and amortized cost. Know that O-notation hides constants — a O(n²) with tiny constants can beat O(n log n) for small n.

### Sorting

Implement these by hand:

- **Bubble sort** — O(n²), only useful for already-nearly-sorted data or as a teaching tool.
- **Insertion sort** — O(n²) worst, O(n) best. Excellent for small arrays (< 10 elements). Used as a base case inside timsort and introsort.
- **Merge sort** — O(n log n) guaranteed. Stable. Requires O(n) extra memory. The sorting algorithm of choice when stability matters.
- **Quick sort** — O(n log n) average, O(n²) worst. In-place. Fastest in practice due to cache locality. `qsort` in libc is typically an optimized quicksort variant.
- **Heap sort** — O(n log n) guaranteed. In-place. Not stable. Worse constants than quicksort.

Know `qsort` from `<stdlib.h>`:
```c
// void qsort(void *base, size_t nmemb, size_t size,
//            int (*compar)(const void *, const void *));
// Sorts nmemb elements of size bytes each starting at base.
// compar returns negative/zero/positive like strcmp.
```

### Binary Search

O(log n) search in a sorted array. Deceptively simple to implement incorrectly (off-by-one errors).

```c
// int bsearch returns pointer, or NULL if not found
// void *bsearch(const void *key, const void *base,
//               size_t nmemb, size_t size,
//               int (*compar)(const void *, const void *));
```

### Graph Traversal

**DFS (Depth-First Search):** Explore as deep as possible before backtracking. Recursive (uses call stack) or iterative (explicit stack). Used for: cycle detection, topological sort, connected components, maze solving.

**BFS (Breadth-First Search):** Explore level by level. Uses a queue. Used for: shortest path in unweighted graphs, level-order traversal, flood fill.

### Recursion

Understand the call stack implications. Every recursive call adds a stack frame. Deep recursion can overflow the stack. Know when to convert to iteration.

Tail recursion — where the recursive call is the last thing the function does — can be optimized by the compiler into a loop (`-O2`). Write tail-recursive functions when depth is a concern.

### Sliding Window

A technique for problems involving contiguous subarrays. Maintain two pointers and slide them forward, updating the window state incrementally. Turns O(n²) into O(n).

### Bit Manipulation

Know the bitwise operators: `&` (AND), `|` (OR), `^` (XOR), `~` (NOT), `<<` (left shift), `>>` (right shift).

Essential tricks:
- Check if bit `n` is set: `(x >> n) & 1`
- Set bit `n`: `x | (1 << n)`
- Clear bit `n`: `x & ~(1 << n)`
- Check if power of 2: `x && !(x & (x - 1))`
- Get lowest set bit: `x & (-x)`

Bit manipulation is heavily used in systems programming — flags, masks, protocol headers, hardware registers.

---

## 5. Linux Internals

This is where systems programming truly begins. Understanding Linux is not optional — it is the environment everything runs in.

### Processes

A process is an instance of a running program. It has its own virtual address space, file descriptor table, and process ID.

**Key concepts:**

**PID (Process ID):** Every process has a unique integer ID. `getpid()` returns the current PID. `getppid()` returns the parent's PID.

**`fork()`:** Creates a copy of the current process. Parent and child continue from the same point but in separate address spaces. Returns the child's PID in the parent, and 0 in the child.

**Copy-on-write (COW):** After `fork`, the parent and child do not immediately get physically separate copies of all memory. The kernel marks all memory pages as read-only and shared between both processes. When either process tries to write to a page, the CPU triggers a page fault. The kernel then makes a private copy of just that page for the writing process, marks it writable, and resumes. Pages that are never written are never copied — they remain shared. This makes `fork` fast even when the process has gigabytes of memory, because most of it is typically never written before `exec` replaces the whole address space anyway.

**`exec()` family:** Replaces the current process image with a new program. Does not create a new process — it transforms the current one. Combined with `fork()`, this is how shells launch programs.

**`wait()`/`waitpid()`:** The parent waits for a child to terminate and collects its exit status. If you don't call `wait`, the child becomes a **zombie** — it has exited but its entry in the process table is not cleaned up.

**Zombie:** A process that has exited but whose parent has not called `wait`. It holds a process table slot but no resources. A large number of zombies indicates a bug in the parent.

**Orphan:** A process whose parent has exited. Orphans are automatically adopted by `init` (PID 1), which eventually calls `wait` on them.

**Signals:** Asynchronous notifications sent to processes. Common signals:
- `SIGTERM` (15) — polite termination request (default for `kill`)
- `SIGKILL` (9) — forced termination, cannot be caught or ignored
- `SIGSEGV` (11) — segmentation fault (invalid memory access)
- `SIGINT` (2) — interrupt from keyboard (Ctrl+C)
- `SIGCHLD` — sent to parent when child exits
- `SIGHUP` (1) — terminal hangup or daemon reload convention

### Threads

Threads are lighter than processes — they share the same address space and file descriptors but have separate stacks and register state.

**pthreads (POSIX Threads):**

```c
// int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
//                    void *(*start_routine)(void *), void *arg);
// Creates a new thread. start_routine is the function it runs.

// int pthread_join(pthread_t thread, void **retval);
// Waits for thread to finish. Like wait() for threads.

// int pthread_mutex_lock(pthread_mutex_t *mutex);
// Acquires the mutex, blocking if already held.

// int pthread_mutex_unlock(pthread_mutex_t *mutex);
// Releases the mutex.

// int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
// Atomically releases mutex and waits for condition signal.

// int pthread_cond_signal(pthread_cond_t *cond);
// Wakes one thread waiting on the condition.
```

Under the hood, Linux implements threads using the `clone(2)` system call with flags that tell the kernel which resources to share (`CLONE_VM` for address space, `CLONE_FILES` for file descriptors, etc.).

pthreads uses `futex(2)` (Fast Userspace muTEX) for efficient mutex implementation. A futex is a single integer in shared memory. When the mutex is uncontested, locking it is just an atomic compare-and-swap on that integer — entirely in userspace, no kernel involvement, no system call overhead. Only when a thread actually needs to block (the mutex is held) does it call into the kernel to sleep. This means uncontested mutexes are nearly free, which is why using mutexes liberally is not a performance disaster in most code.

**Mutex:** Mutual exclusion lock. Only one thread can hold it at a time. Prevents race conditions on shared data. Must always unlock what you lock.

**Read-Write Lock:** Multiple readers can hold it simultaneously. Only one writer, which blocks all readers and writers. `pthread_rwlock_t`.

**Semaphore:** A counter. `sem_wait` decrements (blocks at 0), `sem_post` increments. More general than a mutex. Use `sem_t` from `<semaphore.h>`.

**Condition Variable:** Allows a thread to sleep until a specific condition is true. Always used with a mutex. The pattern is: lock mutex → check condition → if false, `pthread_cond_wait` (atomically releases mutex and sleeps) → when signaled, re-acquire mutex → re-check condition.

**Deadlock:** Two or more threads each hold a lock the other needs. They block forever. Prevent with lock ordering (always acquire locks in the same global order).

**Starvation:** A thread perpetually denied the resources it needs because other threads keep preempting it.

### Memory

**Virtual Memory:** Each process sees a private 64-bit address space. The kernel maps virtual pages to physical pages. Processes are isolated — one process cannot access another's memory directly.

**Paging:** Virtual memory is divided into fixed-size pages (typically 4 KB on x86). The page table maps virtual page numbers to physical frame numbers. The CPU's MMU (Memory Management Unit) performs the translation.

**TLB (Translation Lookaside Buffer):** A cache for page table entries. TLB miss → walk the page table → expensive. TLB thrashing (using more pages than the TLB can cache) crushes performance.

**Page Fault:** When a process accesses a virtual page that is not mapped or not in physical memory. The kernel handles it — allocating a page, loading from disk (swap), or sending `SIGSEGV` if access is illegal.

**`mmap()`:** Maps a file or anonymous memory into the process's address space. Used for large allocations, shared memory between processes, and memory-mapped I/O.

**Heap Allocator:** `malloc` manages the heap by requesting pages from the OS and subdividing them for individual allocations.

**`brk` and `sbrk`:** The heap is a region of memory that starts right after the BSS segment. `brk(addr)` sets the end of the heap to `addr`. `sbrk(increment)` moves it forward by `increment` bytes and returns the old end — effectively allocating a chunk of new memory. `malloc` used to call `sbrk` for small allocations, but modern allocators use `mmap` for large ones and maintain a pool for small ones.

**Free list:** When you `free` a block, `malloc` does not return the memory to the OS. It adds the block to an internal free list. The next `malloc` that fits checks the free list first.

**Allocation strategies:**
- **First-fit:** Scan the free list from the beginning, return the first block that is large enough. Fast but can fragment the beginning of the heap.
- **Best-fit:** Scan the entire free list and return the smallest block that fits. Minimizes wasted space but slower and can leave many tiny unusable fragments.
- **Bin-based (used by glibc's ptmalloc):** Maintain separate free lists ("bins") for different size ranges. Small allocations find their bin directly — near O(1). Most production allocators use this approach.

**Fragmentation:**
- **Internal:** Allocator gives you more memory than you asked for (rounding up to alignment or minimum block size). Space is wasted inside allocations.
- **External:** Many small free blocks exist but no single one is large enough for a new allocation. The heap looks like Swiss cheese.

### Filesystems

**Inode:** Every file has an inode — a data structure containing metadata: owner, permissions, timestamps, file size, and pointers to data blocks. The filename lives in the directory, not the inode. This is why hard links work — two directory entries, one inode.

**File Descriptor:** An integer that represents an open file in a process. `open()` returns one. 0 = stdin, 1 = stdout, 2 = stderr. File descriptors are inherited across `fork()`.

**Pipes:** Unidirectional byte stream between processes. `pipe(int fd[2])` — `fd[0]` is read end, `fd[1]` is write end. Shells use pipes to connect programs.

**Sockets:** Bidirectional communication endpoints. Can be local (Unix domain sockets) or networked (TCP/UDP).

**Symbolic Links:** A file containing a path to another file. Like a shortcut. Broken if the target is deleted.

**Permissions:** Three categories: owner, group, others. Three bits each: read (4), write (2), execute (1). `chmod 755` = rwxr-xr-x.

**`umask`:** A mask that is subtracted from the permissions you specify when creating a file. If your process has `umask 022`, and you create a file with permissions `0666` (rw-rw-rw-), the actual result is `0666 & ~022 = 0644` (rw-r--r--). The umask strips the write bits for group and others. It is a process-level default that prevents newly created files from being too permissive. Check yours with `umask` in the shell; change it with `umask(mode_t mask)` in C.

### Scheduling

**Context Switch:** The kernel saves the CPU state (registers, PC) of one process and restores another's. Has overhead — this is why too many threads or too frequent I/O can hurt performance.

**Kernel Mode vs User Mode:** Processors have privilege levels. User programs run in user mode — restricted. System calls switch to kernel mode where the OS code runs with full hardware access. The transition has a cost.

**Preemption:** The kernel can forcibly interrupt a running process to give CPU time to another. Prevents one process from monopolizing the CPU.

**Scheduling Policies:**
- `SCHED_OTHER` — default CFS (Completely Fair Scheduler). Fair time-sharing.
- `SCHED_FIFO` — real-time, first in first out. Higher priority than normal. Runs until it blocks or yields.
- `SCHED_RR` — real-time, round-robin with time slices.

---

## 6. System Calls & POSIX APIs

**POSIX** (Portable Operating System Interface) is a family of standards that define a common API for Unix-like operating systems — Linux, macOS, BSDs. If you write to the POSIX API instead of Linux-specific syscalls, your code compiles and runs on all of them. Most of the functions in this section are POSIX. Linux extends POSIX with its own extras like `epoll`, which is Linux-only.

When a system call fails, it returns -1 and sets the global variable `errno` to an error code. Always check return values. `perror("context")` prints a human-readable error message including the `errno` string. `strerror(errno)` gives you the string directly.

**`EAGAIN` / `EWOULDBLOCK`:** When a file descriptor is set to non-blocking mode (via `O_NONBLOCK`) and an operation cannot complete immediately — no data available to read, or the write buffer is full — the call returns -1 with `errno == EAGAIN` instead of blocking. This is not an error; it means "try again later." Your event loop checks `EAGAIN` and moves on to handle other fds, then comes back when `epoll` signals the fd is ready again.

These are the actual functions you call. Know what they do, what they return, and when they fail.

### File I/O

```c
// int open(const char *pathname, int flags, mode_t mode);
// Opens or creates a file. Returns fd or -1 on error.
// flags: O_RDONLY, O_WRONLY, O_RDWR, O_CREAT, O_TRUNC, O_APPEND, O_NONBLOCK

// ssize_t read(int fd, void *buf, size_t count);
// Reads up to count bytes into buf. Returns bytes read, 0 on EOF, -1 on error.
// May return fewer bytes than requested — always loop on reads.

// ssize_t write(int fd, const void *buf, size_t count);
// Writes count bytes from buf. Returns bytes written or -1 on error.
// May return fewer bytes — always loop on writes.

// int close(int fd);
// Closes file descriptor. Always check return value — close can fail (e.g. on NFS).

// off_t lseek(int fd, off_t offset, int whence);
// Repositions file offset. whence: SEEK_SET, SEEK_CUR, SEEK_END.
// Returns new offset or -1. Use lseek(fd, 0, SEEK_END) to get file size.

// int stat(const char *pathname, struct stat *statbuf);
// Gets file metadata (size, permissions, inode, timestamps) into statbuf.

// int fcntl(int fd, int cmd, ...);
// File control: get/set flags, duplicate fd, set non-blocking mode, advisory locks.
// F_GETFL/F_SETFL for flags, F_DUPFD to duplicate.
```

### Process APIs

```c
// pid_t fork(void);
// Creates child process. Returns child PID in parent, 0 in child, -1 on error.

// int execve(const char *pathname, char *const argv[], char *const envp[]);
// Replaces current process with new program. Does not return on success.
// execl, execlp, execvp — convenience wrappers around execve.

// pid_t waitpid(pid_t pid, int *wstatus, int options);
// Waits for child process. pid=-1 waits for any child.
// WIFEXITED(wstatus) — child exited normally.
// WEXITSTATUS(wstatus) — exit code.
// WIFSIGNALED(wstatus) — child killed by signal.

// int kill(pid_t pid, int sig);
// Sends signal sig to process pid.

// int pipe(int pipefd[2]);
// Creates pipe. pipefd[0] = read end, pipefd[1] = write end.

// int dup2(int oldfd, int newfd);
// Makes newfd a copy of oldfd. Standard way to redirect stdout/stdin.
// dup2(fd, STDOUT_FILENO) redirects stdout to fd.
```

### Memory APIs

```c
// void *malloc(size_t size);
// Allocates size bytes. Returns pointer or NULL on failure.
// Contents are uninitialized (may contain garbage).

// void *calloc(size_t nmemb, size_t size);
// Allocates nmemb * size bytes, zero-initialized. Safer than malloc for arrays.

// void *realloc(void *ptr, size_t size);
// Resizes allocation. May move to new address. If NULL, acts like malloc.
// Old pointer is invalid after successful realloc — use returned pointer.

// void free(void *ptr);
// Releases allocation. Never free twice. Never free stack memory.

// void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
// Maps file or anonymous memory into address space.
// prot: PROT_READ, PROT_WRITE, PROT_EXEC
// flags: MAP_PRIVATE (copy-on-write), MAP_SHARED (shared with other processes), MAP_ANONYMOUS
// Returns pointer or MAP_FAILED.

// int munmap(void *addr, size_t length);
// Unmaps previously mapped region.
```

### Thread APIs

```c
// int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
//                    void *(*start_routine)(void *), void *arg);
// Creates thread running start_routine(arg).

// int pthread_join(pthread_t thread, void **retval);
// Waits for thread to finish. Collects return value.

// int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr);
// Initializes mutex. Or use PTHREAD_MUTEX_INITIALIZER for static init.

// int pthread_mutex_lock(pthread_mutex_t *mutex);
// Acquires mutex. Blocks if already held.

// int pthread_mutex_trylock(pthread_mutex_t *mutex);
// Non-blocking: returns EBUSY if already held.

// int pthread_mutex_unlock(pthread_mutex_t *mutex);
// Releases mutex.

// int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
// Atomically releases mutex and waits for signal on cond.
// Always call in a loop: while (!condition) pthread_cond_wait(&cond, &mutex);

// int pthread_cond_signal(pthread_cond_t *cond);
// Wakes one waiting thread.

// int pthread_cond_broadcast(pthread_cond_t *cond);
// Wakes all waiting threads.
```

### Socket APIs

```c
// int socket(int domain, int type, int protocol);
// Creates socket. domain: AF_INET (IPv4), AF_UNIX (local).
// type: SOCK_STREAM (TCP), SOCK_DGRAM (UDP).

// int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
// Assigns address/port to socket (server side).

// int listen(int sockfd, int backlog);
// Marks socket as passive (server). backlog = pending connection queue size.

// int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
// Accepts incoming connection. Returns new fd for that connection.

// int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
// Initiates connection (client side).

// ssize_t send(int sockfd, const void *buf, size_t len, int flags);
// ssize_t recv(int sockfd, void *buf, size_t len, int flags);
// Send/receive data. Like write/read but with socket-specific flags.

// int epoll_create1(int flags);
// Creates epoll instance. Returns fd.

// int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// Add/modify/remove fd from epoll interest list.
// op: EPOLL_CTL_ADD, EPOLL_CTL_MOD, EPOLL_CTL_DEL

// int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
// Waits for events. Returns number of ready fds.
```

### Signals

```c
// int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
// Registers signal handler. Prefer over signal() — more portable and safer.

// typedef void (*sighandler_t)(int);
// Signal handler signature. int argument is the signal number.

// unsigned int alarm(unsigned int seconds);
// Schedules SIGALRM after seconds. Returns remaining time from previous alarm.

// int pause(void);
// Suspends process until any signal is received.
```

---

## 7. Networking

Networking vastly increases your employability. Every server, every daemon, every microservice uses it.

### TCP Fundamentals

**Three-Way Handshake:**
1. Client sends `SYN` — "I want to connect"
2. Server responds `SYN-ACK` — "I accept, here is my sequence number"
3. Client sends `ACK` — "Acknowledged"

Connection established. Each side maintains sequence numbers for reliability.

**Why TCP is reliable:** Every segment is acknowledged. Missing segments are retransmitted. Segments are reordered if they arrive out of order.

**Blocking vs Non-Blocking I/O:**
- **Blocking:** `read` waits until data is available. The thread is suspended.
- **Non-blocking:** Set with `fcntl(fd, F_SETFL, O_NONBLOCK)`. `read` returns immediately with `EAGAIN` if no data. You must poll.

**I/O Multiplexing:** Handling many fds in one thread.
- `select` — oldest, limited to 1024 fds.
- `poll` — no fd limit, slightly better API.
- `epoll` — Linux-specific, O(1) for large fd sets. The right choice for servers.

**epoll model:**
```
epoll_create1() → get epfd
epoll_ctl(epfd, EPOLL_CTL_ADD, clientfd, &event) → register interest
epoll_wait(epfd, events, MAX, -1) → block until events ready
for each ready event: read/write the fd
```

### Projects to Build

- **TCP echo server:** Accept connections, echo back what you receive. Teaches accept loop and basic I/O.
- **Multithreaded server:** One thread per connection. Teaches pthreads + networking together.
- **epoll event loop:** Single-threaded, handles hundreds of connections. Teaches event-driven programming.
- **HTTP/1.0 server:** Parse GET requests, serve files. Teaches string parsing, headers, and the HTTP request/response model.
- **Chat server:** Multiple clients broadcasting to each other. Teaches shared state under concurrency.

---

## 8. Tooling

You are unemployable without these. They are not optional.

### Compiler Flags You Must Know

```bash
gcc -Wall -Wextra -Werror -g -fsanitize=address,undefined file.c -o program
```

- `-Wall -Wextra` — enable most useful warnings. Always use.
- `-Werror` — treat warnings as errors. Forces you to fix them.
- `-g` — include debug symbols (for gdb).
- `-O2` — optimization level 2. Use for release.
- `-fsanitize=address` — AddressSanitizer (detects memory bugs).
- `-fsanitize=undefined` — UBSanitizer (catches undefined behavior).
- `-fsanitize=thread` — ThreadSanitizer (detects data races).

### GDB — GNU Debugger

```bash
gdb ./program          # start
run arg1 arg2          # run with args
break main             # set breakpoint at function
break file.c:42        # set breakpoint at line
next                   # execute next line (don't enter function)
step                   # step into function
continue               # run until next breakpoint
print variable         # print variable value
x/10x pointer          # examine 10 hex words at address
backtrace              # show call stack
info registers         # show register values
watch variable         # break when variable changes
```

When a program crashes, run it under gdb and type `backtrace` to see where it died and why.

### Valgrind

Runs your program in a virtual machine that tracks every memory access.

```bash
valgrind --leak-check=full --show-leak-kinds=all ./program
```

Reports: invalid reads/writes, use of uninitialized memory, memory leaks (lost bytes at exit), double frees.

Slows program by ~20x. Use only for debugging.

### strace / ltrace

```bash
strace ./program         # trace system calls
ltrace ./program         # trace library calls
strace -p PID            # attach to running process
strace -e trace=read,write ./program  # filter by syscall
```

Invaluable for understanding what a program is actually doing — what files it opens, what syscalls it makes.

### perf

```bash
perf stat ./program      # CPU event counts (cycles, cache misses, etc.)
perf record ./program    # record profiling data
perf report              # interactive hotspot view
```

Shows you where your program is spending time at the function level.

### Binary Analysis

```bash
nm program               # list symbols (functions, globals) in object file
objdump -d program       # disassemble — see the machine code
readelf -h program       # ELF header info
readelf -S program       # section headers
ldd program              # list shared library dependencies
file program             # identify file type
```

---

## 9. Build Systems

Juniors who ignore this get destroyed when working on real codebases.

### Makefiles

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -g
TARGET = program
SRCS = main.c utils.c data.c
OBJS = $(SRCS:.c=.o)

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)

.PHONY: clean
```

Make only recompiles what changed. It tracks dependencies through the modification timestamps of files.

`$@` — target name. `$<` — first prerequisite. `$^` — all prerequisites.

### Static Libraries

A `.a` file is an archive of `.o` files. Linked at compile time, code is copied into the binary.

```bash
ar rcs libutils.a utils.o data.o    # create static library
gcc main.c -L. -lutils -o program   # link against it
```

### Shared Libraries

A `.so` file loaded at runtime. Multiple programs share one copy in memory.

```bash
gcc -shared -fPIC -o libutils.so utils.c  # create shared library
gcc main.c -L. -lutils -o program          # link against it
export LD_LIBRARY_PATH=.                   # tell loader where to find it
```

`-fPIC` — **Position Independent Code.** Here is what this means: normal executable code contains hardcoded addresses. Instructions like `call 0x401234` (call the function at this exact address) work fine in an executable because the OS loads it at a fixed address. But a shared library can be loaded at any address in any process's address space — the address differs between processes. If the library contained hardcoded addresses, they would all be wrong.

PIC solves this by making all address references relative — "call the function that is 240 bytes ahead of the current instruction" instead of "call 0x401234." The CPU can execute this correctly regardless of where the library was loaded. The cost is slightly larger code and one extra indirection through the **GOT (Global Offset Table)** for global variables. On x86-64, PIC is essentially free because the architecture supports relative addressing natively.

### Symbol Resolution

When the linker combines object files, it resolves symbols (function and global variable names) across files. `extern` declarations tell a file "this symbol is defined elsewhere." If a symbol is defined in multiple places, you get a linker error.

`static` on a global limits its visibility to the translation unit — it won't conflict with same-named statics in other files.

---

## 10. Memory Debugging

This is what separates careful systems programmers from careless ones.

### Bug Types

**Use-after-free:** Accessing memory after freeing it. The allocator may have reused that memory for something else. Produces subtle, non-deterministic bugs.

**Double free:** Calling `free` twice on the same pointer. Corrupts allocator internal state. Set pointer to NULL after freeing.

**Stack overflow:** Local variable allocation or deep recursion exhausts stack space. Results in SIGSEGV.

**Heap corruption:** Writing past the end of an allocation, or writing to freed memory. Corrupts allocator bookkeeping. Often manifests as a crash deep inside `malloc` or `free`, far from the actual bug.

**Buffer overflow:** Writing past the end of an array — on the stack or heap. On the stack, this can overwrite the return address, leading to arbitrary code execution (classic security vulnerability).

**Memory leak:** Allocating memory and losing all references to it without freeing. The program's memory usage grows until it is killed. Use Valgrind to detect.

**Race condition:** Two threads access shared data without synchronization, and the result depends on timing. Produces bugs that are nearly impossible to reproduce under a debugger.

### Tools

**AddressSanitizer (ASan):** Compile with `-fsanitize=address`. Detects use-after-free, buffer overflows, use-after-return, double-free. Slows program by ~2x. Use during development.

**ThreadSanitizer (TSan):** Compile with `-fsanitize=thread`. Detects data races. Cannot be used with ASan simultaneously.

**Valgrind:** Slower (20x) but catches more obscure bugs. Detects uninitialized memory reads.

**Electric Fence:** Old but simple library that places guard pages around allocations, making heap overflows crash immediately at the bug site.

---

## 11. Concurrency

The adult zone of systems programming. This is where your understanding of hardware matters.

### Synchronization

The fundamental problem: multiple threads writing shared state without coordination produce **data races** — results that depend on the interleaving of instructions, which is non-deterministic.

The solution is **mutual exclusion**: ensure only one thread can access shared state at a time.

### Atomic Operations

Operations that execute as a single, indivisible unit. The CPU guarantees no other thread sees an intermediate state.

```c
#include <stdatomic.h>

atomic_int counter = 0;
atomic_fetch_add(&counter, 1);   // atomic increment
atomic_load(&counter);           // atomic read
atomic_store(&counter, 0);       // atomic write
```

Atomic operations are lock-free — no mutex needed. But they are only useful for single variables. For complex invariants involving multiple variables, use mutexes.

### Lock-Free Concepts

Lock-free algorithms allow progress even if some threads are stalled. They use atomic **compare-and-swap (CAS)** — `atomic_compare_exchange` in C11. CAS does this atomically: "if the value at this address is still X, replace it with Y, otherwise do nothing and tell me it changed."

The pattern:
```c
int old = atomic_load(&shared);
int new_val = compute(old);
// If shared is still `old`, swap in new_val. If not, some other thread changed it —
// retry the whole thing with the fresh value.
while (!atomic_compare_exchange_weak(&shared, &old, new_val)) {
    new_val = compute(old);  // old was updated by CAS to the current value
}
```

This is the foundation of all lock-free data structures. At junior level: understand the concept, know `stdatomic.h` exists, understand why it is hard. You do not need to implement lock-free data structures yet.

### Cache Coherence & False Sharing

Modern CPUs have multiple cores with separate L1/L2 caches. When two cores write to different variables that share a cache line (64 bytes on x86), they invalidate each other's cache line constantly — **false sharing**. This kills performance even though they are not touching the same variable.

Fix: pad shared variables to fill a cache line, or use `__attribute__((aligned(64)))`.

### Memory Ordering

Compilers and CPUs reorder instructions for performance. This can break concurrent code. Memory barriers and acquire/release semantics in `stdatomic.h` provide ordering guarantees.

At junior level: know this exists and that `atomic_thread_fence` is the tool. Deep understanding comes with experience.

---

## 12. Embedded & RTOS (Optional)

Extremely valuable in India — automotive, IoT, telecom, and hardware companies all need this.

### Hardware Interfaces

**UART:** Serial communication. Two wires (TX, RX). Used for debugging output and simple device communication. Speed is "baud rate" (bits/second).

**SPI:** 4-wire synchronous serial. Master/slave. Faster than UART. Used for flash memory, displays, sensors.

**I2C:** 2-wire bus. Multiple devices on same bus, each with an address. Slower than SPI, simpler wiring.

**GPIO (General Purpose I/O):** Digital pins you can set high/low (output) or read (input). Fundamental to embedded control.

**Interrupts:** Hardware events that stop the CPU and jump to an interrupt handler. Much more efficient than polling. The handler must be fast — don't do heavy work in an ISR (Interrupt Service Routine).

### RTOS Concepts

An RTOS (Real-Time Operating System) provides deterministic scheduling. Tasks must meet deadlines.

Key concepts:
- **Tasks/threads** — independently scheduled units of execution
- **Scheduler** — typically preemptive priority-based
- **Semaphores/mutexes** — same concept as pthreads but RTOS-specific APIs
- **Message queues** — inter-task communication
- **Tick timer** — the heartbeat of the RTOS, drives scheduling

FreeRTOS is the dominant open-source RTOS. Study its API.

### Cross-Compilation

Compiling on one architecture (your x86 PC) for another (ARM microcontroller).

```bash
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb file.c -o program.elf
```

Toolchain: `arm-none-eabi-gcc`, `arm-none-eabi-objcopy`, `arm-none-eabi-gdb`.

---

## 13. Projects

A calculator app proves nothing. You need projects that demonstrate real understanding.

### Tier 1 — Fundamentals (Build These First)

**Custom Memory Allocator:**
Implement `my_malloc` and `my_free`. Pre-allocate a large buffer, manage a free list with first-fit or best-fit strategy. Handle alignment, coalescing adjacent free blocks, and internal/external fragmentation. This is a classic interview project that signals deep pointer and memory understanding.

**Mini Shell:**
Implement `fork`, `exec`, pipes (`|`), redirection (`>`, `<`, `>>`), background jobs (`&`), and signal handling (Ctrl+C, Ctrl+Z). You will learn every process API hands-on. Historically the gold standard of systems coursework.

**Custom Data Structure Library:**
Implement a complete library: dynamic array, hash table, linked list, heap, trie. Write a clean C API with consistent conventions. Practice const-correctness, error handling, and documentation.

### Tier 2 — Strong (Pick One)

**Multithreaded HTTP/1.0 Server:**
Listen on a TCP port, parse GET requests, serve files from a directory. Add a thread pool (fixed number of threads, task queue), proper header parsing, and graceful shutdown on SIGTERM. This is directly hireable work.

**epoll Event Loop:**
Single-threaded server handling hundreds of connections without blocking. Implement an event loop around `epoll_wait`. Handle partial reads/writes correctly. This teaches event-driven architecture deeply.

### Tier 1 — Elite (If You Want to Stand Out)

**Mini Container Runtime:**
This is the foundation of Docker. To understand it you need to understand two Linux kernel features:

**Linux Namespaces:** A namespace wraps a global resource and makes a process think it is the only process using it. The key namespaces:
- `CLONE_NEWPID` — the process gets a new PID namespace. Inside, it sees itself as PID 1. It cannot see processes outside.
- `CLONE_NEWNET` — the process gets its own network stack — its own interfaces, routing tables, and ports. The container's port 80 is not the host's port 80.
- `CLONE_NEWNS` — the process gets its own mount namespace. You can mount a different filesystem root and the process sees it as `/`. This is how container filesystems work.
- `CLONE_NEWUTS` — separate hostname. The container can have its own hostname.

You create a namespaced process by passing these flags to `clone(2)` (the lower-level version of `fork`).

**cgroups (Control Groups):** A kernel mechanism for limiting and accounting for resources used by groups of processes. To set a memory limit on a container, you write to files under `/sys/fs/cgroup/memory/your-group/memory.limit_in_bytes`. That is literally it — the kernel reads those files and enforces the limits. cgroups also handle CPU quotas, I/O bandwidth, and network priority.

Build: create a namespaced process with `clone`, `chroot` it into a minimal filesystem (BusyBox works), set cgroup resource limits, implement basic process management. This project alone can outperform 100 tutorial portfolios on a resume.

---

## 14. Interview Preparation

### Common Theory Questions

**"Why does `sizeof(arr)` differ from `sizeof(ptr)` when `int *ptr = arr`?"**
An array name in an expression decays to a pointer to its first element. `sizeof` is special — it evaluates at compile time and sees the actual array type, giving the total size. `sizeof(ptr)` sees only a pointer (8 bytes on 64-bit). Once the array decays, size information is gone.

**"How does `malloc` work?"**
It maintains a free list of heap chunks. For small allocations, it finds a free chunk (first-fit, best-fit, or bin-based). If none, it requests more memory from the OS via `sbrk` or `mmap`. The chunk header stores its size. `free` returns the chunk to the free list and coalesces adjacent free chunks.

**"What happens when you call `fork`?"**
The kernel creates a copy of the calling process — same code, same stack, same file descriptors, same heap contents. Copy-on-write means physical pages are shared until written. Both processes continue from the line after `fork`. The parent gets the child's PID, the child gets 0.

**"Difference between process and thread?"**
Processes have separate virtual address spaces, file descriptor tables, and PIDs. They communicate via pipes, sockets, or shared memory. Threads share address space and file descriptors within a process. Context switching between threads is cheaper. Bugs in one thread can corrupt the entire process's memory.

**"What is a segfault?"**
The CPU's MMU detected a memory access violation: the address is not mapped, the access violates permissions (writing to read-only), or it is unaligned (on strict architectures). The kernel sends `SIGSEGV`. In gdb, `backtrace` at the crash point shows where it happened.

**"How do you investigate a segfault?"**
Run under gdb. `run` the program. At the crash, `backtrace` shows the call stack. `frame N` selects a frame. `print variable` inspects values. Look for NULL pointers being dereferenced, array accesses out of bounds, or use-after-free.

### Coding Interview Focus

Systems interviews care about:
- Correctness first — does it work? Does it handle edge cases?
- Memory behavior — are there leaks, out-of-bounds accesses?
- Complexity awareness — can you explain the Big-O of what you wrote?
- Pointer safety — do you null-check, bounds-check?

Practice implementing: linked list with all operations, hash table, binary search, BFS/DFS on a graph, string parsing in C.

**Why not `strtok`:** `strtok` stores state in a hidden static internal buffer — the pointer to "where parsing left off." This means if two threads call `strtok` on different strings simultaneously, they corrupt each other's state. It also means you cannot parse two strings interleaved (e.g. parse string A, then mid-loop parse string B, then resume A — impossible with `strtok`). Use `strtok_r` instead, which takes an explicit `char **saveptr` that you own — no hidden state, thread-safe, re-entrant.

---

## 15. Books & Resources

### Essential Books (In Order)

1. **"The C Programming Language" — Kernighan & Ritchie (K&R)**
   The original. Short, dense, essential. Read every exercise.

2. **"Computer Systems: A Programmer's Perspective" — Bryant & O'Hallaron (CS:APP)**
   The single best book for understanding what your C code actually does at the machine level. Memory, caches, linking, concurrency. Required reading.

3. **"The Linux Programming Interface" — Michael Kerrisk**
   The definitive Linux API reference. Covers every syscall in depth. Use it alongside `man` pages.

4. **"Operating Systems: Three Easy Pieces" — Arpaci-Dusseau**
   Free online at ostep.org. The clearest explanation of OS concepts — processes, memory, filesystems, concurrency. Read it.

5. **"Advanced Programming in the UNIX Environment" — W. Richard Stevens**
   Deep POSIX and UNIX API coverage. The older sibling of Kerrisk's book.

### Online

- **`man` pages** — `man 2 open`, `man 3 malloc`, `man 7 epoll`. The authoritative reference, always available.
- **kernelnewbies.org** — Linux kernel concepts explained accessibly.
- **OSDev Wiki** — For hobby OS development.
- **xv6** — MIT's minimal teaching OS, written in clean C. Read the source.
- **CS:APP Labs** — Buffer lab, bomb lab, malloc lab. Do them all.

---

## 16. Expected Salary (India)

| Level | Profile | Range |
|---|---|---|
| Weak Junior | Basic C syntax only, no Linux knowledge | ₹3–5 LPA |
| Good Junior | Linux internals, sockets, threading, debugging | ₹6–10 LPA |
| Strong Junior | Solid projects + real fundamentals | ₹10–18 LPA |
| Exceptional Junior | Deep systems + elite projects (container runtime, allocator) | ₹18–30+ LPA |

Companies at the top end: Qualcomm, NVIDIA, Intel, Cisco, Texas Instruments, Arm India.

---

## The Honest Truth

Systems programming punishes inconsistency brutally. You cannot "kind of understand" pointers. You cannot "sort of know" what a race condition is. The machine exposes confusion immediately and silently.

The filter is not intelligence — it is willingness to sit with confusion until it becomes clarity. Read the man page. Run the code. Read the disassembly. Use gdb. Most people give up before that point. That is your advantage.

You are already using Linux daily. You already debug iteratively. You already care about what your tools actually do. That puts you ahead of the majority of candidates before you write a single line of systems code.

Build things. Break them. Understand why they broke. That is the entire job.
