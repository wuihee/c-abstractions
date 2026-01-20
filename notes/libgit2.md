# `libgit2`

## Summary

### Layer 1: Primitive Data Structures

| Data Structure | Type                                       | File                       |
| -------------- | ------------------------------------------ | -------------------------- |
| `git_array_t`  | Dynamic array (macro-based, 1.5x growth)   | `src/util/array.h:26-28`   |
| `git_vector`   | Dynamic array (void pointers, 1.5x growth) | `src/util/vector.h:19-25`  |
| `git_str`      | Dynamic string (1.5x growth)               | `src/util/str.h:12-16`     |
| `git_hashmap`  | Hash table (macro-based, 0.77 load factor) | `src/util/hashmap.h:65-77` |
| `git_bitvec`   | Bit vector (hybrid inline/heap)            | `src/util/bitvec.h:19-25`  |

### Layer 2: Composite Data Structures

| Data Structure    | Type                          | File                              |
| ----------------- | ----------------------------- | --------------------------------- |
| `git_commit_list` | Singly-linked list            | `src/libgit2/commit_list.h:26-46` |
| `git_pqueue`      | Priority queue (heap)         | `src/util/pqueue.h:14`            |
| `git_pool`        | Chunked memory pool           | `src/util/pool.h:33-37`           |
| `git_sortedcache` | Sorted cache (vector+hashmap) | `src/util/sortedcache.h:31-42`    |

---

# Layer 1: Primitive Data Structures

## `git_array_t` - Dynamic Array (Macro)

File: `repos/libgit2/src/util/array.h:26-28`

```c
#define git_array_t(type) struct { type *ptr; size_t size, asize; }
#define GIT_ARRAY_INIT { NULL, 0, 0 }
```

A type-safe dynamic array implemented via C macros. Provides compile-time type checking by generating struct definitions with the element type. Uses 1.5x growth strategy with a minimum size of 8. The `size` field tracks element count; `asize` tracks allocated capacity.

```rust
struct GitArray<T> {
    elements: Vec<T>,
}
```

---

## `git_vector` - Dynamic Array (Void Pointers)

File: `repos/libgit2/src/util/vector.h:19-25`

```c
typedef struct git_vector {
	size_t _alloc_size;
	git_vector_cmp _cmp;
	void **contents;
	size_t length;
	uint32_t flags;
} git_vector;
```

A dynamic array of void pointers with optional sorting support. Stores a comparator function (`_cmp`) for binary search and sorted insertion. The `flags` field tracks whether the vector is sorted. Uses 1.5x growth strategy. More flexible than `git_array_t` but without type safety.

```rust
struct GitVector<T> {
    contents: Vec<T>,
    cmp: Option<fn(&T, &T) -> Ordering>,
    is_sorted: bool,
}
```

---

## `git_str` - Dynamic String

File: `repos/libgit2/src/util/str.h:12-16`

```c
struct git_str {
	char *ptr;
	size_t asize;
	size_t size;
};
```

A growable string buffer with 1.5x growth strategy, rounded to 8-byte alignment. Uses a sentinel pointer (`git_str__oom`) to mark out-of-memory state rather than returning errors on every operation. Supports "borrowed" mode where `asize=0` but `size>0`, indicating the buffer points to external memory.

```rust
struct GitStr {
    data: String,
}
```

---

## `git_hashmap` - Hash Table (Macro)

File: `repos/libgit2/src/util/hashmap.h:65-77`

```c
#define GIT_HASHMAP_STRUCT_MEMBERS(key_t, val_t) \
	uint32_t n_buckets, \
	         size, \
	         n_occupied, \
	         upper_bound; \
	uint32_t *flags; \
	key_t *keys; \
	val_t *vals;

#define GIT_HASHMAP_STRUCT(name, key_t, val_t) \
	struct name { \
		GIT_HASHMAP_STRUCT_MEMBERS(key_t, val_t) \
	};
```

A type-safe hash table using macros to generate specialized implementations. Uses open addressing with quadratic probing. Maintains a 0.77 load factor (`__ac_HASH_UPPER`). The `flags` array uses 2 bits per bucket for empty/deleted/occupied states. Derived from khash.

```rust
struct GitHashmap<K, V> {
    table: HashMap<K, V>,
}
```

---

## `git_bitvec` - Bit Vector

File: `repos/libgit2/src/util/bitvec.h:19-25`

```c
typedef struct {
	size_t length;
	union {
		uint64_t *words;
		uint64_t bits;
	} u;
} git_bitvec;
```

A hybrid bit vector that stores up to 64 bits inline (in `u.bits`) and switches to heap allocation (in `u.words`) for larger sizes. The `length` field is 0 for inline storage, otherwise it's the word count. This optimization avoids heap allocation for small bitsets.

```rust
enum GitBitvec {
    Inline(u64),
    Heap(Vec<u64>),
}
```

---

# Layer 2: Composite Data Structures

## `git_commit_list` - Singly-Linked List

File: `repos/libgit2/src/libgit2/commit_list.h:26-47`

```c
typedef struct git_commit_list_node {
	git_oid oid;
	int64_t time;
	uint32_t generation;
	unsigned int seen:1,
	             uninteresting:1,
	             topo_delay:1,
	             parsed:1,
	             added:1,
	             flags : FLAG_BITS;
	uint16_t in_degree;
	uint16_t out_degree;
	struct git_commit_list_node **parents;
} git_commit_list_node;

typedef struct git_commit_list {
	git_commit_list_node *item;
	struct git_commit_list *next;
} git_commit_list;
```

A singly-linked list specialized for commit traversal. Each node contains commit metadata (OID, timestamp, generation number) plus bit flags for graph algorithms. The `parents` pointer array enables DAG traversal. Used extensively in revision walking, merge-base finding, and topological sorting.

```rust
struct GitCommitListNode {
    oid: GitOid,
    time: i64,
    generation: u32,
    flags: CommitFlags,
    in_degree: u16,
    out_degree: u16,
    parents: Vec<Rc<GitCommitListNode>>,
}

struct GitCommitList {
    nodes: Vec<Rc<GitCommitListNode>>,
}
```

---

## `git_pqueue` - Priority Queue

File: `repos/libgit2/src/util/pqueue.h:14`

```c
typedef git_vector git_pqueue;
```

A binary heap implemented on top of `git_vector`. Uses the vector's comparator function for heap ordering. Provides O(log n) insert and extract-min operations. Used for commit date ordering in revision walks and merge-base computation.

```rust
struct GitPqueue<T> {
    heap: BinaryHeap<T>,
}
```

---

## `git_pool` - Memory Pool

File: `repos/libgit2/src/util/pool.h:33-37`, `repos/libgit2/src/util/pool.c:15-20`

```c
typedef struct {
	git_pool_page *pages;
	size_t item_size;
	size_t page_size;
} git_pool;

struct git_pool_page {
	git_pool_page *next;
	size_t size;
	size_t avail;
	GIT_ALIGN(char data[GIT_FLEX_ARRAY], 8);
};
```

A chunked memory pool for bulk allocation of same-sized objects. Allocates large pages and carves out fixed-size items. Pages form a linked list for deallocation. Used for allocating many small objects (like tree entries) with a single bulk free at the end.

```rust
struct GitPool<T> {
    arena: typed_arena::Arena<T>,
}
```

---

## `git_sortedcache` - Sorted Cache

File: `repos/libgit2/src/util/sortedcache.h:31-42`

```c
typedef struct {
	git_refcount rc;
	git_rwlock   lock;
	size_t       item_path_offset;
	git_sortedcache_free_item_fn free_item;
	void         *free_item_payload;
	git_pool     pool;
	git_vector   items;
	git_hashmap_str map;
	git_futils_filestamp stamp;
	char         path[GIT_FLEX_ARRAY];
} git_sortedcache;
```

A thread-safe cache combining a sorted vector and a hash map. Items are accessible by index (sorted order) or by string key (hash lookup). Uses `git_pool` for item storage, `git_vector` for sorted iteration, and `git_hashmap_str` for O(1) key lookup. Tracks file modification time via `stamp` for cache invalidation.

```rust
struct GitSortedcache<T> {
    items: Vec<Rc<T>>,
    map: HashMap<String, Rc<T>>,
    lock: RwLock<()>,
    path: PathBuf,
    stamp: SystemTime,
}
```
