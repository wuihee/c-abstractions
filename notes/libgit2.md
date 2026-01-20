# `libgit2`

## Summary

| Data Structure    | Type                                       | File                              |
| ----------------- | ------------------------------------------ | --------------------------------- |
| `git_array_t`     | Dynamic array (macro-based, 1.5x growth)   | `src/util/array.h:26-28`          |
| `git_vector`      | Dynamic array (void pointers, 1.5x growth) | `src/util/vector.h:19-25`         |
| `git_str`         | Dynamic string (1.5x growth)               | `src/util/str.h:12-16`            |
| `git_hashmap`     | Hash table (macro-based, 0.77 load factor) | `src/util/hashmap.h:65-77`        |
| `git_commit_list` | Singly-linked list                         | `src/libgit2/commit_list.h:26-46` |
| `git_pqueue`      | Priority queue (heap on git_vector)        | `src/util/pqueue.h:14`            |
| `git_pool`        | Chunked memory pool                        | `src/util/pool.h:33-37`           |
| `git_bitvec`      | Bit vector (hybrid inline/heap)            | `src/util/bitvec.h:19-25`         |
| `git_sortedcache` | Sorted cache (vector + hashmap)            | `src/util/sortedcache.h:31-42`    |

---

## Dynamic Array (Macro-Based)

Array Macro (git_array_t)
File: `repos/libgit2/src/util/array.h:26-28`

```c
#define git_array_t(type) struct { type *ptr; size_t size, asize; }
#define GIT_ARRAY_INIT { NULL, 0, 0 }
```

Array Allocation (1.5x Growth)
File: `repos/libgit2/src/util/array.h:44-73`

```c
GIT_INLINE(void *) git_array__alloc(void *_a, size_t esize)
{
	git_array_generic_t *a = _a;
	size_t new_size;
	char *new_array;

	if (a->size < 8)
		new_size = 8;
	else {
		new_size = a->size;
		new_size += new_size / 2;

		if (new_size < a->size)
			return NULL;
	}

	new_array = git__reallocarray(a->ptr, new_size, esize);
	if (!new_array)
		return NULL;

	a->ptr = new_array;
	a->asize = new_size;
	a->size++;

	return a->ptr + (a->size - 1) * esize;
}
```

---

## Dynamic Array (Vector)

Vector Struct (git_vector)
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

Vector Growth Calculation
File: `repos/libgit2/src/util/vector.c:15-29`

```c
GIT_INLINE(size_t) compute_new_size(git_vector *v)
{
	size_t new_size = v->_alloc_size;

	if (v->_alloc_size == 0) {
		new_size = MIN_ALLOCSIZE;
	} else {
		/* if the new_size overflows then we're out of memory anyway */
		new_size += new_size / 2;
	}

	return new_size;
}
```

Vector Resize
File: `repos/libgit2/src/util/vector.c:31-45`

```c
GIT_INLINE(int) resize_vector(git_vector *v, size_t new_size)
{
	void *new_contents;

	new_contents = git__reallocarray(v->contents, new_size, sizeof(void *));
	GIT_ERROR_CHECK_ALLOC(new_contents);

	v->_alloc_size = new_size;
	v->contents = new_contents;

	return 0;
}
```

Vector Insert
File: `repos/libgit2/src/util/vector.c:135-148`

```c
int git_vector_insert(git_vector *v, void *element)
{
	GIT_ASSERT_ARG(v);

	if (v->length >= v->_alloc_size &&
	    resize_vector(v, compute_new_size(v)) < 0)
		return -1;

	v->contents[v->length++] = element;

	git_vector_set_sorted(v, v->length <= 1);

	return 0;
}
```

---

## Dynamic String

String Buffer Struct (git_str)
File: `repos/libgit2/src/util/str.h:12-16`

```c
struct git_str {
	char *ptr;
	size_t asize;
	size_t size;
};
```

String Growth (1.5x, Rounded to 8)
File: `repos/libgit2/src/util/str.c:36-106`

```c
int git_str_try_grow(
	git_str *buf, size_t target_size, bool mark_oom)
{
	char *new_ptr;
	size_t new_size;

	if (buf->ptr == git_str__oom)
		return -1;

	if (buf->asize == 0 && buf->size != 0) {
		git_error_set(GIT_ERROR_INVALID, "cannot grow a borrowed buffer");
		return GIT_EINVALID;
	}

	if (!target_size)
		target_size = buf->size;

	if (target_size <= buf->asize)
		return 0;

	if (buf->asize == 0) {
		new_size = target_size;
		new_ptr = NULL;
	} else {
		new_size = buf->asize;
		new_ptr = buf->ptr;
	}

	/* grow the buffer size by 1.5, until it's big enough
	 * to fit our target size */
	while (new_size < target_size)
		new_size = (new_size << 1) - (new_size >> 1);

	/* round allocation up to multiple of 8 */
	new_size = (new_size + 7) & ~7;

	if (new_size < buf->size) {
		if (mark_oom)
			buf->ptr = git_str__oom;

		git_error_set_oom();
		return -1;
	}

	new_ptr = git__realloc(new_ptr, new_size);

	if (!new_ptr) {
		if (mark_oom) {
			if (buf->ptr && (buf->ptr != git_str__initstr))
				git__free(buf->ptr);
			buf->ptr = git_str__oom;
		}
		return -1;
	}

	buf->asize = new_size;
	buf->ptr   = new_ptr;

	return 0;
}
```

---

## Hash Table

Hash Table Struct Members (macro)
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

Hash Table Resize
File: `repos/libgit2/src/util/hashmap.h:152-249`

```c
GIT_INLINE(int) name##__resize(struct name *h, uint32_t new_n_buckets)    \
{                                                                         \
    /* Load factor threshold: 0.77 */                                     \
    uint32_t *new_flags = 0;                                              \
    uint32_t j = 1;                                                       \
    {                                                                     \
        kroundup32(new_n_buckets);                                        \
        if (new_n_buckets < 4) new_n_buckets = 4;                         \
        if (h->size >= (uint32_t)(new_n_buckets * __ac_HASH_UPPER + 0.5)) \
            j = 0;                                                        \
        else {                                                            \
            new_flags = git__reallocarray(NULL, __ac_fsize(new_n_buckets),\
                sizeof(uint32_t));                                        \
            if (!new_flags) return -1;                                    \
            memset(new_flags, 0xaa, __ac_fsize(new_n_buckets) * sizeof(uint32_t)); \
            if (h->n_buckets < new_n_buckets) {                           \
                key_t *new_keys = git__reallocarray(h->keys,              \
                    new_n_buckets, sizeof(key_t));                        \
                if (!new_keys) { git__free(new_flags); return -1; }       \
                h->keys = new_keys;                                       \
                if (!is_set) {                                            \
                    val_t *new_vals = git__reallocarray(h->vals,          \
                        new_n_buckets, sizeof(val_t));                    \
                    if (!new_vals) { git__free(new_flags); return -1; }   \
                    h->vals = new_vals;                                   \
                }                                                         \
            }                                                             \
        }                                                                 \
    }                                                                     \
    /* ... rehashing logic follows ... */                                 \
}
```

---

## Linked List

Commit List Node Struct (git_commit_list_node)
File: `repos/libgit2/src/libgit2/commit_list.h:26-42`

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
```

Commit List Struct (git_commit_list)
File: `repos/libgit2/src/libgit2/commit_list.h:44-47`

```c
typedef struct git_commit_list {
	git_commit_list_node *item;
	struct git_commit_list *next;
} git_commit_list;
```

---

## Priority Queue

Priority Queue Type
File: `repos/libgit2/src/util/pqueue.h:14`

```c
typedef git_vector git_pqueue;
```

Priority Queue Insert (Heap Sift-Up)
File: `repos/libgit2/src/util/pqueue.c:82-102`

```c
int git_pqueue_insert(git_pqueue *pq, void *item)
{
	size_t el = pq->length;

	if (git_vector_insert(pq, item) < 0)
		return -1;

	pqueue_up(pq, el);
	return 0;
}
```

---

## Memory Pool

Pool Struct (git_pool)
File: `repos/libgit2/src/util/pool.h:33-37`

```c
typedef struct {
	git_pool_page *pages;
	size_t item_size;
	size_t page_size;
} git_pool;
```

Pool Page Struct (git_pool_page)
File: `repos/libgit2/src/util/pool.c:15-20`

```c
struct git_pool_page {
	git_pool_page *next;
	size_t size;
	size_t avail;
	GIT_ALIGN(char data[GIT_FLEX_ARRAY], 8);
};
```

Pool Page Allocation
File: `repos/libgit2/src/util/pool.c:61-78`

```c
static void *pool_alloc_page(git_pool *pool, size_t size)
{
	git_pool_page *page;
	const size_t new_page_size = compute_page_size(pool, size);
	size_t alloc_size;

	if (GIT_ADD_SIZET_OVERFLOW(&alloc_size, new_page_size, sizeof(git_pool_page)))
		return NULL;

	page = git__malloc(alloc_size);
	if (!page)
		return NULL;

	page->size = new_page_size;
	page->avail = new_page_size - size;
	page->next = pool->pages;

	pool->pages = page;

	return page->data;
}
```

---

## Bit Vector

Bit Vector Struct (git_bitvec)
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

Bit Vector Init (Hybrid Inline/Heap)
File: `repos/libgit2/src/util/bitvec.h:27-39`

```c
GIT_INLINE(int) git_bitvec_init(git_bitvec *bv, size_t capacity)
{
	memset(bv, 0x0, sizeof(*bv));

	if (capacity >= 64) {
		bv->length = (capacity / 64) + 1;
		bv->u.words = git__calloc(bv->length, sizeof(uint64_t));
		if (!bv->u.words)
			return -1;
	}

	return 0;
}
```

---

## Sorted Cache

Sorted Cache Struct (git_sortedcache)
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
