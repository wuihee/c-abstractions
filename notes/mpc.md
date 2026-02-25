# `mpc`

## Summary

| Data Structure   | Type                                                         | File            |
| ---------------- | ------------------------------------------------------------ | --------------- |
| `mpc_input_t`    | Input reader + embedded pool allocator + dynamic marks stack | `mpc.c:78-101`  |
| `mpc_err_t`      | Dynamic array of expected strings (no capacity)              | `mpc.h:40-47`   |
| `mpc_parser_t`   | Tagged union (sum type)                                      | `mpc.c:964-969` |
| `mpc_ast_t`      | N-ary tree with dynamic children array                       | `mpc.h:285-291` |
| `mpc_ast_trav_t` | Parent-pointer linked list (traversal stack)                 | `mpc.h:316-321` |

---

## `mpc_input_t` - Input Reader with Pool Allocator

File: `repos/mpc/mpc.c:78-101`

```c
typedef struct {
  int type;           /* STRING, FILE, or PIPE */
  char *filename;
  mpc_state_t state;

  char *string;
  char *buffer;
  FILE *file;

  int suppress;
  int backtrack;
  int marks_slots;
  int marks_num;
  mpc_state_t *marks;  /* dynamic array of backtrack positions */

  char *lasts;         /* parallel array: last char at each mark */
  char last;

  size_t mem_index;
  char mem_full[MPC_INPUT_MEM_NUM];   /* 512-slot occupancy bitmap */
  mpc_mem_t mem[MPC_INPUT_MEM_NUM];   /* 512 fixed 64-byte slots */
} mpc_input_t;
```

Combines three responsibilities: input source abstraction (string/file/pipe), a dynamic backtracking marks stack, and an embedded slab allocator for small parse-time allocations.

```rust
struct Input {
    source: InputSource,       // String | File | Pipe
    state: State,
    marks: Vec<State>,         // backtrack stack
    pool: SlabAllocator,       // 512 x 64-byte slots
}
```

### Marks Stack - `mpc_input_mark()`

File: `repos/mpc/mpc.c:301-320`

```c
static void mpc_input_mark(mpc_input_t *i) {
  i->marks_num++;
  if (i->marks_num > i->marks_slots) {
    i->marks_slots = i->marks_num + i->marks_num / 2;
    i->marks = realloc(i->marks, sizeof(mpc_state_t) * i->marks_slots);
    i->lasts = realloc(i->lasts, sizeof(char) * i->marks_slots);
  }
  i->marks[i->marks_num-1] = i->state;
  i->lasts[i->marks_num-1] = i->last;
}
```

Growth strategy: 1.5x (`marks_num + marks_num / 2`). Starts at `MPC_INPUT_MARKS_MIN` (32). Uses `realloc` directly. `marks` and `lasts` are parallel arrays (backtrack state + last char).

### Pool Allocator - `mpc_malloc()` / `mpc_free()`

File: `repos/mpc/mpc.c:237-268`

```c
typedef struct { char mem[64]; } mpc_mem_t;

static void *mpc_malloc(mpc_input_t *i, size_t n) {
  if (n > sizeof(mpc_mem_t)) { return malloc(n); }
  /* scan for a free slot in the circular pool */
  j = i->mem_index;
  do {
    if (!i->mem_full[i->mem_index]) {
      p = (void*)(i->mem + i->mem_index);
      i->mem_full[i->mem_index] = 1;
      i->mem_index = (i->mem_index+1) % MPC_INPUT_MEM_NUM;
      return p;
    }
    i->mem_index = (i->mem_index+1) % MPC_INPUT_MEM_NUM;
  } while (j != i->mem_index);
  return malloc(n); /* fallback if pool is exhausted */
}
```

A fixed-size slab allocator for small allocations (≤ 64 bytes). 512 slots tracked by a bitmap (`mem_full`). Scans circularly for a free slot. Falls back to `malloc` for large allocations or when the pool is full. This avoids heap overhead for the many small strings created during parsing.

---

## `mpc_err_t` - Error with Dynamic Expected Array

File: `repos/mpc/mpc.h:40-47`

```c
typedef struct {
  mpc_state_t state;
  int expected_num;
  char *filename;
  char *failure;
  char **expected;   /* dynamic array of expected-token strings */
  char received;
} mpc_err_t;
```

Errors carry a list of what tokens were expected at the failure point. The `expected` array grows one element at a time with no separate capacity field — every insert calls `realloc`.

```rust
struct ParseError {
    state: State,
    filename: String,
    failure: Option<String>,
    expected: Vec<String>,
    received: char,
}
```

### Insert - `mpc_err_add_expected()`

File: `repos/mpc/mpc.c:740-746`

```c
static void mpc_err_add_expected(mpc_input_t *i, mpc_err_t *x, char *expected) {
  x->expected_num++;
  x->expected = mpc_realloc(i, x->expected, sizeof(char*) * x->expected_num);
  x->expected[x->expected_num-1] = mpc_malloc(i, strlen(expected) + 1);
  strcpy(x->expected[x->expected_num-1], expected);
}
```

No capacity tracking. Grows by exactly 1 on every insert (`realloc` to `expected_num * sizeof(char*)`). Acceptable because error objects are short-lived and the number of expected tokens is typically small.

---

## `mpc_parser_t` - Tagged Union

File: `repos/mpc/mpc.c:964-969`

```c
struct mpc_parser_t {
  char *name;
  mpc_pdata_t data;   /* union of all parser variant payloads */
  char type;          /* discriminant: MPC_TYPE_OR, MPC_TYPE_AND, etc. */
  char retained;
};
```

A sum type: `type` selects which field of the `mpc_pdata_t` union is active. The union covers 18 variants (leaf parsers, combinators, repeaters, etc.).

```rust
enum Parser {
    Fail(String),
    Single(char),
    Range(char, char),
    Or(Vec<Parser>),
    And { parsers: Vec<Parser>, dtors: Vec<Dtor>, fold: FoldFn },
    Many { parser: Box<Parser>, fold: FoldFn },
    // ...
}
```

The `or.xs` and `and.xs` sub-arrays are fixed-size: allocated once at construction with `malloc(n * sizeof(mpc_parser_t*))` since `n` is known from the varargs call site.

---

## `mpc_ast_t` - N-ary Tree with Dynamic Children

File: `repos/mpc/mpc.h:285-291`

```c
typedef struct mpc_ast_t {
  char *tag;
  char *contents;
  mpc_state_t state;
  int children_num;
  struct mpc_ast_t **children;   /* dynamic array of child pointers */
} mpc_ast_t;
```

An n-ary tree. Each node owns a flat array of child pointers. No capacity field — grows by 1 on every `mpc_ast_add_child` call.

```rust
struct Ast {
    tag: String,
    contents: String,
    state: State,
    children: Vec<Box<Ast>>,
}
```

### Insert - `mpc_ast_add_child()`

File: `repos/mpc/mpc.c:3025-3030`

```c
mpc_ast_t *mpc_ast_add_child(mpc_ast_t *r, mpc_ast_t *a) {
  r->children_num++;
  r->children = realloc(r->children, sizeof(mpc_ast_t*) * r->children_num);
  r->children[r->children_num-1] = a;
  return r;
}
```

Grows by exactly 1 per insert. No capacity tracking. Each `realloc` is a potential allocation; acceptable because AST construction is a one-time pass and typical children counts are small.

---

## `mpc_ast_trav_t` - Parent-Pointer Stack (Traversal)

File: `repos/mpc/mpc.h:316-321`

```c
typedef struct mpc_ast_trav_t {
  mpc_ast_t             *curr_node;
  struct mpc_ast_trav_t *parent;
  int                    curr_child;
  mpc_ast_trav_order_t   order;
} mpc_ast_trav_t;
```

An iterator state implemented as an intrusive linked list. Each frame records the current node and which child to visit next. Frames are individually heap-allocated and freed as traversal moves up the tree.

```rust
struct Traversal<'a> {
    stack: Vec<(&'a Ast, usize)>,  // (node, next_child_index)
    order: Order,
}
```

Frames are created on `traverse_next` as the traversal descends and freed immediately when it backtracks — effectively a hand-rolled call stack.
