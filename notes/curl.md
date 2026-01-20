# `curl`

## Summary

### Layer 1: Primitive Data Structures

| Data Structure | Type                               | File                       |
| -------------- | ---------------------------------- | -------------------------- |
| `dynbuf`       | Dynamic string (doubling strategy) | `lib/curlx/dynbuf.h:27-35` |
| `Curl_llist`   | Doubly-linked list                 | `lib/llist.h:33-41`        |
| `Curl_hash`    | Hash table (string keys, chained)  | `lib/hash.h:53-67`         |
| `bufq`         | Buffer queue (chunked, pooled)     | `lib/bufq.h:92-101`        |

### Layer 2: Specialized Data Structures

| Data Structure  | Type                          | File                      |
| --------------- | ----------------------------- | ------------------------- |
| `uint_hash`     | Hash table (uint32 keys)      | `lib/uint-hash.h:33-41`   |
| `uint32_tbl`    | Dynamic table (sparse array)  | `lib/uint-table.h:31-40`  |
| `uint32_bset`   | Dense bitset                  | `lib/uint-bset.h:40-47`   |
| `uint32_spbset` | Sparse bitset (linked chunks) | `lib/uint-spbset.h:48-53` |

---

# Layer 1: Primitive Data Structures

## `dynbuf` - Dynamic String

File: `repos/curl/lib/curlx/dynbuf.h:27-35`

```c
struct dynbuf {
  char *bufr;    /* point to a null-terminated allocated buffer */
  size_t leng;   /* number of bytes *EXCLUDING* the null-terminator */
  size_t allc;   /* size of the current allocation */
  size_t toobig; /* size limit for the buffer */
#ifdef DEBUGBUILD
  int init;      /* detect API usage mistakes */
#endif
};
```

A growable byte buffer for string construction. Uses a doubling growth strategy with a configurable maximum size (`toobig`). Always maintains a null terminator. The debug build includes an `init` sentinel to detect uninitialized usage.

```rust
struct Dynbuf {
    data: Vec<u8>,
    max_size: usize,
}
```

---

## `Curl_llist` - Doubly-Linked List

File: `repos/curl/lib/llist.h:33-51`

```c
struct Curl_llist {
  struct Curl_llist_node *_head;
  struct Curl_llist_node *_tail;
  Curl_llist_dtor _dtor;
  size_t _size;
#ifdef DEBUGBUILD
  int _init;      /* detect API usage mistakes */
#endif
};

struct Curl_llist_node {
  struct Curl_llist *_list; /* the list where this belongs */
  void *_ptr;
  struct Curl_llist_node *_prev;
  struct Curl_llist_node *_next;
#ifdef DEBUGBUILD
  int _init;      /* detect API usage mistakes */
#endif
};
```

A doubly-linked list with head/tail pointers for O(1) insertion at both ends. Each node stores a back-pointer to its containing list. Supports a custom destructor function for element cleanup. The underscore prefix convention indicates private fields.

```rust
struct CurlLlist<T> {
    nodes: VecDeque<T>,
    dtor: Option<fn(&mut T)>,
}
```

---

## `Curl_hash` - Hash Table

File: `repos/curl/lib/hash.h:45-67`

```c
struct Curl_hash_element {
  struct Curl_hash_element *next;
  void *ptr;
  Curl_hash_elem_dtor dtor;
  size_t key_len;
  char key[1]; /* allocated memory following the struct */
};

struct Curl_hash {
  struct Curl_hash_element **table;
  hash_function hash_func;
  comp_function comp_func;
  Curl_hash_dtor dtor;
  size_t slots;
  size_t size;
#ifdef DEBUGBUILD
  int init;
#endif
};
```

A chained hash table with configurable hash and comparison functions. Elements use a flexible array member (`key[1]`) to store variable-length keys inline. Supports per-element destructors in addition to a table-wide destructor. The `slots` field is the bucket count; `size` is the element count.

```rust
struct CurlHash<K, V> {
    table: HashMap<K, V>,
    dtor: Option<fn(&mut V)>,
}
```

---

## `bufq` - Buffer Queue

File: `repos/curl/lib/bufq.h:33-101`

```c
struct buf_chunk {
  struct buf_chunk *next;  /* to keep it in a list */
  size_t dlen;             /* the amount of allocated x.data[] */
  size_t r_offset;         /* first unread bytes */
  size_t w_offset;         /* one after last written byte */
  union {
    uint8_t data[1];       /* the buffer for `dlen` bytes */
    void *dummy;           /* alignment */
  } x;
};

struct bufc_pool {
  struct buf_chunk *spare;  /* list of available spare chunks */
  size_t chunk_size;        /* the size of chunks in this pool */
  size_t spare_count;       /* current number of spare chunks in list */
  size_t spare_max;         /* max number of spares to keep */
};

struct bufq {
  struct buf_chunk *head;       /* chunk with bytes to read from */
  struct buf_chunk *tail;       /* chunk to write to */
  struct buf_chunk *spare;      /* list of free chunks, unless `pool` */
  struct bufc_pool *pool;       /* optional pool for free chunks */
  size_t chunk_count;           /* current number of chunks in `head+spare` */
  size_t max_chunks;            /* max `head` chunks to use */
  size_t chunk_size;            /* size of chunks to manage */
  int opts;                     /* options for handling queue, see below */
};
```

A chunked buffer queue optimized for streaming data. Maintains separate read/write offsets per chunk. Supports chunk pooling to reduce allocation overhead. Used extensively for HTTP/2 and HTTP/3 multiplexed streams where data arrives out of order.

```rust
struct Bufq {
    chunks: VecDeque<Vec<u8>>,
    chunk_size: usize,
    max_chunks: usize,
    read_offset: usize,
}
```

---

# Layer 2: Specialized Data Structures

## `uint_hash` - uint32 Hash Table

File: `repos/curl/lib/uint-hash.h:33-41`

```c
struct uint_hash {
  struct uint_hash_entry **table;
  Curl_uint32_hash_dtor *dtor;
  uint32_t slots;
  uint32_t size;
#ifdef DEBUGBUILD
  int init;
#endif
};
```

A specialized hash table for uint32 keys. Simpler than `Curl_hash` since keys are fixed-size integers. Used for mapping numeric identifiers (stream IDs, connection IDs) to associated data structures.

```rust
struct UintHash<V> {
    table: HashMap<u32, V>,
    dtor: Option<fn(&mut V)>,
}
```

---

## `uint32_tbl` - Sparse Array

File: `repos/curl/lib/uint-table.h:31-40`

```c
struct uint32_tbl {
  void **rows;  /* array of void* holding entries */
  Curl_uint32_tbl_entry_dtor *entry_dtor;
  uint32_t nrows;  /* length of `rows` array */
  uint32_t nentries; /* entries in table */
  uint32_t last_key_added; /* UINT_MAX or last key added */
#ifdef DEBUGBUILD
  int init;
#endif
};
```

A sparse array indexed by uint32 keys. Grows via `realloc` when keys exceed current capacity. Tracks `last_key_added` for optimized sequential insertion. More memory-efficient than a hash table when keys are dense and mostly sequential.

```rust
struct Uint32Tbl<V> {
    entries: Vec<Option<V>>,
    count: u32,
}
```

---

## `uint32_bset` - Dense Bitset

File: `repos/curl/lib/uint-bset.h:40-47`

```c
struct uint32_bset {
  uint64_t *slots;
  uint32_t nslots;
  uint32_t first_slot_used;
#ifdef DEBUGBUILD
  int init;
#endif
};
```

A dense bitset backed by an array of 64-bit words. Tracks `first_slot_used` to optimize iteration. Grows via `realloc` when setting bits beyond current capacity. Used for tracking sets of stream IDs or connection states.

```rust
struct Uint32Bset {
    bits: Vec<u64>,
}
```

---

## `uint32_spbset` - Sparse Bitset

File: `repos/curl/lib/uint-spbset.h:42-53`

```c
struct uint32_spbset_chunk {
  struct uint32_spbset_chunk *next;
  uint64_t slots[CURL_UINT32_SPBSET_CH_SLOTS];  /* 4 slots = 256 bits */
  uint32_t offset;
};

struct uint32_spbset {
  struct uint32_spbset_chunk head;
#ifdef DEBUGBUILD
  int init;
#endif
};
```

A sparse bitset using linked 256-bit chunks. Each chunk stores its offset, allowing non-contiguous bit ranges without allocating intervening memory. More memory-efficient than `uint32_bset` when bits are clustered in widely separated ranges.

```rust
struct Uint32Spbset {
    chunks: BTreeMap<u32, [u64; 4]>,
}
```
