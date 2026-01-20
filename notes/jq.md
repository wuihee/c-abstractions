# `jq`

## Summary

| Data Structure    | Type                                  | File                     |
| ----------------- | ------------------------------------- | ------------------------ |
| `jvp_array`       | Dynamic array (flexible array member) | `src/jv.c:822-826`       |
| `jvp_string`      | Dynamic string (doubling strategy)    | `src/jv.c:1090-1098`     |
| `jvp_object`      | Hash table (chained, flexible array)  | `src/jv.c:1572-1576`     |
| `object_slot`     | Hash table slot (collision chain)     | `src/jv.c:1565-1570`     |
| `struct stack`    | Execution stack (doubling strategy)   | `src/exec_stack.h:50-54` |
| `struct inst`     | Doubly-linked list (IR nodes)         | `src/compile.c:23-65`    |
| `block`           | Linked list container (first/last)    | `src/compile.h:12-15`    |
| `jv_parser`       | Parser state (dynamic arrays)         | `src/jv_parse.c:33-66`   |
| `struct bytecode` | Bytecode container (dynamic arrays)   | `src/bytecode.h:72-88`   |
| `struct locfile`  | Source file tracker (dynamic linemap) | `src/locfile.h:12-21`    |
| `jv_refcnt`       | Reference counter                     | `src/jv.c:53-55`         |

---

## Dynamic Array

Core Array Struct (jvp_array)
File: `repos/jq/src/jv.c:822-826`

```c
typedef struct {
  jv_refcnt refcnt;
  int length, alloc_length;
  jv elements[];
} jvp_array;
```

Array Reallocation
File: `repos/jq/src/jv.c:878-909`

```c
static jv* jvp_array_write(jv* a, int i) {
  assert(i >= 0);
  jvp_array* array = jvp_array_ptr(*a);

  int pos = i + jvp_array_offset(*a);
  if (pos < array->alloc_length && jvp_refcnt_unshared(a->u.ptr)) {
    // use existing array space
    for (int j = array->length; j <= pos; j++) {
      array->elements[j] = JV_NULL;
    }
    array->length = imax(pos + 1, array->length);
    a->size = imax(i + 1, a->size);
    return &array->elements[pos];
  } else {
    // allocate a new array
    int new_length = imax(i + 1, jvp_array_length(*a));
    jvp_array* new_array = jvp_array_alloc(ARRAY_SIZE_ROUND_UP(new_length));
    int j;
    for (j = 0; j < jvp_array_length(*a); j++) {
      new_array->elements[j] =
        jv_copy(array->elements[j + jvp_array_offset(*a)]);
    }
    for (; j < new_length; j++) {
      new_array->elements[j] = JV_NULL;
    }
    new_array->length = new_length;
    jvp_array_free(*a);
    jv new_jv = {JVP_FLAGS_ARRAY, 0, 0, new_length, {&new_array->refcnt}};
    *a = new_jv;
    return &new_array->elements[i];
  }
}
```

## Dynamic String

Dynamic String Struct (jvp_string)
File: `repos/jq/src/jv.c:1090-1098`

```c
typedef struct {
  jv_refcnt refcnt;
  uint32_t hash;
  uint32_t length_hashed;
  uint32_t alloc_length;
  char data[];
} jvp_string;
```

String Reallocation (Doubling Strategy)
File: `repos/jq/src/jv.c:1174-1198`

```c
static jv jvp_string_append(jv string, const char* data, uint32_t len) {
  jvp_string* s = jvp_string_ptr(string);
  uint32_t currlen = jvp_string_length(s);

  if (jvp_refcnt_unshared(string.u.ptr) &&
      jvp_string_remaining_space(s) >= len) {
    // the next string fits at the end of a
    memcpy(s->data + currlen, data, len);
    s->data[currlen + len] = 0;
    s->length_hashed = (currlen + len) << 1;
    return string;
  } else {
    // allocate a bigger buffer and copy
    uint32_t allocsz = (currlen + len) * 2;
    if (allocsz < 32) allocsz = 32;
    jvp_string* news = jvp_string_alloc(allocsz);
    news->length_hashed = (currlen + len) << 1;
    memcpy(news->data, s->data, currlen);
    memcpy(news->data + currlen, data, len);
    news->data[currlen + len] = 0;
    jvp_string_free(string);
    jv r = {JVP_FLAGS_STRING, 0, 0, 0, {&news->refcnt}};
    return r;
  }
}
```

## Hash Table

Object Slot Struct (object_slot)
File: `repos/jq/src/jv.c:1565-1570`

```c
struct object_slot {
  int next; /* next slot with same hash, for collisions */
  uint32_t hash;
  jv string;
  jv value;
};
```

Hash Table Struct (jvp_object)
File: `repos/jq/src/jv.c:1572-1576`

```c
typedef struct {
  jv_refcnt refcnt;
  int next_free;
  struct object_slot elements[];
} jvp_object;
```

## Stack

Stack Struct (struct stack)
File: `repos/jq/src/exec_stack.h:50-54`

```c
struct stack {
  char* mem_end;
  stack_ptr bound;
  stack_ptr limit;
};
```

Stack Reallocation (Doubling Strategy)
File: `repos/jq/src/exec_stack.h:83-93`

```c
static void stack_reallocate(struct stack* s, size_t sz) {
  int old_mem_length = -(s->bound) + ALIGNMENT;
  char* old_mem_start = (s->mem_end != NULL) ? (s->mem_end - old_mem_length) : NULL;

  int new_mem_length = align_round_up((old_mem_length + sz + 256) * 2);
  char* new_mem_start = jv_mem_realloc(old_mem_start, new_mem_length);
  memmove(new_mem_start + (new_mem_length - old_mem_length),
            new_mem_start, old_mem_length);
  s->mem_end = new_mem_start + new_mem_length;
  s->bound = -(new_mem_length - ALIGNMENT);
}
```

## Doubly-Linked List (IR)

Instruction Node Struct (struct inst)
File: `repos/jq/src/compile.c:23-65`

```c
struct inst {
  struct inst* next;
  struct inst* prev;

  opcode op;

  struct {
    uint16_t intval;
    struct inst* target;
    jv constant;
    const struct cfunction* cfunc;
  } imm;

  struct locfile* locfile;
  location source;

  // Binding
  struct inst* bound_by;
  char* symbol;
  int any_unbound;
  int referenced;

  int nformals;
  int nactuals;

  block subfn;   // used by CLOSURE_CREATE (body of function)
  block arglist; // used by CLOSURE_CREATE (formals) and CALL_JQ (arguments)

  // This instruction is compiled as part of which function?
  struct bytecode* compiled;

  int bytecode_pos; // position just after this insn
};
```

Block Container Struct (block)
File: `repos/jq/src/compile.h:12-15`

```c
typedef struct block {
  inst* first;
  inst* last;
} block;
```

Node Allocation
File: `repos/jq/src/compile.c:67-83`

```c
static inst* inst_new(opcode op) {
  inst* i = jv_mem_alloc(sizeof(inst));
  i->next = i->prev = 0;
  i->op = op;
  i->bytecode_pos = -1;
  i->bound_by = 0;
  i->symbol = 0;
  i->any_unbound = 0;
  i->referenced = 0;
  i->nformals = -1;
  i->nactuals = -1;
  i->subfn = gen_noop();
  i->arglist = gen_noop();
  i->source = UNKNOWN_LOCATION;
  i->locfile = 0;
  return i;
}
```

## Parser State

Parser Struct (jv_parser)
File: `repos/jq/src/jv_parse.c:33-66`

```c
struct jv_parser {
  const char* curr_buf;
  int curr_buf_length;
  int curr_buf_pos;
  int curr_buf_is_partial;
  int eof;
  unsigned bom_strip_position;

  int flags;

  jv* stack;                   // dynamic array
  int stackpos;
  int stacklen;
  jv path;
  enum last_seen last_seen;
  jv output;
  jv next;

  char* tokenbuf;              // dynamic buffer
  int tokenpos;
  int tokenlen;

  int line, column;

  struct dtoa_context dtoa;

  enum {
    JV_PARSER_NORMAL,
    JV_PARSER_STRING,
    JV_PARSER_STRING_ESCAPE,
    JV_PARSER_WAITING_FOR_RS
  } st;
  unsigned int last_ch_was_ws:1;
};
```

Parser Stack Push (Doubling + 10 Strategy)
File: `repos/jq/src/jv_parse.c:146-154`

```c
static void push(struct jv_parser* p, jv v) {
  assert(p->stackpos <= p->stacklen);
  if (p->stackpos == p->stacklen) {
    p->stacklen = p->stacklen * 2 + 10;
    p->stack = jv_mem_realloc(p->stack, p->stacklen * sizeof(jv));
  }
  assert(p->stackpos < p->stacklen);
  p->stack[p->stackpos++] = v;
}
```

## Bytecode Container

Bytecode Struct (struct bytecode)
File: `repos/jq/src/bytecode.h:72-88`

```c
struct bytecode {
  uint16_t* code;
  int codelen;

  int nlocals;
  int nclosures;

  jv constants; // JSON array of constants
  struct symbol_table* globals;

  struct bytecode** subfunctions;
  int nsubfunctions;

  struct bytecode* parent;

  jv debuginfo;
};
```

## Source Location Tracker

Location File Struct (struct locfile)
File: `repos/jq/src/locfile.h:12-21`

```c
struct locfile {
  jv fname;
  const char* data;
  int length;
  int* linemap;   // dynamic array for line positions
  int nlines;
  char *error;
  jq_state *jq;
  int refct;
};
```

## Reference Counter

Reference Count Struct (jv_refcnt)
File: `repos/jq/src/jv.c:53-55`

```c
typedef struct jv_refcnt {
  int count;
} jv_refcnt;
```

Reference Count Operations
File: `repos/jq/src/jv.c:59-71`

```c
static void jvp_refcnt_inc(jv_refcnt* c) {
  c->count++;
}

static int jvp_refcnt_dec(jv_refcnt* c) {
  c->count--;
  return c->count == 0;
}

static int jvp_refcnt_unshared(jv_refcnt* c) {
  assert(c->count > 0);
  return c->count == 1;
}
```
