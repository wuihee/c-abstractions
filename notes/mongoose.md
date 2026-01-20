# `mongoose`

## Summary

### Layer 1: Primitive Data Structures

| Data Structure | Type                                | File                   |
| -------------- | ----------------------------------- | ---------------------- |
| `mg_iobuf`     | Dynamic buffer (aligned allocation) | `mongoose.h:1551-1556` |
| `mg_queue`     | Circular queue (fixed size)         | `mongoose.h:1211-1216` |
| `mg_str`       | String view (non-owning)            | `mongoose.h:1187-1190` |

### Layer 2: Application Data Structures

| Data Structure  | Type                                 | File                   |
| --------------- | ------------------------------------ | ---------------------- |
| `mg_timer`      | Intrusive linked list (timers)       | `mongoose.h:1308-1320` |
| `mg_rpc`        | Intrusive linked list (RPC handlers) | `mongoose.h:3042-3047` |
| `mg_connection` | Intrusive linked list (connections)  | `mongoose.h:1713-1749` |
| `dns_data`      | Intrusive linked list (DNS requests) | `mongoose.c:130-135`   |

---

# Layer 1: Primitive Data Structures

## `mg_iobuf` - Dynamic Buffer

File: `repos/mongoose/mongoose.h:1551-1556`

```c
struct mg_iobuf {
  unsigned char *buf;  // Pointer to stored data
  size_t size;         // Total size available
  size_t len;          // Current number of bytes
  size_t align;        // Alignment during allocation
};
```

A growable byte buffer with configurable alignment. Uses `calloc`/`free` instead of `realloc` for embedded system compatibility (FreeRTOS). The `align` field rounds allocations to the specified boundary. Supports insertion at arbitrary offsets with automatic data shifting.

```rust
struct MgIobuf {
    data: Vec<u8>,
    align: usize,
}
```

---

## `mg_queue` - Circular Queue

File: `repos/mongoose/mongoose.h:1211-1216`

```c
struct mg_queue {
  char *buf;
  size_t size;
  volatile size_t tail;
  volatile size_t head;
};
```

A fixed-size circular buffer for producer-consumer communication. Uses `volatile` indices for lock-free operation between ISR and main context on embedded systems. The buffer wraps at `size` boundary. Stores length-prefixed messages to support variable-sized items.

```rust
struct MgQueue {
    buf: Box<[u8]>,
    tail: AtomicUsize,
    head: AtomicUsize,
}
```

---

## `mg_str` - String View

File: `repos/mongoose/mongoose.h:1187-1190`

```c
struct mg_str {
  char *buf;   // String data
  size_t len;  // String length
};
```

A non-owning string view (pointer + length). Does not require null termination, enabling zero-copy references into larger buffers. Used throughout mongoose for parsing HTTP headers and JSON without allocation. The `mg_strdup` function creates an owned copy when needed.

```rust
struct MgStr<'a> {
    data: &'a str,
}
```

---

# Layer 2: Application Data Structures

## `mg_timer` - Timer Linked List

File: `repos/mongoose/mongoose.h:1308-1320`

```c
struct mg_timer {
  uint64_t period_ms;          // Timer period in milliseconds
  uint64_t expire;             // Expiration timestamp in milliseconds
  unsigned flags;              // Timer flags (ONCE, REPEAT, RUN_NOW, etc.)
  void (*fn)(void *);          // Function to call
  void *arg;                   // Function argument
  struct mg_timer *next;       // Linkage
};
```

An intrusive singly-linked list of timers. Each timer stores its callback, period, and next-fire timestamp. Supports one-shot (`MG_TIMER_ONCE`) and repeating modes. The `MG_TIMER_AUTODELETE` flag enables automatic removal after firing. Timers insert at the head for O(1) addition.

```rust
struct MgTimer {
    period_ms: u64,
    expire: u64,
    flags: TimerFlags,
    callback: Box<dyn Fn()>,
}

struct MgTimerList {
    timers: Vec<MgTimer>,
}
```

---

## `mg_rpc` - RPC Handler Linked List

File: `repos/mongoose/mongoose.h:3042-3047`

```c
struct mg_rpc {
  struct mg_rpc *next;              // Next in list
  struct mg_str method;             // Method pattern
  void (*fn)(struct mg_rpc_req *);  // Handler function
  void *fn_data;                    // Handler function argument
};
```

An intrusive singly-linked list of JSON-RPC method handlers. Each entry maps a method name pattern to a handler function. Method strings are duplicated on registration. Handlers insert at the head and are searched linearly on dispatch.

```rust
struct MgRpc {
    method: String,
    handler: Box<dyn Fn(&mut MgRpcReq)>,
}

struct MgRpcList {
    handlers: Vec<MgRpc>,
}
```

---

## `mg_connection` - Connection Linked List

File: `repos/mongoose/mongoose.h:1713-1749`

```c
struct mg_connection {
  struct mg_connection *next;     // Linkage in struct mg_mgr :: connections
  struct mg_mgr *mgr;             // Our container
  struct mg_addr loc;             // Local address
  struct mg_addr rem;             // Remote address
  void *fd;                       // Opaque file descriptor
  unsigned long id;               // Connection ID
  struct mg_iobuf recv;           // Incoming data (dynamic buffer)
  struct mg_iobuf send;           // Outgoing data (dynamic buffer)
  struct mg_iobuf prof;           // Profile data (dynamic buffer)
  struct mg_iobuf rtls;           // TLS encrypted data (dynamic buffer)
  // ... additional fields
};
```

An intrusive singly-linked list of active connections managed by `mg_mgr`. Each connection owns multiple `mg_iobuf` instances for I/O buffering. The `next` pointer enables O(1) insertion and linear traversal. Connections are identified by a unique `id` for event routing.

```rust
struct MgConnection {
    id: u64,
    loc: MgAddr,
    rem: MgAddr,
    recv: MgIobuf,
    send: MgIobuf,
}

struct MgMgr {
    connections: Vec<MgConnection>,
}
```

---

## `dns_data` - DNS Request Linked List

File: `repos/mongoose/mongoose.c:130-135`

```c
struct dns_data {
  struct dns_data *next;
  struct mg_connection *c;
  uint64_t expire;
  uint16_t txnid;
};
```

An intrusive singly-linked list tracking pending DNS requests. Links a transaction ID (`txnid`) to the connection waiting for the response. The `expire` timestamp enables timeout handling. Entries are removed when a response arrives or the timeout fires.

```rust
struct DnsData {
    connection: Weak<MgConnection>,
    expire: u64,
    txnid: u16,
}

struct DnsPending {
    requests: Vec<DnsData>,
}
```

---

# Utility: List Manipulation Macros

File: `repos/mongoose/mongoose.h:1519-1537`

```c
#define LIST_ADD_HEAD(type_, head_, elem_) \
  do {                                     \
    (elem_)->next = *(head_);              \
    *(head_) = (elem_);                    \
  } while (0)

#define LIST_ADD_TAIL(type_, head_, elem_)                     \
  do {                                                         \
    type_ **h = head_;                                         \
    while (*h != NULL) h = &(*h)->next;                        \
    (elem_)->next = NULL;                                      \
    *h = (elem_);                                              \
  } while (0)

#define LIST_DELETE(type_, head_, elem_)                       \
  do {                                                         \
    type_ **h = head_;                                         \
    while (*h != NULL && *h != (elem_)) h = &(*h)->next;       \
    if (*h != NULL) *h = (elem_)->next;                        \
  } while (0)
```

Generic macros for intrusive linked list operations. All Layer 2 data structures use these macros for consistency. `LIST_ADD_HEAD` is O(1); `LIST_ADD_TAIL` and `LIST_DELETE` are O(n). The pointer-to-pointer idiom (`type_ **h`) elegantly handles the head case.
