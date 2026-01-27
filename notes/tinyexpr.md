# `tinyexpr`

## Summary

**No dynamic arrays found.** TinyExpr is a minimal (~700 LOC) expression parser and evaluator that uses a tree-based representation. All allocations are fixed-size with no `realloc`-based growth.

### Data Structures Present (Non-Qualifying)

| Data Structure | Type                    | File                 | Why Excluded                       |
| -------------- | ----------------------- | -------------------- | ---------------------------------- |
| `te_expr`      | Expression tree node    | `tinyexpr.h:36-40`   | Flexible array member, no growth   |
| `te_variable`  | Variable binding        | `tinyexpr.h:55-60`   | Simple struct, no dynamic behavior |
| `state`        | Parser state            | `tinyexpr.c:66-75`   | No capacity tracking, no growth    |
| `functions`    | Built-in function table | `tinyexpr.c:162-193` | Static const array                 |

---

## `te_expr` - Expression Tree Node

File: `repos/tinyexpr/tinyexpr.h:36-40`

```c
typedef struct te_expr {
    int type;
    union {double value; const double *bound; const void *function;};
    void *parameters[1];
} te_expr;
```

An expression tree node using a flexible array member (`parameters[1]`) to store child pointers. The number of parameters depends on the function arity (0-7). Each node is allocated once at the exact required size via `new_expr()`:

File: `repos/tinyexpr/tinyexpr.c:87-101`

```c
static te_expr *new_expr(const int type, const te_expr *parameters[]) {
    const int arity = ARITY(type);
    const int psize = sizeof(void*) * arity;
    const int size = (sizeof(te_expr) - sizeof(void*)) + psize + (IS_CLOSURE(type) ? sizeof(void*) : 0);
    te_expr *ret = malloc(size);
    // ...
}
```

**Why excluded:** No length/capacity fields. Size is computed at allocation time based on arity and never changes. No `realloc` growth.

---

## `te_variable` - Variable Binding

File: `repos/tinyexpr/tinyexpr.h:55-60`

```c
typedef struct te_variable {
    const char *name;
    const void *address;
    int type;
    void *context;
} te_variable;
```

A simple struct for binding variable names to memory addresses. Users pass an array of these to `te_compile()` for variable lookup.

**Why excluded:** Plain struct with no dynamic array behavior.

---

## `state` - Parser State

File: `repos/tinyexpr/tinyexpr.c:66-75`

```c
typedef struct state {
    const char *start;
    const char *next;
    int type;
    union {double value; const double *bound; const void *function;};
    void *context;

    const te_variable *lookup;
    int lookup_len;
} state;
```

Parser state for the recursive descent parser. The `lookup` pointer references user-provided variables (not owned/grown by the parser).

**Why excluded:** No capacity field, no `realloc` growth. The `lookup` array is provided externally and not modified.

---

## Architecture Notes

TinyExpr parses expressions into a tree of `te_expr` nodes, where each node represents either:

- A constant value
- A variable reference
- A function call with 0-7 parameters

The tree is built via recursive descent parsing, with each node individually `malloc`'d at the exact size needed. Evaluation walks the tree recursively. This design avoids dynamic arrays entirely.
