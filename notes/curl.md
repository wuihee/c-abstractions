# `curl`

## Summary

| Data Structure | Type | File |
|---|---|---|
| `Curl_llist` | Doubly-linked list | `lib/llist.h:33-41` |
| `Curl_llist_node` | Linked list node | `lib/llist.h:43-51` |
| `dynbuf` | Dynamic string (doubling strategy) | `lib/curlx/dynbuf.h:27-35` |
| `Curl_hash` | Hash table (string keys, chained) | `lib/hash.h:53-67` |
| `Curl_hash_element` | Hash table element | `lib/hash.h:45-51` |
| `bufq` | Buffer queue (chunked, pooled) | `lib/bufq.h:92-101` |
| `buf_chunk` | Buffer chunk | `lib/bufq.h:33-42` |
| `bufc_pool` | Chunk pool | `lib/bufq.h:51-56` |
| `uint_hash` | Hash table (uint32 keys) | `lib/uint-hash.h:33-41` |
| `uint32_tbl` | Dynamic table (sparse array) | `lib/uint-table.h:31-40` |
| `uint32_bset` | Dense bitset | `lib/uint-bset.h:40-47` |
| `uint32_spbset` | Sparse bitset (linked chunks) | `lib/uint-spbset.h:48-53` |

---

## Doubly Linked List

List Struct (Curl_llist)
File: `repos/curl/lib/llist.h:33-41`

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
```

Node Struct (Curl_llist_node)
File: `repos/curl/lib/llist.h:43-51`

```c
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

Insert Operation
File: `repos/curl/lib/llist.c:67-106`

```c
void Curl_llist_insert_next(struct Curl_llist *list,
                            struct Curl_llist_node *e, /* may be NULL */
                            const void *p,
                            struct Curl_llist_node *ne)
{
  DEBUGASSERT(list);
  DEBUGASSERT(list->_init == LLISTINIT);
  DEBUGASSERT(ne);

#ifdef DEBUGBUILD
  ne->_init = NODEINIT;
#endif
  ne->_ptr = CURL_UNCONST(p);
  ne->_list = list;
  if(list->_size == 0) {
    list->_head = ne;
    list->_head->_prev = NULL;
    list->_head->_next = NULL;
    list->_tail = ne;
  }
  else {
    /* if 'e' is NULL here, we insert the new element first in the list */
    ne->_next = e ? e->_next : list->_head;
    ne->_prev = e;
    if(!e) {
      list->_head->_prev = ne;
      list->_head = ne;
    }
    else if(e->_next) {
      e->_next->_prev = ne;
    }
    else {
      list->_tail = ne;
    }
    if(e)
      e->_next = ne;
  }

  ++list->_size;
}
```

## Dynamic String

Dynamic Buffer Struct (dynbuf)
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

Buffer Reallocation (Doubling Strategy)
File: `repos/curl/lib/curlx/dynbuf.c:67-121`

```c
static CURLcode dyn_nappend(struct dynbuf *s,
                            const unsigned char *mem, size_t len)
{
  size_t idx = s->leng;
  size_t a = s->allc;
  size_t fit = len + idx + 1; /* new string + old string + zero byte */

  /* try to detect if there is rubbish in the struct */
  DEBUGASSERT(s->init == DYNINIT);
  DEBUGASSERT(s->toobig);
  DEBUGASSERT(idx < s->toobig);
  DEBUGASSERT(!s->leng || s->bufr);
  DEBUGASSERT(a <= s->toobig);
  DEBUGASSERT(!len || mem);

  if(fit > s->toobig) {
    curlx_dyn_free(s);
    return CURLE_TOO_LARGE;
  }
  else if(!a) {
    DEBUGASSERT(!idx);
    /* first invoke */
    if(MIN_FIRST_ALLOC > s->toobig)
      a = s->toobig;
    else if(fit < MIN_FIRST_ALLOC)
      a = MIN_FIRST_ALLOC;
    else
      a = fit;
  }
  else {
    while(a < fit)
      a *= 2;
    if(a > s->toobig)
      /* no point in allocating a larger buffer than this is allowed to use */
      a = s->toobig;
  }

  if(a != s->allc) {
    /* this logic is not using Curl_saferealloc() to make the tool not have to
       include that as well when it uses this code */
    void *p = curlx_realloc(s->bufr, a);
    if(!p) {
      curlx_dyn_free(s);
      return CURLE_OUT_OF_MEMORY;
    }
    s->bufr = p;
    s->allc = a;
  }

  if(len)
    memcpy(&s->bufr[idx], mem, len);
  s->leng = idx + len;
  s->bufr[s->leng] = 0;
  return CURLE_OK;
}
```

## Hash Table

Hash Table Struct (Curl_hash)
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

  /* Hash function to be used for this hash table */
  hash_function hash_func;
  /* Comparator function to compare keys */
  comp_function comp_func;
  /* General element construct, unless element itself carries one */
  Curl_hash_dtor dtor;
  size_t slots;
  size_t size;
#ifdef DEBUGBUILD
  int init;
#endif
};
```

## Buffer Queue

Chunk Struct (buf_chunk)
File: `repos/curl/lib/bufq.h:33-42`

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
```

Chunk Pool Struct (bufc_pool)
File: `repos/curl/lib/bufq.h:51-56`

```c
struct bufc_pool {
  struct buf_chunk *spare;  /* list of available spare chunks */
  size_t chunk_size;        /* the size of chunks in this pool */
  size_t spare_count;       /* current number of spare chunks in list */
  size_t spare_max;         /* max number of spares to keep */
};
```

Buffer Queue Struct (bufq)
File: `repos/curl/lib/bufq.h:92-101`

```c
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

## uint32 Hash Table

Hash Table Struct (uint_hash)
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

## Dynamic Table (Sparse Array)

Table Struct (uint32_tbl)
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

Table Resize
File: `repos/curl/lib/uint-table.c:63-83`

```c
CURLcode Curl_uint32_tbl_resize(struct uint32_tbl *tbl, uint32_t nrows)
{
  /* we use `tbl->nrows + 1` during iteration, want that to work */
  DEBUGASSERT(tbl->init == CURL_UINT32_TBL_MAGIC);
  if(!nrows)
    return CURLE_BAD_FUNCTION_ARGUMENT;
  if(nrows != tbl->nrows) {
    void **rows = curlx_calloc(nrows, sizeof(void *));
    if(!rows)
      return CURLE_OUT_OF_MEMORY;
    if(tbl->rows) {
      memcpy(rows, tbl->rows, (CURLMIN(nrows, tbl->nrows) * sizeof(void *)));
      if(nrows < tbl->nrows)
        uint32_tbl_clear_rows(tbl, nrows, tbl->nrows);
      curlx_free(tbl->rows);
    }
    tbl->rows = rows;
    tbl->nrows = nrows;
  }
  return CURLE_OK;
}
```

## Dense Bitset

Bitset Struct (uint32_bset)
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

Bitset Resize
File: `repos/curl/lib/uint-bset.c:40-61`

```c
CURLcode Curl_uint32_bset_resize(struct uint32_bset *bset, uint32_t nmax)
{
  uint32_t nslots = (nmax < (UINT32_MAX - 63)) ?
                    ((nmax + 63) / 64) : (UINT32_MAX / 64);

  DEBUGASSERT(bset->init == CURL_UINT32_BSET_MAGIC);
  if(nslots != bset->nslots) {
    uint64_t *slots = curlx_calloc(nslots, sizeof(uint64_t));
    if(!slots)
      return CURLE_OUT_OF_MEMORY;

    if(bset->slots) {
      memcpy(slots, bset->slots,
             (CURLMIN(nslots, bset->nslots) * sizeof(uint64_t)));
      curlx_free(bset->slots);
    }
    bset->slots = slots;
    bset->nslots = nslots;
    bset->first_slot_used = 0;
  }
  return CURLE_OK;
}
```

## Sparse Bitset

Chunk Struct (uint32_spbset_chunk)
File: `repos/curl/lib/uint-spbset.h:42-46`

```c
struct uint32_spbset_chunk {
  struct uint32_spbset_chunk *next;
  uint64_t slots[CURL_UINT32_SPBSET_CH_SLOTS];  /* 4 slots = 256 bits */
  uint32_t offset;
};
```

Sparse Bitset Struct (uint32_spbset)
File: `repos/curl/lib/uint-spbset.h:48-53`

```c
struct uint32_spbset {
  struct uint32_spbset_chunk head;
#ifdef DEBUGBUILD
  int init;
#endif
};
```
