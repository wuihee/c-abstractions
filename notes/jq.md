# `jq`

## Summary

### Layer 1: Primitive Data Structures

| Data Structure | Type                                  | File                     |
| -------------- | ------------------------------------- | ------------------------ |
| `jvp_array`    | Dynamic array (flexible array member) | `src/jv.c:822-826`       |
| `jvp_string`   | Dynamic string (doubling strategy)    | `src/jv.c:1090-1098`     |
| `jvp_object`   | Hash table (chained, flexible array)  | `src/jv.c:1572-1576`     |
| `struct stack` | Execution stack (doubling strategy)   | `src/exec_stack.h:50-54` |

### Layer 2: Composite Data Structures

| Data Structure     | Type                          | File                   |
| ------------------ | ----------------------------- | ---------------------- |
| `struct jq_state`  | Interpreter state             | `src/execute.c:22-54`  |
| `struct jv_parser` | JSON parser (internal arrays) | `src/jv_parse.c:33-66` |
| `struct inst`      | Doubly-linked list (IR nodes) | `src/compile.c:23-65`  |
| `struct bytecode`  | Compiled bytecode container   | `src/bytecode.h:72-88` |
| `struct locfile`   | Source location tracker       | `src/locfile.h:12-21`  |

---

# Layer 1: Primitive Data Structures

## `jvp_array` - Dynamic Array

File: `repos/jq/src/jv.c:822-826`

```c
typedef struct {
  jv_refcnt refcnt;
  int length, alloc_length;
  jv elements[];
} jvp_array;
```

A reference-counted dynamic array using a flexible array member. Stores JSON values (`jv`) and grows via reallocation when capacity is exceeded. The `refcnt` enables copy-on-write semantics - arrays are only copied when modified while shared.

```rust
struct JvpArray {
    refcnt: Rc<RefCell<Vec<Jv>>>,
}
```

---

## `jvp_string` - Dynamic String

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

A reference-counted dynamic string with cached hash value. The `length_hashed` field packs both the string length and a "hash computed" flag into a single value (length in upper bits). Uses doubling growth strategy on append with a minimum allocation of 32 bytes.

```rust
struct JvpString {
    refcnt: Rc<RefCell<StringInner>>,
}

struct StringInner {
    hash: Option<u32>,
    data: String,
}
```

---

## `jvp_object` - Hash Table

File: `repos/jq/src/jv.c:1565-1576`

```c
struct object_slot {
  int next;
  uint32_t hash;
  jv string;
  jv value;
};

typedef struct {
  jv_refcnt refcnt;
  int next_free;
  struct object_slot elements[];
} jvp_object;
```

A reference-counted hash table using chained collision resolution. Each slot stores a key-value pair plus a `next` index for collision chains. The `next_free` field tracks the next available slot in the flexible array. Keys are always strings; values are any `jv` type.

```rust
struct JvpObject {
    refcnt: Rc<RefCell<HashMap<String, Jv>>>,
}
```

---

## `struct stack` - Execution Stack

File: `repos/jq/src/exec_stack.h:50-54`

```c
struct stack {
  char* mem_end;
  stack_ptr bound;
  stack_ptr limit;
};
```

A growable byte-addressable stack that grows downward in memory. Used by the jq virtual machine to store stack frames and local variables. The stack is addressed via negative offsets from `mem_end`. Reallocates with doubling strategy when space is exhausted.

```rust
struct Stack {
    memory: Vec<u8>,
    top: usize,
}
```

---

# Layer 2: Composite Data Structures

## `struct jq_state` - Interpreter State

File: `repos/jq/src/execute.c:22-54`

```c
struct jq_state {
  jq_msg_cb err_cb;
  void *err_cb_data;
  jv error;

  struct stack stk;
  stack_ptr curr_frame;
  stack_ptr stk_top;
  stack_ptr fork_top;

  jv path;
  int subexp_nest;
  int debug_trace_enabled;
  int initial_execution;
  unsigned next_label;

  int halted;
  jv exit_code;
  jv error_message;

  struct bytecode* bc;

  jq_input_cb input_cb;
  void *input_cb_data;
  jq_msg_cb debug_cb;
  void *debug_cb_data;
  jq_msg_cb stderr_cb;
  void *stderr_cb_data;

  struct jv_parser *parser;
  jv attrs;
};
```

The main interpreter state container. Holds the execution stack, current bytecode, error handling callbacks, and I/O callbacks. Manages the virtual machine's runtime state including fork points for backtracking and debug/trace settings.

```rust
struct JqState {
    error: Option<Jv>,
    stack: Stack,
    curr_frame: usize,
    path: Jv,
    halted: bool,
    exit_code: Jv,
    bytecode: Rc<Bytecode>,
    parser: Option<JvParser>,
    attrs: Jv,
    // callbacks omitted - would use trait objects or closures
}
```

---

## `struct jv_parser` - JSON Parser

File: `repos/jq/src/jv_parse.c:33-66`

```c
struct jv_parser {
  const char* curr_buf;
  int curr_buf_length;
  int curr_buf_pos;
  int eof;

  int flags;

  jv* stack;
  int stackpos;
  int stacklen;
  jv path;
  jv output;
  jv next;

  char* tokenbuf;
  int tokenpos;
  int tokenlen;

  int line, column;

  enum {
    JV_PARSER_NORMAL,
    JV_PARSER_STRING,
    JV_PARSER_STRING_ESCAPE,
    JV_PARSER_WAITING_FOR_RS
  } st;
};
```

A streaming JSON parser with internal dynamic arrays for nested value construction. The `stack` array tracks nested objects/arrays being parsed. The `tokenbuf` accumulates characters for string/number tokens. Supports incremental parsing of partial input.

```rust
struct JvParser {
    curr_buf: Vec<u8>,
    pos: usize,
    eof: bool,
    flags: ParserFlags,
    stack: Vec<Jv>,
    path: Jv,
    output: Option<Jv>,
    tokenbuf: String,
    line: usize,
    column: usize,
    state: ParserState,
}
```

---

## `struct inst` / `block` - IR Linked List

File: `repos/jq/src/compile.c:23-65`, `repos/jq/src/compile.h:12-15`

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

  struct inst* bound_by;
  char* symbol;

  int nformals;
  int nactuals;

  block subfn;
  block arglist;

  struct bytecode* compiled;
  int bytecode_pos;
};

typedef struct block {
  inst* first;
  inst* last;
} block;
```

A doubly-linked list of intermediate representation instructions. Each `inst` node represents one IR operation with optional immediate values (constants, jump targets, C function pointers). The `block` struct provides O(1) access to both ends for efficient concatenation during compilation.

```rust
struct Inst {
    op: Opcode,
    imm: Immediate,
    source: Location,
    bound_by: Option<Weak<Inst>>,
    symbol: Option<String>,
    nformals: i32,
    nactuals: i32,
    subfn: Block,
    arglist: Block,
}

struct Block {
    instructions: VecDeque<Rc<Inst>>,
}
```

---

## `struct bytecode` - Compiled Bytecode

File: `repos/jq/src/bytecode.h:72-88`

```c
struct bytecode {
  uint16_t* code;
  int codelen;

  int nlocals;
  int nclosures;

  jv constants;
  struct symbol_table* globals;

  struct bytecode** subfunctions;
  int nsubfunctions;

  struct bytecode* parent;

  jv debuginfo;
};
```

Container for compiled jq bytecode. Holds the instruction stream (`code`), a constant pool (`constants` as a jv array), and nested subfunctions for closures. The `parent` pointer links to the enclosing function's bytecode for lexical scoping.

```rust
struct Bytecode {
    code: Vec<u16>,
    nlocals: i32,
    nclosures: i32,
    constants: Jv,
    globals: Rc<SymbolTable>,
    subfunctions: Vec<Rc<Bytecode>>,
    parent: Option<Weak<Bytecode>>,
    debuginfo: Jv,
}
```

---

## `struct locfile` - Source Location Tracker

File: `repos/jq/src/locfile.h:12-21`

```c
struct locfile {
  jv fname;
  const char* data;
  int length;
  int* linemap;
  int nlines;
  char *error;
  jq_state *jq;
  int refct;
};
```

Tracks source file content and line positions for error reporting. The `linemap` dynamic array stores byte offsets where each line begins, enabling O(1) line number lookup from byte position. Reference-counted to allow sharing across compilation units.

```rust
struct Locfile {
    fname: String,
    data: String,
    linemap: Vec<usize>,
    error: Option<String>,
}
```
