---
name: stc-containers
description: Expert guidance for the STC (Standard Template Containers) library in C — stack, vec, deque, list, hmap, hset, smap, sset, cstr, and more. Use when working with STC macro-based generic containers, inplace (fixed-capacity) variants, key trait declarations (c_class_key, c_pro_key, c_compare_key), and the full lifecycle (init, push, pop, clone, drop).
metadata:
  origin: community
---

# STC Containers

Patterns and reference for the **[STC](https://github.com/stclib/STC)** library —
a modern, header-only, macro-based generic container library for C99/C11.
STC provides type-safe containers without `void*` casts, using `#define`-based
template instantiation similar to C++ templates.

## When to Use

- Working with STC containers (`stack`, `vec`, `deque`, `list`, `hmap`, `hset`, `smap`, `sset`, `cstr`, etc.).
- Declaring a new container type with `#define T` or `#define i_key`.
- Choosing the right key trait (`c_class_key`, `c_pro_key`, `c_compare_key`).
- Using inplace (fixed-capacity, stack-allocated) container variants.
- Managing element ownership — cloning, dropping, and moving values safely.
- Implementing custom comparators, equality functions, or raw key conversions.
- Iterating over containers with `c_foreach`.

## Core Concept: Template Instantiation

STC containers are instantiated via `#define` macros before `#include`.
Each `#include` creates a **new concrete type**. Re-including with different
defines creates a different type — never mix instances.

```c
// Creates type `stack_int` with push/pop/top/drop etc.
#define T stack_int, int
#include <stc/stack.h>

// Creates type `vec_float`
#define T vec_float, float
#include <stc/vec.h>
```

The shorthand `#define T <ct>, <kt>` sets both the container type name (`ct`)
and the element/key type (`kt`). Without it, the name defaults to
`stack_<kt>`, `vec_<kt>`, etc.

## Key Traits

Traits are specified as the third argument to `T` (or via individual `#define`s).

| Trait | Purpose |
|-------|---------|
| `c_compare_key` | `<kt>` is a comparable typedef. Binds `<kt>_cmp()` and `<kt>_eq()`. Auto-enables `c_use_compare`. |
| `c_class_key` | Like `c_compare_key` but also binds `<kt>_clone()` and `<kt>_drop()`. Use for STC containers as elements. |
| `c_pro_key` | "Pro" key — supports raw input type conversion. Use for `cstr`, `zsview`, `rc`, `arc`, `box`. |
| `c_use_eq` | Enable `<kt>_eq()` for `_contains()` and `_find()`. Uses `==` unless a compare trait was given. |
| `c_use_cmp` | Enable `<kt>_cmp()` for `_sort()`. Uses `<`/`>` unless a compare trait was given. |
| `c_use_compare` | Shorthand for `(c_use_cmp \| c_use_eq)`. |

Combine traits: `(c_class_key | c_use_cmp)`.

## Stack (`stc/stack.h`)

A LIFO container. Backed by a dynamic array. Pushes and pops from the back (top).

### Declaration

```c
#define T IStack, int          // type name + element type
#include <stc/stack.h>
```

With traits and custom functions:

```c
#define T MyStack, MyStruct, c_class_key   // enables clone + drop
#include <stc/stack.h>
```

### Key Methods

```c
IStack stk = {0};                  // zero-init is valid
IStack_push(&stk, 42);             // push value
int top = *IStack_top(&stk);       // peek top (const)
int *top_mut = IStack_top_mut(&stk); // peek top (mutable)
IStack_pop(&stk);                  // destroy top element
int val = IStack_pull(&stk);       // move out top element (no destroy)
isize_t n = IStack_size(&stk);
bool empty = IStack_is_empty(&stk);
IStack_drop(&stk);                 // destructor — always call when done
```

### Inplace (Fixed-Capacity) Stack

Use inplace when capacity is known at compile time and heap allocation is
undesirable (embedded, real-time, stack-allocation):

```c
#define T IStack, int, 0, 64   // 0 = no extra opt, 64 = inplace capacity
#include <stc/stack.h>

IStack stk = {0};              // lives entirely on the C stack — no malloc
IStack_push(&stk, 1);
```

### Full Example

```c
#define T IStack, int
#include <stc/stack.h>
#include <stdio.h>

int main(void) {
    IStack stk = {0};

    for (int i = 0; i < 100; ++i)
        IStack_push(&stk, i * i);

    for (int i = 0; i < 90; ++i)
        IStack_pop(&stk);

    printf("top: %d\n", *IStack_top(&stk)); // 81

    IStack_drop(&stk);
}
```

## Vec (`stc/vec.h`)

A dynamic array (like `std::vector`). Random access, push/pop at back.

```c
#define T Vec, int
#include <stc/vec.h>

Vec v = {0};
Vec_push(&v, 10);
Vec_push(&v, 20);
printf("%d\n", *Vec_at(&v, 0)); // 10
Vec_drop(&v);
```

## Deque (`stc/deque.h`)

Double-ended queue — push/pop at both front and back.

```c
#define T Deq, float
#include <stc/deque.h>

Deq d = {0};
Deq_push_back(&d, 1.0f);
Deq_push_front(&d, 0.0f);
Deq_drop(&d);
```

## List (`stc/list.h`)

Doubly-linked list. O(1) insert/remove, no random access.

```c
#define T List, int
#include <stc/list.h>

List lst = {0};
List_push_back(&lst, 1);
List_push_front(&lst, 0);
c_foreach (it, List, lst) printf("%d ", *it.ref);
List_drop(&lst);
```

## Hash Map (`stc/hmap.h`)

Unordered key-value map (open-addressing hash table).

```c
#define T Map, int, int   // Map<int,int>
#include <stc/hmap.h>

Map m = {0};
Map_insert(&m, 1, 100);
Map_insert(&m, 2, 200);

Map_iter it = Map_find(&m, 1);
if (it.ref) printf("%d\n", it.ref->second); // 100

Map_drop(&m);
```

## String Map (`stc/smap.h`)

Ordered key-value map (red-black tree). Use when ordered iteration matters.

```c
#define T SMap, cstr, int, c_pro_key    // string keys via cstr
#include <stc/smap.h>
```

## cstr — Owned String

`cstr` is STC's owned string type. Use `c_pro_key` when using `cstr` as a key
to enable raw `const char*` input for lookups and emplace operations.

```c
#include <stc/cstr.h>

cstr s = cstr_from("hello");
cstr_push_back(&s, '!');
printf("%s\n", cstr_str(&s)); // hello!
cstr_drop(&s);
```

## Iteration

All STC containers support `c_foreach`:

```c
#define T Vec, int
#include <stc/vec.h>

Vec v = {0};
for (int i = 0; i < 5; ++i) Vec_push(&v, i);

c_foreach (it, Vec, v)
    printf("%d ", *it.ref);

Vec_drop(&v);
```

For maps, `it.ref` gives a pointer to `{first, second}`:

```c
c_foreach (it, Map, m)
    printf("%d -> %d\n", it.ref->first, it.ref->second);
```

## Ownership and Lifecycle

| Situation | Pattern |
|-----------|---------|
| Element owns heap memory | Define `i_keydrop` + `i_keyclone` (or use `c_class_key`) |
| Nested STC containers as elements | Use `c_class_key` — binds `<kt>_clone()` and `<kt>_drop()` |
| Move a value out without destroying | Use `_pull()` instead of `_pop()` |
| Transfer ownership to another container | Use `_take()` — takes an unowned container |
| Cheap copy of the whole container | Use `_clone()` — deep copies all elements |

**Always call `_drop()` on every container you own.** Zero-initialized containers
(`{0}`) are safe to `_drop()` without pushing anything.

## Custom Comparator

```c
typedef struct { int x, y; } Point;

static int Point_cmp(const Point *a, const Point *b) {
    int d = a->x - b->x;
    return d != 0 ? d : a->y - b->y;
}

#define T PointVec, Point, c_compare_key
#define i_cmp Point_cmp
#include <stc/vec.h>

PointVec v = {0};
PointVec_push(&v, (Point){3, 1});
PointVec_push(&v, (Point){1, 2});
PointVec_sort(&v);   // uses Point_cmp
PointVec_drop(&v);
```

## Anti-Patterns

| Anti-pattern | Fix |
|-------------|-----|
| Forgetting `_drop()` | Always call destructor. Use RAII wrappers in C++ if mixing. |
| Re-using a dropped container | Re-initialize with `= {0}` before reuse. |
| Using `_pop()` and saving the pointer | Use `_pull()` to move ownership out instead. |
| Mixing two instances of the same `#define T` | Each `#include` creates one type. Use unique `ct` names. |
| `i_keydrop` without `i_keyclone` | Both must be defined together for owned elements. |
| Using `_find()` without `c_use_eq` | Add `c_use_eq` trait (or `c_use_compare`) to the declaration. |

## Review Checklist

- [ ] Is `_drop()` called for every container that was initialized?
- [ ] If elements own resources, are `i_keydrop` + `i_keyclone` both defined?
- [ ] Are `c_class_key` or `c_pro_key` used for STC containers or `cstr` as elements?
- [ ] Are inplace (CAP) variants used where heap allocation is inappropriate?
- [ ] Are `_pull()` and `_take()` used correctly to transfer ownership?
- [ ] Are `c_use_eq` / `c_use_cmp` enabled before calling `_find()` / `_sort()`?
- [ ] Is each container type instantiated with a unique name (`ct`)?

## References

- [STC GitHub](https://github.com/stclib/STC)
- [STC stack documentation](https://github.com/stclib/STC/blob/master/docs/stack_api.md)
- [STC vec documentation](https://github.com/stclib/STC/blob/master/docs/vec_api.md)
- [STC all containers](https://github.com/stclib/STC/blob/master/docs/README.md)
