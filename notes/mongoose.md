# `mongoose`

## Summary

| Data Structure  | Type                                 | File                   |
| --------------- | ------------------------------------ | ---------------------- |
| `mg_iobuf`      | Dynamic buffer (aligned allocation)  | `mongoose.h:1551-1556` |
| `mg_queue`      | Circular queue (fixed size)          | `mongoose.h:1211-1216` |
| `mg_str`        | String view (non-owning)             | `mongoose.h:1187-1190` |
| `mg_timer`      | Intrusive linked list (timers)       | `mongoose.h:1308-1320` |
| `mg_rpc`        | Intrusive linked list (RPC handlers) | `mongoose.h:3042-3047` |
| `mg_connection` | Intrusive linked list (connections)  | `mongoose.h:1713-1749` |
| `dns_data`      | Intrusive linked list (DNS requests) | `mongoose.c:130-135`   |

---

## Dynamic Buffer

Buffer Struct (mg_iobuf)
File: `repos/mongoose/mongoose.h:1551-1556`

```c
struct mg_iobuf {
  unsigned char *buf;  // Pointer to stored data
  size_t size;         // Total size available
  size_t len;          // Current number of bytes
  size_t align;        // Alignment during allocation
};
```

Roundup Helper
File: `repos/mongoose/mongoose.c:2768-2770`

```c
static size_t roundup(size_t size, size_t align) {
  return align == 0 ? size : (size + align - 1) / align * align;
}
```

Buffer Resize
File: `repos/mongoose/mongoose.c:2772-2797`

```c
int mg_iobuf_resize(struct mg_iobuf *io, size_t new_size) {
  int ok = 1;
  new_size = roundup(new_size, io->align);
  if (new_size == 0) {
    mg_free(io->buf);
    io->buf = NULL;
    io->len = io->size = 0;
  } else if (new_size != io->size) {
    // NOTE: do not use realloc here. Use calloc/free only, to ease the
    // porting to FreeRTOS and other embedded environments
    unsigned char *p = (unsigned char *) mg_calloc(1, new_size);
    if (p != NULL) {
      size_t len = new_size < io->len ? new_size : io->len;
      if (io->buf != NULL) memmove(p, io->buf, len);
      mg_free(io->buf);
      io->buf = p;
      io->size = new_size;
    } else {
      ok = 0;
      MG_ERROR(("OOM %zu", new_size));
    }
  }
  return ok;
}
```

Buffer Initialization
File: `repos/mongoose/mongoose.c:2799-2804`

```c
void mg_iobuf_init(struct mg_iobuf *io, size_t size, size_t align) {
  io->buf = NULL;
  io->align = align;
  io->size = io->len = 0;
  if (size > 0) mg_iobuf_resize(io, size);
}
```

Buffer Add
File: `repos/mongoose/mongoose.c:2806-2816`

```c
size_t mg_iobuf_add(struct mg_iobuf *io, size_t ofs, const void *buf,
                    size_t len) {
  size_t new_size = roundup(io->len + len, io->align);
  if (new_size > io->size) mg_iobuf_resize(io, new_size);
  if (new_size <= io->size) {
    if (ofs < io->len) memmove(io->buf + ofs + len, io->buf + ofs, io->len - ofs);
    if (buf != NULL) memmove(io->buf + ofs, buf, len);
    if (ofs <= io->len) io->len += len;
    return len;
  }
  return 0;
}
```

Buffer Delete
File: `repos/mongoose/mongoose.c:2818-2825`

```c
size_t mg_iobuf_del(struct mg_iobuf *io, size_t ofs, size_t len) {
  if (ofs > io->len) ofs = io->len;
  if (ofs + len > io->len) len = io->len - ofs;
  if (io->buf) memmove(io->buf + ofs, io->buf + ofs + len, io->len - ofs - len);
  if (io->len > len) io->len -= len;
  else io->len = 0;
  return len;
}
```

---

## Circular Queue

Queue Struct (mg_queue)
File: `repos/mongoose/mongoose.h:1211-1216`

```c
struct mg_queue {
  char *buf;
  size_t size;
  volatile size_t tail;
  volatile size_t head;
};
```

Queue Initialization
File: `repos/mongoose/mongoose.c:9355-9359`

```c
void mg_queue_init(struct mg_queue *q, char *buf, size_t size) {
  q->buf = buf;
  q->size = size;
  q->tail = q->head = 0;
}
```

Queue Reserve Space
File: `repos/mongoose/mongoose.c:9375-9386`

```c
char *mg_queue_book(struct mg_queue *q, size_t *len) {
  size_t space = mg_queue_space(q), head = q->head % q->size;
  if (space < *len) *len = space;
  if (*len == 0) return NULL;
  if (*len > q->size - head) *len = q->size - head;  // Wrap
  return &q->buf[head];
}
```

Queue Add
File: `repos/mongoose/mongoose.c:9402-9407`

```c
void mg_queue_add(struct mg_queue *q, size_t len) {
  mg_queue_write_len(q, &len);
  GCC_MEMORY_BARRIER;
  q->head += len + sizeof(len);
}
```

Queue Delete
File: `repos/mongoose/mongoose.c:9409-9412`

```c
void mg_queue_del(struct mg_queue *q, size_t len) {
  q->tail += len + sizeof(len);
}
```

---

## String View

String Struct (mg_str)
File: `repos/mongoose/mongoose.h:1187-1190`

```c
struct mg_str {
  char *buf;   // String data
  size_t len;  // String length
};
```

String Duplication
File: `repos/mongoose/mongoose.c:11123-11135`

```c
struct mg_str mg_strdup(const struct mg_str s) {
  struct mg_str r = {NULL, 0};
  if (s.len > 0 && s.buf != NULL) {
    char *buf = (char *) mg_calloc(1, s.len + 1);
    if (buf != NULL) {
      memcpy(buf, s.buf, s.len);
      buf[s.len] = '\0';
      r.buf = buf;
      r.len = s.len;
    }
  }
  return r;
}
```

---

## Timer Linked List

Timer Struct (mg_timer)
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

Timer Init (Insert at Head)
File: `repos/mongoose/mongoose.c:11289-11294`

```c
void mg_timer_init(struct mg_timer **head, struct mg_timer *t, uint64_t ms,
                   unsigned flags, void (*fn)(void *), void *arg) {
  t->period_ms = ms, t->expire = 0, t->flags = flags, t->fn = fn, t->arg = arg;
  t->next = *head, *head = t;
}
```

Timer Poll
File: `repos/mongoose/mongoose.c:11310-11329`

```c
void mg_timer_poll(struct mg_timer **head, uint64_t now_ms) {
  struct mg_timer *t, *tmp, **p;
  for (p = head; (t = *p) != NULL;) {
    tmp = t->next;
    if (t->expire == 0) {
      t->expire = now_ms + t->period_ms;  // Init
      if ((t->flags & MG_TIMER_RUN_NOW) && t->fn != NULL) t->fn(t->arg);
    } else if (t->expire <= now_ms) {
      if (t->fn != NULL) t->fn(t->arg);
      t->expire = now_ms + t->period_ms;  // Repeat
      if (t->flags & MG_TIMER_ONCE) {
        if (t->flags & MG_TIMER_AUTODELETE) mg_timer_free(head, t);
        t->fn = NULL;
      }
    }
    p = (t->next == tmp) ? &t->next : p;
  }
}
```

---

## RPC Handler Linked List

RPC Struct (mg_rpc)
File: `repos/mongoose/mongoose.h:3042-3047`

```c
struct mg_rpc {
  struct mg_rpc *next;              // Next in list
  struct mg_str method;             // Method pattern
  void (*fn)(struct mg_rpc_req *);  // Handler function
  void *fn_data;                    // Handler function argument
};
```

RPC Add Handler
File: `repos/mongoose/mongoose.c:9421-9430`

```c
void mg_rpc_add(struct mg_rpc **head, struct mg_str method,
                void (*fn)(struct mg_rpc_req *), void *fn_data) {
  struct mg_rpc *h = (struct mg_rpc *) mg_calloc(1, sizeof(*h));
  if (h != NULL) {
    h->method = mg_strdup(method);
    h->fn = fn;
    h->fn_data = fn_data;
    h->next = *head, *head = h;
  }
}
```

RPC Delete Handler
File: `repos/mongoose/mongoose.c:9432-9443`

```c
void mg_rpc_del(struct mg_rpc **head, void (*fn)(struct mg_rpc_req *)) {
  struct mg_rpc *h, **p;
  for (p = head; (h = *p) != NULL;) {
    if (h->fn == fn) {
      *p = h->next;
      mg_free((void *) h->method.buf);
      mg_free(h);
    } else {
      p = &h->next;
    }
  }
}
```

---

## Connection Linked List

Connection Struct (mg_connection, partial)
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

---

## List Manipulation Macros

List Macros
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

---

## DNS Request Linked List

DNS Data Struct (dns_data)
File: `repos/mongoose/mongoose.c:130-135`

```c
struct dns_data {
  struct dns_data *next;
  struct mg_connection *c;
  uint64_t expire;
  uint16_t txnid;
};
```
